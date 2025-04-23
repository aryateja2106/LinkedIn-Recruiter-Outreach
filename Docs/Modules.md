# Module Map (monorepo)

This document outlines the key modules in the LinkedIn Recruiter Outreach application, their responsibilities, and how they interact.

## Core Modules

|Path|Module|Description|
|---|---|---|
|`backend/agents/job_search.py`|`JobSearchAgent`|Keyword → LinkedIn jobs list using jobs-guest API|
|`backend/agents/recruiter.py`|`RecruiterAgent`|Job URL → recruiter PDF + email|
|`backend/agents/personalise.py`|`PersonaliseAgent`|Generate Markdown note via LLM|
|`backend/worker.py`|Playwright Worker|Runs queued page actions, stores PDFs → Supabase.Storage|
|`frontend/components/`|React components|`SearchForm`, `RecruiterCard`, `MarkdownEditor`|
|`frontend/lib/api.ts`|API wrapper|Fetch `/search`, `/draft`|

## Agent Architecture Details

### JobSearchAgent

**Responsibility**: Convert user search terms into a list of relevant LinkedIn job postings.

**Implementation**:

- Lightweight agent (can use simple prompt without complex reasoning)
- Makes API call to LinkedIn's jobs-guest endpoint
- Filters results by relevance and recency
- Returns structured job data

**Input/Output**:

```python
# Input
{
  "keywords": "AI Product Manager",
  "location": "San Francisco, CA",
  "posted_within_days": 7
}

# Output
[
  {
    "job_id": "3612039485",
    "title": "AI Product Manager",
    "company": "TechCorp Inc.",
    "url": "https://www.linkedin.com/jobs/view/3612039485",
    "posted_date": "2023-04-18T14:30:00Z"
  },
  # ... more jobs
]
```

### RecruiterAgent

**Responsibility**: Visit job postings, identify recruiters, and capture their LinkedIn profiles.

**Implementation**:

- ReAct pattern agent (reason-act-observe loop)
- Uses Playwright browser automation via worker
- Implements fallback strategies when direct recruiter link isn't found
- Captures profile as PDF document
- Attempts to find email via Proxycurl API

**Input/Output**:

```python
# Input
{
  "job_id": "3612039485",
  "job_url": "https://www.linkedin.com/jobs/view/3612039485"
}

# Output
{
  "recruiter_id": "johndoe",
  "name": "John Doe",
  "profile_url": "https://www.linkedin.com/in/johndoe/",
  "email": "john.doe@techcorp.com",
  "pdf_path": "profiles/johndoe-12345.pdf",
  "job_id": "3612039485"
}
```

### PersonaliseAgent

**Responsibility**: Generate personalized outreach based on job details and recruiter profile.

**Implementation**:

- Planning agent (first summarizes content, then crafts message)
- Uses Azure OpenAI o4-mini with temperature=0.7
- Fallback to local Llama if offline
- Applies templates with personalized elements

**Input/Output**:

```python
# Input
{
  "recruiter_id": "johndoe",
  "job_id": "3612039485",
  "pdf_url": "https://storage.supabase.com/profiles/johndoe-12345.pdf",
  "user_profile": {
    "name": "Alex Garcia",
    "current_role": "Product Manager",
    "key_experience": "AI startup, 3 years"
  }
}

# Output
{
  "subject": "AI Product Manager candidate – quick intro",
  "body_md": "Hi John,\n\nI noticed you're hiring for **AI Product Manager** at TechCorp...",
  "status": "draft"
}
```

## Worker Implementation

The Playwright worker is a critical security-sensitive component:

```python
# Simplified structure of backend/worker.py
class PlaywrightWorker:
    def __init__(self, cookies: str):
        self.browser = playwright.chromium.launch(headless=False)
        self.context = self.browser.new_context()
        self._set_cookies(cookies)  # LinkedIn auth cookies
        
    async def open_job(self, url: str) -> dict:
        page = await self.context.new_page()
        await page.goto(url)
        # Rate limiting and anti-detection measures
        await self._random_delay(1.5, 4.0)
        # [...implementation...]
        
    async def find_recruiter(self, page) -> dict:
        try:
            # Try to click "See hiring team" or similar
            # [...implementation...]
        except:
            # Fallback: company page → people → filter recruiters
            # [...implementation...]
            
    async def download_pdf(self, profile_url: str) -> str:
        page = await self.context.new_page()
        await page.goto(profile_url)
        pdf_buffer = await page.pdf()
        # Upload to Supabase Storage
        # [...implementation...]
        
    def _set_cookies(self, cookie_str: str):
        # Process and set cookies securely
        # [...implementation...]
        
    async def _random_delay(self, min_sec: float, max_sec: float):
        # Add human-like delay with jitter
        await asyncio.sleep(random.uniform(min_sec, max_sec))
```

## Frontend Components

### SearchForm

User interface for searching jobs with keywords and filters.

### RecruiterCard

Displays recruiter information and job details, with options to draft messages.

### MarkdownEditor

Rich editor for customizing outreach messages before sending.

## Data Contracts

```python
# backend/schemas.py
class Job(BaseModel):
    id: int
    title: str
    company: str
    location: str
    url: str
    date_found: datetime = Field(default_factory=datetime.now)

class Recruiter(BaseModel):
    id: str
    name: str
    profile_url: HttpUrl
    email: Optional[str]
    pdf_path: str
    job_id: int
    
class Outreach(BaseModel):
    id: UUID = Field(default_factory=uuid4)
    recruiter_id: str
    subject: str
    body_md: str
    status: Literal["draft", "sent", "replied"] = "draft"
    sent_at: Optional[datetime] = None
```

## LangGraph Flow

The agent workflow is orchestrated using LangGraph's flow structure:

```python
from langgraph.graph import Graph
from backend.agents import JobSearchAgent, RecruiterAgent, PersonaliseAgent

def create_agent_flow():
    with Graph() as g:
        jobs = g.add(JobSearchAgent)
        recs = g.add(RecruiterAgent).after(jobs)
        note = g.add(PersonaliseAgent).after(recs)
        g.set_entry_point(jobs)
    
    return g.compile()
```

## Error Handling

Each module implements robust error handling:

```python
try:
    result = await agent.run(input_data)
except RateLimitError:
    # Implement exponential backoff
    await asyncio.sleep(backoff_time)
    backoff_time *= 2
except RecruiterNotFoundError:
    # Switch to fallback strategy
    result = await fallback_recruiter_search(input_data)
except Exception as e:
    # Log structured error
    await log_error(user_id, "agent_error", {"agent": agent_name, "input": input_data, "error": str(e)})
    raise HTTPException(status_code=500, detail="Agent processing failed")
```

## Integration Points

These are the key integration points between modules:

1. **Frontend → Backend**: REST API calls from Next.js to FastAPI
2. **Backend → Agents**: LangGraph orchestration
3. **Agents → Worker**: Task queue for browser automation
4. **Agents → External Services**: API calls to Proxycurl, Azure OpenAI
5. **Worker → Storage**: PDF uploads to Supabase