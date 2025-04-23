# Deployment Guide

This document outlines the steps to deploy the LinkedIn Recruiter Outreach application to various environments.

## Prerequisites

Before deployment, ensure you have:

1. Supabase account and project
2. Azure OpenAI API access
3. Proxycurl API key (optional but recommended)
4. Resend API key
5. GitHub account for CI/CD
6. Vercel account (for frontend)
7. Azure account (for backend API + worker)

## 1. Supabase Setup

The first step is setting up the database and authentication:

```bash
# Initialize Supabase locally
supabase init

# Apply the database schema
supabase db reset
```

For production, create a new Supabase project via the web dashboard and run:

```bash
# Link to your production project
supabase link --project-ref your-project-ref

# Push migrations to production
supabase db push
```

Important Supabase settings:

- Enable email authentication
- Configure row-level security policies
- Create storage bucket for profiles
- Save the project URL and anon key for environment variables

## 2. Azure App Service (Backend + Worker)

The backend API and Playwright worker are deployed together to Azure App Service:

```bash
# Create Azure resource group
az group create --name outreach-rg --location eastus

# Deploy to Azure App Service
az webapp up -n recruiter-api -s B1 -g outreach-rg --runtime "PYTHON|3.12"

# Set environment variables
az webapp config appsettings set -g outreach-rg -n recruiter-api --settings \
  AZURE_OPENAI_ENDPOINT=your-endpoint \
  AZURE_OPENAI_KEY=your-key \
  PROXYCURL_API_KEY=your-key \
  RESEND_API_KEY=your-key \
  SUPABASE_URL=your-url \
  SUPABASE_ANON_KEY=your-key \
  RATE_LIMIT_PROFILE_FETCH=15
```

### Azure App Service Configuration

1. Enable system-assigned managed identity:

```bash
az webapp identity assign -g outreach-rg -n recruiter-api
```

2. Configure a deployment slot for staging:

```bash
az webapp deployment slot create -g outreach-rg -n recruiter-api --slot staging
```

3. Enable application insights for monitoring:

```bash
az monitor app-insights component create -g outreach-rg --app recruiter-api-insights -l eastus
az webapp config appsettings set -g outreach-rg -n recruiter-api --slot staging --settings APPINSIGHTS_INSTRUMENTATIONKEY=your-key
```

### Worker Process Configuration

For the Playwright worker process, additional configuration is needed:

1. Install browser dependencies:

```bash
# Add to deployment script
apt-get update
apt-get install -y ca-certificates fonts-liberation libasound2 libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgbm1 libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 lsb-release wget xdg-utils
```

2. Configure virtual memory for browser:

```bash
# Add to deployment script
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p
```

## 3. Vercel Deployment (Frontend)

The Next.js frontend can be deployed using Vercel's continuous deployment:

1. Connect your GitHub repository to Vercel
    
2. Configure the following environment variables:
    
    - `NEXT_PUBLIC_SUPABASE_URL`
    - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
    - `NEXT_PUBLIC_API_URL`
3. Deploy the frontend:
    

```bash
vercel --prod
```

### Custom Domain (Optional)

To set up a custom domain in Vercel:

1. Go to your project settings
2. Navigate to "Domains"
3. Add your domain and follow the verification steps

## 4. CI/CD Pipeline (GitHub Actions)

Set up continuous integration and deployment using GitHub Actions.

Create a file at `.github/workflows/main.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'
          
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install uv
          uv pip install -r requirements.txt
          uv pip install pytest pytest-cov
          
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install Node.js dependencies
        run: npm install
        
      - name: Run Python tests
        run: pytest
        
      - name: Run TypeScript tests
        run: npm test
  
  deploy-backend:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Azure CLI
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
          
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'recruiter-api'
          slot-name: 'staging'
          package: .
          
      - name: Swap staging to production
        run: az webapp deployment slot swap -g outreach-rg -n recruiter-api --slot staging --target-slot production
  
  deploy-frontend:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

## 5. Secrets Management

### GitHub Secrets

Configure the following secrets in your GitHub repository:

1. `AZURE_CREDENTIALS`: Azure service principal credentials
2. `VERCEL_TOKEN`: Vercel API token
3. `VERCEL_ORG_ID`: Vercel organization ID
4. `VERCEL_PROJECT_ID`: Vercel project ID

### Azure Key Vault (Production)

For production, store sensitive credentials in Azure Key Vault:

```bash
# Create Key Vault
az keyvault create --name outreach-vault --resource-group outreach-rg --location eastus

# Add secrets
az keyvault secret set --vault-name outreach-vault --name "AZURE-OPENAI-KEY" --value "your-key"
az keyvault secret set --vault-name outreach-vault --name "PROXYCURL-API-KEY" --value "your-key"
az keyvault secret set --vault-name outreach-vault --name "RESEND-API-KEY" --value "your-key"
az keyvault secret set --vault-name outreach-vault --name "SUPABASE-ANON-KEY" --value "your-key"

# Grant access to App Service
az keyvault set-policy --name outreach-vault --object-id $(az webapp identity show -g outreach-rg -n recruiter-api --query principalId -o tsv) --secret-permissions get list
```

## 6. Monitoring and Logging

### Application Insights

Monitor application performance and errors using Azure Application Insights:

1. Create a dashboard for key metrics:

```bash
az portal dashboard create --name "Recruiter-Outreach-Dashboard" --resource-group outreach-rg --location eastus --dashboard-path dashboard.json
```

2. Set up alerts for critical errors:

```bash
az monitor alert create --name "High Error Rate" --resource-group outreach-rg --scopes $(az monitor app-insights component show -g outreach-rg --app recruiter-api-insights --query id -o tsv) --condition "requests/failed > 5"
```

### Supabase Logging

SQL query performance can be monitored in the Supabase dashboard:

1. Enable query performance insights
2. Set up daily reports for slow queries

## 7. Backup and Disaster Recovery

### Database Backups

Supabase automatically creates daily backups. For additional protection:

```bash
# Schedule additional backups using Supabase CLI
supabase db dump -f backup-$(date +%Y%m%d).sql
```

### Application Backups

Back up application code and configuration:

```bash
# Back up environment variables
az webapp config appsettings list -g outreach-rg -n recruiter-api > appsettings-backup.json
```

## 8. Post-Deployment Verification

After deployment, run these verification checks:

1. Test authentication flow
2. Verify job search functionality
3. Test PDF capture
4. Confirm email sending works
5. Check all monitoring is active

## 9. Troubleshooting Common Issues

### Playwright Browser Launch Failures

If the worker fails to launch browsers:

```bash
# Check browser dependencies
ldd $(which playwright)

# Ensure X virtual framebuffer is running
Xvfb :0 -screen 0 1280x720x24 &
export DISPLAY=:0
```

### API Rate Limiting

If you encounter rate limiting with LinkedIn:

1. Adjust `RATE_LIMIT_PROFILE_FETCH` to a lower value
2. Check `errors` table for rate limit entries
3. Implement longer backoff periods

### Email Delivery Issues

If emails aren't being delivered:

1. Verify Resend domain verification
2. Check email templates for spam triggers
3. Review Resend dashboard for delivery metrics