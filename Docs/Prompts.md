# Prompt Library

This document outlines the system prompts, tool definitions, and message templates used throughout the LinkedIn Recruiter Outreach application.

## 1 · System Messages

System messages define the agent's role and constraints for each LLM call.

### 1.1 General Agent System Message

```yaml
- role: system
  content: |
    You are an elite job-application assistant.
    You must ALWAYS output valid Markdown unless instructed otherwise.
    If you are calling a tool, respond **only** with the JSON arguments.
```

### 1.2 JobSearchAgent System Message

```yaml
- role: system
  content: |
    You are a specialized job search assistant.
    Your goal is to help users find relevant LinkedIn job postings.
    You can ONLY call the get_jobs tool with appropriate parameters.
    Never invent or hallucinate job listings.
```

### 1.3 RecruiterAgent System Message

```yaml
- role: system
  content: |
    You are a specialized recruiter finder.
    Given a job URL, your task is to identify and extract information about the recruiter.
    You will navigate LinkedIn through browser actions to find the recruiter.
    
    When you see a job page:
    1. Look for "Meet the hiring team" or "See connections" links
    2. Click on the link to view the recruiter's profile
    3. If multiple recruiters appear, select the one with "recruiter" or "talent" in their title
    4. Save the profile as a PDF
    5. Try to find their email address
    
    If no recruiter is directly linked, use the fallback strategy to search the company page.
    
    You should output a structured response with the recruiter's information.
```

### 1.4 PersonaliseAgent System Message

```yaml
- role: system
  content: |
    You are an expert email copywriter for job applications.
    Your task is to craft highly personalized outreach messages that:
    1. Reference specific details from the recruiter's profile
    2. Connect the candidate's experience to the job requirements
    3. Are concise, professional, and engaging
    4. Include a clear call to action
    
    First, analyze the recruiter's profile and job description.
    Then, draft a personalized message following the template format.
    
    Always maintain a professional, friendly tone and avoid generic statements.
```

## 2 · Tool Definitions (LangGraph → JSONSchema)

These tool definitions use a TypeScript-like schema format that gets converted to JSON Schema for the LLM.

### 2.1 JobSearchAgent Tools

```python
from zod import z  # TS side would use zod

GetJobs = {
  "name": "get_jobs",
  "description": "Search LinkedIn for jobs by keyword & location",
  "parameters": z.object({
      "keywords": z.string(),
      "location": z.string().optional(),
      "posted_within_days": z.number().optional()
  }).to_json_schema(),
  "execute": get_jobs_fn,
}
```

### 2.2 RecruiterAgent Tools

```python
OpenJob = {
  "name": "open_job",
  "description": "Open a LinkedIn job page in the browser",
  "parameters": z.object({
      "job_url": z.string().url()
  }).to_json_schema(),
  "execute": open_job_fn,
}

FindRecruiter = {
  "name": "find_recruiter",
  "description": "Find the recruiter from the current job page",
  "parameters": z.object({
      "selector_strategy": z.enum(["hiring_team", "connections", "company_page"])
  }).to_json_schema(),
  "execute": find_recruiter_fn,
}

DownloadProfile = {
  "name": "download_profile",
  "description": "Download the recruiter's profile as PDF",
  "parameters": z.object({
      "profile_url": z.string().url()
  }).to_json_schema(),
  "execute": download_profile_fn,
}

FindEmail = {
  "name": "find_email",
  "description": "Try to find the recruiter's email address",
  "parameters": z.object({
      "profile_url": z.string().url(),
      "name": z.string(),
      "company": z.string()
  }).to_json_schema(),
  "execute": find_email_fn,
}
```

### 2.3 PersonaliseAgent Tools

```python
SummarisePDF = {
  "name": "summarise_pdf",
  "description": "Extract key information from a recruiter's LinkedIn PDF",
  "parameters": z.object({
      "pdf_url": z.string().url()
  }).to_json_schema(),
  "execute": summarise_pdf_fn,
}

GetJobDetails = {
  "name": "get_job_details",
  "description": "Get detailed information about a job posting",
  "parameters": z.object({
      "job_id": z.string()
  }).to_json_schema(),
  "execute": get_job_details_fn,
}

GenerateEmailDraft = {
  "name": "generate_email_draft",
  "description": "Create a personalized email draft",
  "parameters": z.object({
      "recruiter_name": z.string(),
      "job_title": z.string(),
      "company": z.string(),
      "profile_highlights": z.array(z.string()),
      "candidate_experience": z.string(),
      "candidate_name": z.string()
  }).to_json_schema(),
  "execute": generate_email_draft_fn,
}
```

## 3 · Message Templates

### 3.1 LinkedIn Connection Note (≤ 300 chars, plain text)

```
Hi {{firstName}} – just applied for {{jobTitle}} at {{company}}.
Your note on {{profileHook}} caught my eye; I've done similar at {{yourProject}}.
Would love to connect.
```

### 3.2 Email Template (Markdown for Resend)

```markdown
**Subject**: {{jobTitle}} candidate – quick intro

Hi {{firstName}},

I noticed you're hiring for **{{jobTitle}}** at **{{company}}**.  After reading _{{profileHook}}_ on your LinkedIn, I felt compelled to reach out.

> **Why me**  
> {{twoSentenceProof}}

If the role is still open, could we schedule a 10-minute call next week?

Best,
{{candidateName}}
```

### 3.3 Tool-Use Example Call (LLM → RecruiterAgent)

```json
{
  "tool": "get_recruiter_pdf",
  "args": { "profile_url": "https://linkedin.com/in/..." }
}
```

## 4 · Few-Shot Examples

### 4.1 Good vs. Bad Outreach Examples

#### Bad Example:

```
Hi,

I saw your job posting for Product Manager and I'm interested. I have 3 years of experience and am looking for new opportunities.

Can we connect?

Thanks,
Alex
```

#### Good Example:

```
Hi Sarah,

I noticed you're leading the AI Product team at TechCorp and are hiring for a Senior PM. Your recent article on ethical AI development resonated with me, as I implemented similar principles while building the recommendation system at DataCo that increased user engagement by 35%.

Would you be open to a quick call next Tuesday to discuss how my experience might align with your team's needs?

Best,
Alex
```

### 4.2 Tool Calling Examples

```
User: Find a recruiter for this job: https://www.linkedin.com/jobs/view/3612039485

Agent:
{
  "tool": "open_job",
  "args": { 
    "job_url": "https://www.linkedin.com/jobs/view/3612039485" 
  }
}

System: Tool returned: { "status": "success", "page_loaded": true }

Agent:
{
  "tool": "find_recruiter",
  "args": { 
    "selector_strategy": "hiring_team" 
  }
}

System: Tool returned: { 
  "status": "success", 
  "recruiter_found": true, 
  "profile_url": "https://www.linkedin.com/in/sarahjohnson/",
  "name": "Sarah Johnson",
  "title": "Technical Recruiter at TechCorp" 
}
```

## 5 · Prompt Lifecycle Management

The system uses a structured approach to prompt management:

1. **Storage**: All prompts are stored in YAML files in the `frontend/prompts/` directory
2. **Versioning**: Prompts are versioned with git alongside code
3. **Validation**: CI pipeline tests prompts for token length and structure
4. **A/B Testing**: Key prompts can be A/B tested in production (future)
5. **User Customization**: Users can customize prompts via settings UI

## 6 · LLM Configuration

### 6.1 Azure OpenAI (Primary)

```python
# LLM configuration for Azure OpenAI
azure_config = {
    "api_type": "azure",
    "api_version": "2023-05-15",
    "deployment_name": "o4-mini",
    "temperature": 0.7,  # Higher for PersonaliseAgent
    "max_tokens": 1024,
    "top_p": 0.95,
    "frequency_penalty": 0.0,
    "presence_penalty": 0.0,
}
```

### 6.2 Local Llama (Fallback)

```python
# Configuration for local Llama using llama.cpp
llama_config = {
    "model_path": "models/llama-2-14b.gguf",
    "n_ctx": 2048,
    "temperature": 0.8,
    "top_p": 0.95,
    "repeat_penalty": 1.1,
    "threads": 4,  # Adjust based on CPU
}
```