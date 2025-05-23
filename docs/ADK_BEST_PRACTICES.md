# 🎯 ADK Best Practices for InstaBids

> **Purpose**: Project-specific patterns and guidelines for using Google ADK 1.0.0 effectively.

## 📋 Table of Contents

1. [Agent Architecture Patterns](#agent-architecture-patterns)
2. [State Management](#state-management)
3. [Tool Design](#tool-design)
4. [Error Handling](#error-handling)
5. [Testing Strategies](#testing-strategies)
6. [Performance Optimization](#performance-optimization)

## 🏗️ Agent Architecture Patterns

### Pattern 1: Plain Python Agent Classes

**✅ CORRECT** (De-facto standard):
```python
# src/instabids/agents/homeowner/agent.py
from google.adk.agents import LlmAgent
from instabids.tools.supabase_tools import get_user_preferences, save_project

class HomeownerAgent(LlmAgent):
    def __init__(self):
        super().__init__(
            name="homeowner",
            model="gemini-2.0-flash-exp",
            instructions="""You are a helpful home improvement assistant...""",
            tools=[get_user_preferences, save_project]
        )

# CRITICAL: Export as 'agent'
agent = HomeownerAgent()
```

**❌ AVOID** (AdkApp wrapper - only for demos):
```python
# Don't use this pattern
from google.adk import AdkApp

class MyApp(AdkApp):
    # This pattern is abandoned by most teams
    pass
```

### Pattern 2: Multi-Agent Coordination

```python
# Using ADK's workflow agents
from google.adk.agents import SequentialAgent, ParallelAgent

# Sequential pipeline
project_pipeline = SequentialAgent(
    name="project_pipeline",
    agents=[
        HomeownerAgent(),
        BidCardAgent(),
        OutboundRecruiterAgent()
    ]
)

# Parallel execution
bid_collectors = ParallelAgent(
    name="bid_collectors",
    agents=[ContractorAgent(f"contractor_{i}") for i in range(5)]
)
```

### Pattern 3: Agent Communication via A2A

```python
# src/instabids/a2a_comm/events.py
from google.adk.events import Event
from pydantic import BaseModel

class ProjectCreatedEvent(BaseModel):
    project_id: str
    owner_id: str
    description: str
    
class BidCardReadyEvent(BaseModel):
    project_id: str
    bid_card_id: str
    confidence: float
```

## 🗄️ State Management

### State Prefix Convention

| Prefix | Purpose | Lifetime | Example |
|--------|---------|----------|----------|
| `user:` | User-specific data | Persistent | `user:budget_preference` |
| `app:` | Application-wide | Persistent | `app:contractor_categories` |
| `temp:` | Workflow temporary | Session | `temp:current_analysis` |

### State Access Patterns

```python
def save_user_preference(tool_context: ToolContext, key: str, value: Any) -> dict:
    """Save user preference with proper prefixing."""
    # Get current user from context
    user_id = tool_context.state.get("app:current_user_id")
    
    # Use prefixed key
    prefixed_key = f"user:{user_id}:{key}"
    tool_context.state[prefixed_key] = value
    
    # Also persist to Supabase
    supabase_client = tool_context.state["app:supabase_client"]
    # ... persistence logic
```

### Memory Patterns

```python
# Long-term memory via Supabase
class PersistentMemory:
    def __init__(self, supabase_client):
        self.client = supabase_client
        
    async def remember(self, user_id: str, key: str, value: Any, confidence: float = 1.0):
        await self.client.table("user_preferences").upsert({
            "user_id": user_id,
            "key": key,
            "value": json.dumps(value),
            "confidence": confidence,
            "updated_at": datetime.utcnow().isoformat()
        }).execute()
```

## 🔧 Tool Design

### Tool Anatomy

```python
def analyze_bid_competitiveness(
    tool_context: ToolContext,
    project_id: str,
    bid_amount: float,
    contractor_id: str
) -> dict:
    """
    Analyzes how competitive a bid is compared to market rates.
    
    This tool queries historical bid data and current market conditions
    to determine if a bid is competitive, helping contractors price
    appropriately and homeowners evaluate offers.
    
    Args:
        tool_context (ToolContext): ADK context for state/session access
        project_id (str): UUID of the project being bid on
        bid_amount (float): Proposed bid amount in USD
        contractor_id (str): UUID of the contractor submitting bid
        
    Returns:
        dict: Competitiveness analysis with structure:
            {
                "status": "success",
                "competitiveness_score": 0.75,  # 0-1 scale
                "market_position": "slightly_above_average",
                "suggested_range": {"min": 8000, "max": 12000},
                "similar_projects_avg": 10500,
                "confidence": 0.82
            }
            
    Raises:
        ValueError: If project_id or contractor_id not found
        RuntimeError: If market data unavailable
    """
    try:
        # Implementation
        pass
    except Exception as e:
        logger.error(f"Bid analysis failed: {e}")
        return {
            "status": "error",
            "message": str(e)
        }
```

### Tool Categories

1. **Data Tools** - Interact with Supabase
2. **Analysis Tools** - Process information
3. **Communication Tools** - Send messages/notifications
4. **Integration Tools** - Connect with external services

## 🚨 Error Handling

### Graceful Degradation

```python
class ResilientAgent(LlmAgent):
    async def _call_with_fallback(self, tool_name: str, **kwargs):
        try:
            return await self.call_tool(tool_name, **kwargs)
        except Exception as e:
            logger.warning(f"Tool {tool_name} failed: {e}")
            
            # Try fallback
            if tool_name == "get_market_rates":
                return await self.call_tool("get_cached_rates", **kwargs)
            
            # Return safe default
            return {
                "status": "degraded",
                "message": "Using default values",
                "data": self._get_defaults(tool_name)
            }
```

### Retry Patterns

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
async def call_openai_vision(image_path: str):
    # API call with automatic retry
    pass
```

## 🧪 Testing Strategies

### Agent Testing

```python
# tests/unit/test_homeowner_agent.py
import pytest
from unittest.mock import Mock, AsyncMock
from instabids.agents.homeowner.agent import agent

@pytest.mark.asyncio
async def test_agent_slot_filling():
    # Mock tool context
    mock_context = Mock()
    mock_context.state = {"user:123:budget": 10000}
    
    # Test slot filling logic
    response = await agent.process_message(
        "I need help with my roof repair",
        context=mock_context
    )
    
    assert "budget" in response.slots_filled
    assert response.next_question is not None
```

### Integration Testing

```python
# tests/integration/test_project_flow.py
@pytest.mark.integration
async def test_complete_project_creation():
    # Start with empty state
    async with test_database() as db:
        # Create project via API
        response = await client.post("/projects", json={
            "description": "Fix leaking roof",
            "images": ["roof_damage.jpg"]
        })
        
        # Verify all components
        assert response.status_code == 201
        project_id = response.json()["project_id"]
        
        # Check database
        project = await db.get_project(project_id)
        assert project.bid_card_id is not None
        
        # Check messages
        messages = await db.get_messages(project_id)
        assert len(messages) > 0
```

## ⚡ Performance Optimization

### Caching Strategies

```python
from functools import lru_cache
from typing import Optional

class CachedMarketData:
    def __init__(self):
        self._cache_ttl = 3600  # 1 hour
        
    @lru_cache(maxsize=1000)
    def get_category_rates(self, category: str, zip_code: str) -> dict:
        # Expensive market data query
        return self._fetch_from_source(category, zip_code)
```

### Batch Operations

```python
async def invite_contractors_batch(project_id: str, contractor_ids: List[str]):
    """Send invitations in batches to avoid rate limits."""
    BATCH_SIZE = 50
    
    for i in range(0, len(contractor_ids), BATCH_SIZE):
        batch = contractor_ids[i:i + BATCH_SIZE]
        await asyncio.gather(*[
            send_invitation(project_id, contractor_id)
            for contractor_id in batch
        ])
        
        # Rate limit pause
        await asyncio.sleep(1)
```

### Token Optimization

```python
class EfficientAgent(LlmAgent):
    def __init__(self):
        super().__init__(
            name="efficient",
            model="gemini-2.0-flash-exp",  # Use fast model
            instructions=self._get_concise_instructions(),
            temperature=0.7,  # Less randomness = fewer tokens
            max_tokens=1000   # Cap response length
        )
        
    def _get_concise_instructions(self):
        return """Be concise. Use bullet points. No verbose explanations."""
```

## 📚 Additional Resources

- [ADK Python Documentation](https://github.com/google/adk-python)
- [Vertex AI Agent Engine Guide](https://cloud.google.com/vertex-ai/docs/agents)
- [A2A Protocol Specification](https://github.com/google/agent-to-agent-protocol)

Remember: These patterns are battle-tested from real ADK 1.0.0 deployments. Follow them to avoid common issues and build robust agents!
