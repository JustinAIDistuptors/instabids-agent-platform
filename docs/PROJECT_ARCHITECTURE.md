# 🏗️ InstaBids Project Architecture

> **Overview**: Complete technical architecture for the AI-driven multi-agent bidding platform.

## 🎯 System Overview

```mermaid
graph TB
    subgraph "Frontend Layer"
        A[Next.js App]
        B[React Components]
        C[Storybook Library]
    end
    
    subgraph "API Layer"
        D[FastAPI Server]
        E[REST Endpoints]
        F[WebSocket Gateway]
    end
    
    subgraph "Agent Layer"
        G[HomeownerAgent]
        H[BidCardAgent]
        I[OutboundRecruiterAgent]
        J[ContractorAgent]
        K[CoreOrchestratorAgent]
        L[PromptSelectorAgent]
    end
    
    subgraph "Data Layer"
        M[(Supabase DB)]
        N[pgvector RAG]
        O[Storage Buckets]
    end
    
    subgraph "External Services"
        P[OpenAI API]
        Q[Google Vertex AI]
        R[Email/SMS Services]
    end
    
    A --> D
    D --> G
    D --> K
    K --> L
    K --> G
    K --> H
    K --> I
    K --> J
    
    G --> M
    H --> M
    I --> M
    J --> M
    
    G --> P
    H --> P
    
    M --> N
    M --> O
```

## 📦 Component Architecture

### 1. Frontend Components

#### Next.js Application
- **Location**: `/frontend`
- **Purpose**: User interface for homeowners and contractors
- **Key Features**:
  - Server-side rendering for SEO
  - Real-time updates via WebSocket
  - Progressive Web App capabilities
  - Responsive design system

#### Component Library
```typescript
// Key UI Components
- ProjectCreationWizard
- BidCardDisplay
- ChatInterface
- ContractorDashboard
- BidSubmissionForm
- NotificationCenter
```

### 2. API Architecture

#### FastAPI Structure
```python
api/
├── main.py              # Application entry
├── routers/
│   ├── projects.py      # Project endpoints
│   ├── agents.py        # Agent interactions
│   ├── bids.py          # Bidding system
│   └── messages.py      # Messaging
├── middleware/
│   ├── auth.py          # Authentication
│   └── logging.py       # Request logging
└── dependencies/
    ├── database.py      # DB connections
    └── agents.py        # Agent factory
```

#### Key Endpoints
| Method | Path | Purpose |
|--------|------|----------|
| POST | `/projects` | Create new project |
| GET | `/projects/{id}` | Get project details |
| POST | `/projects/{id}/messages` | Send message |
| GET | `/projects/{id}/bid-card` | Get bid card |
| POST | `/bids` | Submit contractor bid |
| GET | `/agents/status` | Agent health check |

### 3. Agent Architecture

#### Agent Hierarchy
```
CoreOrchestratorAgent (Master)
├── PromptSelectorAgent (Meta)
├── HomeownerAgent (Primary)
├── BidCardAgent (Processing)
├── OutboundRecruiterAgent (Matching)
└── ContractorAgent (Bidding)
```

#### Agent Communication Patterns

##### Pattern 1: Direct Tool Invocation
```python
# Agent directly calls tool
result = await agent.call_tool(
    "analyze_project_image",
    image_path="roof_damage.jpg"
)
```

##### Pattern 2: Event-Based (A2A)
```python
# Agent emits event
await emit_event(ProjectCreatedEvent(
    project_id=project_id,
    owner_id=user_id
))

# Another agent handles
@on_event(ProjectCreatedEvent)
async def handle_project_created(event):
    # Generate bid card
    pass
```

##### Pattern 3: Workflow Orchestration
```python
# Sequential workflow
workflow = SequentialAgent([
    CreateProjectTask(),
    GenerateBidCardTask(),
    InviteContractorsTask()
])
```

### 4. Data Architecture

#### Database Schema

##### Core Tables
```sql
-- Projects table
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES auth.users(id),
    title TEXT NOT NULL,
    description TEXT,
    status TEXT DEFAULT 'draft',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Bid cards table
CREATE TABLE bid_cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id),
    category TEXT NOT NULL,
    job_type TEXT,
    budget_range JSONB,
    timeline JSONB,
    scope_json JSONB,
    photo_meta JSONB,
    ai_confidence FLOAT,
    status TEXT DEFAULT 'draft',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Messages table
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id),
    sender_role TEXT NOT NULL,
    content TEXT NOT NULL,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User preferences table
CREATE TABLE user_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id),
    key TEXT NOT NULL,
    value JSONB,
    confidence FLOAT DEFAULT 1.0,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, key)
);
```

#### Vector Store (RAG)
```sql
-- Knowledge base with embeddings
CREATE TABLE project_knowledge (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),
    source_type TEXT,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Similarity search function
CREATE FUNCTION match_project_knowledge(
    query_embedding VECTOR(1536),
    match_threshold FLOAT DEFAULT 0.7,
    match_count INT DEFAULT 5
)
RETURNS TABLE (...) AS $$ ... $$;
```

### 5. Security Architecture

#### Authentication Flow
```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant Supabase
    participant Agent
    
    User->>Frontend: Login
    Frontend->>Supabase: Authenticate
    Supabase-->>Frontend: JWT Token
    Frontend->>API: Request + JWT
    API->>API: Validate JWT
    API->>Agent: Execute (Service Role)
    Agent->>Supabase: DB Operations
    Supabase-->>Agent: Results
    Agent-->>API: Response
    API-->>Frontend: Data
```

#### Row Level Security
```sql
-- Homeowners see own projects
CREATE POLICY "own_projects" ON projects
    FOR ALL USING (auth.uid() = owner_id);

-- Messages visible to project participants
CREATE POLICY "project_messages" ON messages
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM projects
            WHERE projects.id = messages.project_id
            AND projects.owner_id = auth.uid()
        )
    );
```

## 🔄 Data Flow Diagrams

### Project Creation Flow
```mermaid
sequenceDiagram
    participant H as Homeowner
    participant UI as Frontend
    participant API as FastAPI
    participant HA as HomeownerAgent
    participant BA as BidCardAgent
    participant DB as Supabase
    participant AI as OpenAI
    
    H->>UI: Describe project + upload images
    UI->>API: POST /projects
    API->>HA: Create project
    HA->>DB: Store initial data
    HA->>AI: Analyze images
    AI-->>HA: Image analysis
    HA->>HA: Slot filling Q&A
    HA->>BA: Generate bid card
    BA->>AI: Categorize project
    AI-->>BA: Category + confidence
    BA->>DB: Save bid card
    BA-->>HA: Bid card created
    HA->>DB: Save final project
    HA-->>API: Project ID
    API-->>UI: Success + redirect
    UI-->>H: Show project dashboard
```

### Contractor Matching Flow
```mermaid
sequenceDiagram
    participant BA as BidCardAgent
    participant ORA as OutboundRecruiterAgent
    participant DB as Supabase
    participant ML as Matching Logic
    participant NS as Notification Service
    participant C as Contractor
    
    BA->>ORA: BidCardReady event
    ORA->>DB: Fetch bid card details
    ORA->>DB: Query contractor pool
    ORA->>ML: Match contractors
    ML->>ML: Apply criteria
    ML-->>ORA: Ranked matches
    ORA->>NS: Send invitations
    NS->>C: Email/SMS notification
    C->>UI: Click invitation link
    UI->>API: GET /projects/{id}/bid-card
    API-->>UI: Display bid opportunity
```

## 🚀 Deployment Architecture

### Container Structure
```yaml
services:
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    environment:
      - NEXT_PUBLIC_API_URL
      - NEXT_PUBLIC_SUPABASE_URL
      
  api:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL
      - OPENAI_API_KEY
      - SUPABASE_SERVICE_KEY
      
  agents:
    build: ./agents
    environment:
      - GOOGLE_API_KEY
      - GOOGLE_CLOUD_PROJECT
      
  postgres:
    image: supabase/postgres
    volumes:
      - ./db/migrations:/docker-entrypoint-initdb.d
```

### Production Infrastructure
```mermaid
graph LR
    subgraph "Google Cloud"
        A[Cloud Load Balancer]
        B[Cloud Run - API]
        C[Vertex AI Agent Engine]
        D[Cloud Storage]
    end
    
    subgraph "Vercel"
        E[Next.js Frontend]
    end
    
    subgraph "Supabase Cloud"
        F[PostgreSQL]
        G[Vector Store]
        H[File Storage]
    end
    
    A --> B
    B --> C
    E --> A
    B --> F
    C --> F
    E --> H
```

## 📏 Scaling Considerations

### Agent Scaling
- Horizontal scaling via Vertex AI Agent Engine
- Agent pool management for concurrent requests
- Caching frequent agent responses

### Database Scaling
- Connection pooling with pgBouncer
- Read replicas for analytics
- Partitioning for large tables (messages, events)

### API Scaling
- Auto-scaling Cloud Run instances
- Redis for session management
- CDN for static assets

## 🔒 Security Measures

1. **API Security**
   - JWT validation on all endpoints
   - Rate limiting per user
   - CORS configuration
   - Input validation with Pydantic

2. **Agent Security**
   - Service role for privileged operations
   - Audit logging for all agent actions
   - Sandboxed code execution

3. **Data Security**
   - Encryption at rest (Supabase)
   - TLS for all connections
   - PII handling compliance
   - Regular security audits

## 📊 Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| API Response Time | <200ms | TBD |
| Agent Response Time | <3s | TBD |
| Project Creation | <5s | TBD |
| Bid Card Generation | <10s | TBD |
| Contractor Matching | <30s | TBD |
| Concurrent Users | 10,000 | TBD |

## 🔧 Development Guidelines

1. **Code Organization**
   - Domain-driven design
   - Clear separation of concerns
   - Dependency injection patterns

2. **Testing Strategy**
   - Unit tests for business logic
   - Integration tests for workflows
   - E2E tests for critical paths

3. **Documentation**
   - API documentation via OpenAPI
   - Agent behavior documentation
   - Deployment runbooks

---

**Note**: This architecture is designed for scalability and maintainability. All components are loosely coupled to allow independent evolution.
