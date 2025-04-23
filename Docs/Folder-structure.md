# Folder Structure

This document outlines the organization of the project's codebase, providing a guide to navigating the repository.

## Root Structure

```
repo/
├─ backend/           # Python/FastAPI backend code
├─ frontend/          # Next.js frontend code
├─ docs/              # Documentation files
├─ pdf/               # Local storage for recruiter PDFs
├─ supabase/          # Supabase migrations and config
├─ .env.sample        # Template for environment variables
├─ .gitignore         # Git ignore patterns
├─ requirements.txt   # Python dependencies
├─ package.json       # JS dependencies and scripts
└─ README.md          # Project overview
```

## Backend Structure

```
backend/
├─ agents/            # AI agent implementation
│  ├─ __init__.py     # Package exports
│  ├─ job_search.py   # JobSearchAgent implementation
│  ├─ recruiter.py    # RecruiterAgent implementation
│  └─ personalise.py  # PersonaliseAgent implementation
├─ models/            # Data models and schemas
│  ├─ __init__.py     # Package exports
│  ├─ job.py          # Job-related models
│  ├─ recruiter.py    # Recruiter-related models
│  └─ outreach.py     # Outreach-related models
├─ services/          # External service integrations
│  ├─ __init__.py     # Package exports
│  ├─ supabase.py     # Supabase client and helpers
│  ├─ proxycurl.py    # Proxycurl API client
│  ├─ azure_openai.py # Azure OpenAI client
│  └─ resend.py       # Resend email client
├─ routes/            # API route handlers
│  ├─ __init__.py     # Package exports
│  ├─ auth.py         # Authentication routes
│  ├─ jobs.py         # Job search routes
│  ├─ recruiters.py   # Recruiter routes
│  └─ outreach.py     # Outreach/email routes
├─ worker.py          # Playwright worker process
├─ main.py            # FastAPI application entry point
├─ config.py          # Configuration and env loading
├─ schemas.py         # Pydantic schemas
└─ utils/             # Utility functions
   ├─ __init__.py     # Package exports
   ├─ linkedin.py     # LinkedIn-specific helpers
   ├─ pdf.py          # PDF processing utilities
   └─ security.py     # Security-related utilities
```

## Frontend Structure

```
frontend/
├─ components/        # React components
│  ├─ ui/             # Base UI components
│  │  ├─ Button.tsx   # Button component
│  │  ├─ Input.tsx    # Input component
│  │  └─ ...          # Other UI components
│  ├─ layout/         # Layout components
│  │  ├─ Header.tsx   # Page header
│  │  ├─ Sidebar.tsx  # Navigation sidebar
│  │  └─ Footer.tsx   # Page footer
│  ├─ SearchForm.tsx  # Job search form
│  ├─ RecruiterCard.tsx  # Recruiter display card
│  ├─ MarkdownEditor.tsx # Outreach editor
│  └─ ...             # Other components
├─ pages/             # Next.js pages
│  ├─ _app.tsx        # Next.js app wrapper
│  ├─ index.tsx       # Home/dashboard page
│  ├─ search.tsx      # Job search page
│  ├─ draft.tsx       # Email draft page
│  ├─ settings.tsx    # User settings page
│  └─ api/            # Next.js API routes
│     ├─ auth/        # Auth-related API routes
│     └─ ...          # Other API routes
├─ lib/               # Shared utilities and hooks
│  ├─ api.ts          # API client functions
│  ├─ auth.ts         # Authentication utilities
│  ├─ supabase.ts     # Supabase client
│  └─ hooks/          # Custom React hooks
│     ├─ useJobs.ts   # Job search hook
│     └─ ...          # Other hooks
├─ public/            # Static assets
│  ├─ images/         # Image assets
│  └─ ...             # Other static files
├─ styles/            # CSS and styling
│  ├─ globals.css     # Global styles
│  └─ ...             # Component-specific styles
├─ prompts/           # LLM prompt templates
│  ├─ system.yaml     # System prompts
│  ├─ templates.yaml  # Message templates
│  └─ examples.yaml   # Few-shot examples
├─ next.config.js     # Next.js configuration
└─ tsconfig.json      # TypeScript configuration
```

## Supabase Structure

```
supabase/
├─ migrations/        # SQL migration files
│  ├─ 20250401000000_initial_schema.sql  # Initial schema
│  └─ ...             # Additional migrations
├─ seed.sql           # Seed data for development
└─ config.toml        # Supabase configuration
```

## Documentation Structure

```
docs/
├─ PRD.md             # Product Requirements Document
├─ ARCHITECTURE.md    # High-level architecture
├─ DATABASE_SCHEMA.md # Database schema details
├─ MODULES.md         # Module documentation
├─ PROMPTS.md         # AI prompt documentation
├─ FOLDER_STRUCTURE.md # This file
├─ DEPLOYMENT.md      # Deployment instructions
└─ SECURITY.md        # Security considerations
```

## Key Files and Their Purpose

### Configuration Files

- **`.env.sample`**: Template for environment variables needed by the application
- **`package.json`**: NPM package definition, scripts, and JS dependencies
- **`requirements.txt`**: Python dependencies
- **`next.config.js`**: Next.js configuration
- **`tsconfig.json`**: TypeScript configuration

### Entry Points

- **`backend/main.py`**: FastAPI application entry point
- **`backend/worker.py`**: Playwright worker process entry point
- **`frontend/pages/_app.tsx`**: Next.js application wrapper
- **`frontend/pages/index.tsx`**: Main landing page

### Key Implementation Files

- **`backend/agents/*.py`**: AI agent implementations
- **`backend/services/*.py`**: External service integrations
- **`frontend/components/SearchForm.tsx`**: Job search interface
- **`frontend/components/RecruiterCard.tsx`**: Recruiter display component
- **`frontend/components/MarkdownEditor.tsx`**: Email editing interface
- **`frontend/lib/api.ts`**: API client for frontend-backend communication

## Development Workflow

The development workflow typically follows this pattern:

1. Make changes to frontend components or pages
2. Update backend routes or agents as needed
3. Modify SQL schema in Supabase migrations if required
4. Add or update tests
5. Run tests and linting
6. Commit and push changes
7. CI/CD pipeline deploys to test environment
8. After review, deploy to production

## File Naming Conventions

- React components: PascalCase (e.g., `RecruiterCard.tsx`)
- Utility modules: camelCase (e.g., `api.ts`)
- Python modules: snake_case (e.g., `job_search.py`)
- SQL migrations: timestamp_description.sql (e.g., `20250401000000_initial_schema.sql`)