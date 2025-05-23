# 🔗 Supabase Integration Patterns for InstaBids

> **Purpose**: Comprehensive guide for integrating Supabase with ADK agents for data persistence, RAG, and real-time features.

## 📋 Table of Contents

1. [Initial Setup](#initial-setup)
2. [Database Schema](#database-schema)
3. [Row Level Security](#row-level-security)
4. [Agent Tools](#agent-tools)
5. [Vector Storage & RAG](#vector-storage--rag)
6. [Storage Patterns](#storage-patterns)
7. [Real-time Subscriptions](#real-time-subscriptions)
8. [Testing Strategies](#testing-strategies)

## 🚀 Initial Setup

### Local Development

```bash
# Install Supabase CLI
npm install -g supabase

# Initialize project
supabase init

# Start local instance
supabase start

# Apply migrations
supabase db push
```

### Python Client Setup

```python
# src/instabids/data/client.py
from supabase import create_client, Client
from typing import Optional
import os

class SupabaseClient:
    _instance: Optional[Client] = None
    
    @classmethod
    def get_client(cls, use_service_key: bool = False) -> Client:
        """Get singleton Supabase client."""
        if cls._instance is None:
            url = os.getenv("SUPABASE_URL")
            key = os.getenv("SUPABASE_SERVICE_KEY" if use_service_key else "SUPABASE_ANON_KEY")
            cls._instance = create_client(url, key)
        return cls._instance
```

## 🗄️ Database Schema

### Migration Files Structure

```
db/migrations/
├── 001_initial_schema.sql
├── 002_bid_cards.sql
├── 003_messages_and_preferences.sql
├── 004_contractors.sql
├── 005_vector_store.sql
└── 006_indexes_and_functions.sql
```

### Core Schema Example

```sql
-- 001_initial_schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";

-- Enable RLS
ALTER TABLE auth.users ENABLE ROW LEVEL SECURITY;

-- Projects table
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'completed', 'cancelled')),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Automatic updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_projects_updated_at BEFORE UPDATE ON projects
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

## 🔒 Row Level Security

### RLS Patterns

```sql
-- User owns resource
CREATE POLICY "users_own_projects" ON projects
    FOR ALL USING (auth.uid() = owner_id);

-- Service role bypass
CREATE POLICY "service_role_all_projects" ON projects
    FOR ALL USING (auth.role() = 'service_role');

-- Shared resource access
CREATE POLICY "contractors_view_bid_cards" ON bid_cards
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM project_invitations
            WHERE project_invitations.project_id = bid_cards.project_id
            AND project_invitations.contractor_id = auth.uid()
        )
    );

-- Public read, authenticated write
CREATE POLICY "public_read_categories" ON categories
    FOR SELECT USING (true);
    
CREATE POLICY "admin_write_categories" ON categories
    FOR INSERT USING (auth.jwt() ->> 'role' = 'admin');
```

### Testing RLS

```python
# tests/test_rls.py
import pytest
from supabase import create_client

@pytest.mark.asyncio
async def test_user_can_only_see_own_projects():
    # Create client with user JWT
    user_client = create_client(url, anon_key)
    user_client.auth.sign_in_with_password(email="user@test.com", password="test123")
    
    # Should only see own projects
    result = user_client.table("projects").select("*").execute()
    assert all(p["owner_id"] == user_client.auth.user().id for p in result.data)
```

## 🔧 Agent Tools

### Repository Pattern

```python
# src/instabids/data/repositories/base.py
from typing import Generic, TypeVar, Optional, List
from pydantic import BaseModel
from supabase import Client

T = TypeVar('T', bound=BaseModel)

class BaseRepository(Generic[T]):
    def __init__(self, client: Client, table_name: str, model_class: type[T]):
        self.client = client
        self.table_name = table_name
        self.model_class = model_class
    
    async def create(self, data: dict) -> T:
        result = self.client.table(self.table_name).insert(data).execute()
        return self.model_class(**result.data[0])
    
    async def get_by_id(self, id: str) -> Optional[T]:
        result = self.client.table(self.table_name).select("*").eq("id", id).single().execute()
        return self.model_class(**result.data) if result.data else None
    
    async def update(self, id: str, data: dict) -> T:
        result = self.client.table(self.table_name).update(data).eq("id", id).execute()
        return self.model_class(**result.data[0])
```

### ADK Tool Implementation

```python
# src/instabids/tools/supabase_tools.py
from google.adk.types import ToolContext
from instabids.data.client import SupabaseClient
import json

def save_user_preference(
    tool_context: ToolContext,
    key: str,
    value: Any,
    confidence: float = 1.0
) -> dict:
    """
    Saves a user preference to Supabase with confidence tracking.
    
    This tool stores user preferences learned during conversations,
    enabling personalization across sessions. Preferences are
    namespaced by user and can decay over time based on confidence.
    
    Args:
        tool_context (ToolContext): ADK context with user session
        key (str): Preference key (e.g., 'default_budget', 'style_preference')
        value (Any): Preference value (will be JSON serialized)
        confidence (float): Confidence in this preference (0.0-1.0)
        
    Returns:
        dict: Result with structure:
            {
                "status": "success",
                "preference_id": "uuid",
                "key": "default_budget",
                "previous_value": 5000,
                "new_value": 10000
            }
    """
    try:
        # Get user from context
        user_id = tool_context.state.get("app:current_user_id")
        if not user_id:
            return {"status": "error", "message": "No user in context"}
        
        # Get Supabase client (service role for user operations)
        client = SupabaseClient.get_client(use_service_key=True)
        
        # Check for existing preference
        existing = client.table("user_preferences").select("*").eq("user_id", user_id).eq("key", key).execute()
        
        data = {
            "user_id": user_id,
            "key": key,
            "value": json.dumps(value) if not isinstance(value, str) else value,
            "confidence": confidence
        }
        
        if existing.data:
            # Update existing
            result = client.table("user_preferences").update(data).eq("id", existing.data[0]["id"]).execute()
            return {
                "status": "success",
                "preference_id": result.data[0]["id"],
                "key": key,
                "previous_value": json.loads(existing.data[0]["value"]),
                "new_value": value
            }
        else:
            # Insert new
            result = client.table("user_preferences").insert(data).execute()
            return {
                "status": "success",
                "preference_id": result.data[0]["id"],
                "key": key,
                "new_value": value
            }
            
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

## 🧠 Vector Storage & RAG

### Setting up pgvector

```sql
-- 005_vector_store.sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Project knowledge base
CREATE TABLE project_knowledge (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding VECTOR(1536),  -- OpenAI embedding dimension
    source_type TEXT NOT NULL,  -- 'project', 'documentation', 'contractor_profile'
    source_id UUID,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX project_knowledge_embedding_idx ON project_knowledge 
    USING ivfflat (embedding vector_cosine_ops);

-- Similarity search function
CREATE OR REPLACE FUNCTION match_project_knowledge(
    query_embedding VECTOR(1536),
    match_threshold FLOAT DEFAULT 0.7,
    match_count INT DEFAULT 5
)
RETURNS TABLE (
    id BIGINT,
    content TEXT,
    source_type TEXT,
    metadata JSONB,
    similarity FLOAT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        pk.id,
        pk.content,
        pk.source_type,
        pk.metadata,
        1 - (pk.embedding <=> query_embedding) AS similarity
    FROM project_knowledge pk
    WHERE 1 - (pk.embedding <=> query_embedding) > match_threshold
    ORDER BY pk.embedding <=> query_embedding
    LIMIT match_count;
END;
$$ LANGUAGE plpgsql;
```

### RAG Implementation

```python
# src/instabids/tools/rag_tools.py
import openai
import numpy as np
from typing import List, Dict

def search_similar_projects(
    tool_context: ToolContext,
    query: str,
    limit: int = 5
) -> dict:
    """
    Searches for similar projects using semantic similarity.
    
    Uses OpenAI embeddings and pgvector to find projects with
    similar descriptions, enabling agents to reference past
    projects when generating estimates or suggestions.
    
    Args:
        tool_context (ToolContext): ADK context
        query (str): Search query
        limit (int): Maximum results to return
        
    Returns:
        dict: Search results with similar projects
    """
    try:
        # Generate embedding
        client = openai.Client()
        response = client.embeddings.create(
            model="text-embedding-ada-002",
            input=query
        )
        embedding = response.data[0].embedding
        
        # Search in Supabase
        supabase = SupabaseClient.get_client()
        results = supabase.rpc(
            "match_project_knowledge",
            {
                "query_embedding": embedding,
                "match_threshold": 0.7,
                "match_count": limit
            }
        ).execute()
        
        return {
            "status": "success",
            "results": results.data,
            "query": query
        }
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

## 📁 Storage Patterns

### File Upload Tool

```python
def upload_project_image(
    tool_context: ToolContext,
    project_id: str,
    image_data: bytes,
    filename: str
) -> dict:
    """
    Uploads project image to Supabase Storage.
    
    Stores images in project-specific folders with automatic
    optimization and thumbnail generation.
    
    Args:
        tool_context (ToolContext): ADK context
        project_id (str): Project UUID
        image_data (bytes): Image file data
        filename (str): Original filename
        
    Returns:
        dict: Upload result with public URL
    """
    try:
        client = SupabaseClient.get_client(use_service_key=True)
        bucket = "project-images"
        path = f"{project_id}/{filename}"
        
        # Upload to storage
        result = client.storage.from_(bucket).upload(
            path=path,
            file=image_data,
            file_options={"content-type": "image/jpeg"}
        )
        
        # Get public URL
        url = client.storage.from_(bucket).get_public_url(path)
        
        return {
            "status": "success",
            "url": url,
            "path": path,
            "size": len(image_data)
        }
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

### Storage Bucket Policies

```sql
-- Storage RLS policies
CREATE POLICY "Users can upload own project images"
    ON storage.objects FOR INSERT
    WITH CHECK (
        bucket_id = 'project-images' AND
        (storage.foldername(name))[1] IN (
            SELECT id::text FROM projects WHERE owner_id = auth.uid()
        )
    );

CREATE POLICY "Public can view project images"
    ON storage.objects FOR SELECT
    USING (bucket_id = 'project-images');
```

## 📡 Real-time Subscriptions

### Agent Real-time Handler

```python
# src/instabids/agents/realtime/handlers.py
from supabase import Client
import asyncio

class RealtimeHandler:
    def __init__(self, client: Client):
        self.client = client
        self.subscriptions = {}
    
    async def subscribe_to_project_messages(self, project_id: str, callback):
        """Subscribe to new messages for a project."""
        
        def on_message(payload):
            # Handle new message
            if payload["eventType"] == "INSERT":
                asyncio.create_task(callback(payload["new"]))
        
        channel = self.client.channel(f"messages:{project_id}")
        channel.on(
            "postgres_changes",
            {
                "event": "INSERT",
                "schema": "public",
                "table": "messages",
                "filter": f"project_id=eq.{project_id}"
            },
            on_message
        ).subscribe()
        
        self.subscriptions[project_id] = channel
    
    async def unsubscribe(self, project_id: str):
        if project_id in self.subscriptions:
            await self.subscriptions[project_id].unsubscribe()
            del self.subscriptions[project_id]
```

## 🧪 Testing Strategies

### Test Fixtures

```python
# tests/fixtures/supabase.py
import pytest
from supabase import create_client
import os

@pytest.fixture
async def test_supabase():
    """Provides test Supabase client."""
    # Use test database
    url = os.getenv("SUPABASE_TEST_URL", "http://localhost:54321")
    key = os.getenv("SUPABASE_TEST_SERVICE_KEY")
    
    client = create_client(url, key)
    
    # Clean test data before each test
    await cleanup_test_data(client)
    
    yield client
    
    # Clean up after test
    await cleanup_test_data(client)

async def cleanup_test_data(client):
    """Remove all test data."""
    # Delete in correct order (foreign keys)
    tables = ["messages", "bid_cards", "projects", "user_preferences"]
    
    for table in tables:
        client.table(table).delete().neq("id", "00000000-0000-0000-0000-000000000000").execute()
```

### Integration Test Example

```python
# tests/integration/test_project_flow.py
@pytest.mark.integration
async def test_complete_project_with_rag(test_supabase):
    # Create project
    project_data = {
        "title": "Kitchen Renovation",
        "description": "Complete kitchen remodel with new cabinets",
        "owner_id": TEST_USER_ID
    }
    
    project = test_supabase.table("projects").insert(project_data).execute()
    project_id = project.data[0]["id"]
    
    # Generate and store embedding
    embedding = await generate_embedding(project_data["description"])
    
    knowledge = test_supabase.table("project_knowledge").insert({
        "content": project_data["description"],
        "embedding": embedding,
        "source_type": "project",
        "source_id": project_id
    }).execute()
    
    # Search for similar
    results = test_supabase.rpc("match_project_knowledge", {
        "query_embedding": embedding,
        "match_count": 1
    }).execute()
    
    assert len(results.data) == 1
    assert results.data[0]["source_id"] == project_id
```

## 📊 Performance Optimization

### Connection Pooling

```python
# src/instabids/data/pool.py
from supabase import Client
from contextlib import asynccontextmanager
import asyncio

class SupabasePool:
    def __init__(self, url: str, key: str, size: int = 10):
        self.url = url
        self.key = key
        self.size = size
        self._pool = asyncio.Queue(maxsize=size)
        self._initialized = False
    
    async def initialize(self):
        """Create connection pool."""
        for _ in range(self.size):
            client = create_client(self.url, self.key)
            await self._pool.put(client)
        self._initialized = True
    
    @asynccontextmanager
    async def acquire(self):
        """Get client from pool."""
        if not self._initialized:
            await self.initialize()
            
        client = await self._pool.get()
        try:
            yield client
        finally:
            await self._pool.put(client)
```

### Batch Operations

```python
def batch_create_messages(
    tool_context: ToolContext,
    messages: List[Dict]
) -> dict:
    """
    Creates multiple messages in a single operation.
    
    Optimized for bulk operations like importing conversation
    history or system notifications.
    """
    try:
        client = SupabaseClient.get_client(use_service_key=True)
        
        # Batch insert (Supabase handles as single transaction)
        result = client.table("messages").insert(messages).execute()
        
        return {
            "status": "success",
            "created_count": len(result.data),
            "message_ids": [m["id"] for m in result.data]
        }
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

## 🔒 Security Best Practices

1. **Never expose service keys in frontend**
2. **Use RLS for all tables**
3. **Validate all inputs before database operations**
4. **Use prepared statements (Supabase client handles this)**
5. **Audit sensitive operations**
6. **Rotate keys regularly**
7. **Use environment-specific keys**

## 📚 Resources

- [Supabase Documentation](https://supabase.com/docs)
- [pgvector Guide](https://supabase.com/docs/guides/ai/vector-embeddings)
- [RLS Guide](https://supabase.com/docs/guides/auth/row-level-security)
- [Storage Guide](https://supabase.com/docs/guides/storage)

---

**Remember**: Supabase is not just a database - it's a complete backend platform. Use all its features to build a robust, scalable system!
