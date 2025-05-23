# 📖 README for AI Coding Agents

> **YOUR MISSION**: Build, maintain, and evolve the InstaBids platform with 100% AI-generated code.

## 🎯 Quick Navigation

1. **Before You Start**: Read `COMMON_PITFALLS.md` to avoid known issues
2. **Find Your Role**: Check `.prompts/system/` for your agent type
3. **Get Task Instructions**: Use `.prompts/tasks/` for specific operations
4. **Follow Standards**: Apply `.prompts/conventions/` to all code

## 🏗️ Project Context

InstaBids connects homeowners with contractors through AI-mediated project scoping and bidding:

```
Homeowner → Describes Project → AI Analyzes → Bid Card Generated → Contractors Matched → Bids Collected
```

### Key Business Logic
1. **Project Creation**: Homeowners describe work needed (text + photos)
2. **AI Enhancement**: Vision analysis + conversation to clarify scope
3. **Bid Card**: Structured summary with category, timeline, budget
4. **Matching**: Find suitable contractors based on project type
5. **Bidding**: Contractors submit competitive bids
6. **Communication**: In-app messaging throughout process

## 🛠️ Development Workflow

### 1. Environment Setup
```bash
# ALWAYS run first to avoid dependency conflicts
./scripts/reset_env.sh  # or .ps1 on Windows

# Verify setup
python -c "from google import genai; print('✅ Import OK')"
```

### 2. Task Execution Pattern
```python
# 1. CoreOrchestratorAgent receives task
# 2. PromptSelectorAgent chooses appropriate prompts
# 3. Specialized agent executes with context
# 4. Code generation → Testing → Review → Commit
```

### 3. Agent Development Checklist
- [ ] Create agent class inheriting from `BaseAgent` or `LlmAgent`
- [ ] Export as `agent` variable (NOT `root_agent`)
- [ ] Add to `.adk/components.json`
- [ ] Create comprehensive tool docstrings (5+ lines)
- [ ] Write unit tests in `tests/unit/`
- [ ] Add integration tests in `tests/integration/`
- [ ] Update relevant documentation

## 📁 Key Directories

### `/src/instabids/agents/`
All ADK agents live here. Each agent has:
- `agent.py` - Main agent implementation
- `__init__.py` - Exports `agent` variable
- `tools.py` - Agent-specific tools
- `prompts/` - Agent-specific prompts

### `/src/instabids/tools/`
Reusable tools for all agents:
- `supabase_tools.py` - Database operations
- `vision_tools.py` - Image analysis
- `matching_tools.py` - Contractor matching

### `/src/instabids/data/`
Supabase repositories and database logic:
- `projects_repo.py` - Project CRUD operations
- `bid_cards_repo.py` - Bid card management
- `messages_repo.py` - Messaging system
- `preferences_repo.py` - User preference storage

## 🔧 Tool Development Guidelines

### Anatomy of a Perfect Tool
```python
def analyze_project_image(tool_context: ToolContext, image_path: str, project_type: str = "general") -> dict:
    """
    Analyzes a project image using vision AI to extract relevant details.
    
    This tool uses OpenAI's vision capabilities to understand home improvement
    project images and extract structured information for bid card generation.
    
    Args:
        tool_context (ToolContext): ADK tool context for state access
        image_path (str): Path to the image file to analyze
        project_type (str): Type of project (repair, renovation, installation)
        
    Returns:
        dict: Analysis results with structure:
            {
                "status": "success",
                "detected_issues": ["missing shingles", "water damage"],
                "estimated_severity": "moderate",
                "recommended_trades": ["roofing", "water damage restoration"],
                "confidence": 0.85
            }
    """
    # Implementation here
```

### State Management Rules
```python
# User-specific data
tool_context.state["user:budget_preference"] = 10000

# Application-wide data
tool_context.state["app:contractor_categories"] = ["roofing", "plumbing"]

# Temporary workflow data
tool_context.state["temp:current_analysis"] = {"status": "pending"}
```

## 🧪 Testing Requirements

### For Every Feature
1. **Unit Test**: Test individual functions/methods
2. **Integration Test**: Test agent with real dependencies
3. **E2E Test**: Test complete user flow (if applicable)

### Test Naming Convention
```
test_<function_name>_<scenario>_<expected_outcome>
```

Example:
```python
def test_analyze_project_image_valid_roof_returns_issues():
    # Test implementation
```

## 🚨 Critical Reminders

### Import Patterns (ADK 1.0.0)
```python
# ✅ CORRECT
from google import genai
from google.adk.agents import BaseAgent, LlmAgent
from google.adk.events import Event
from google.adk.types import Content, Part

# ❌ WRONG
import google.generativeai as genai
from google.adk.runtime import Event  # Wrong module
```

### Windows Development
```python
# Add to __init__.py if developing on Windows
import asyncio
import sys

if sys.platform.startswith('win'):
    asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
```

### Model Selection
```python
# Fast operations, live interactions
model = "gemini-2.0-flash-exp"

# Complex reasoning, deep analysis
model = "gemini-2.0-pro"
```

## 🔄 Git Workflow

### Branch Naming
- Feature: `feature/add-contractor-dashboard`
- Sprint: `sprint/10-payment-integration`
- Bugfix: `fix/bid-card-generation-error`

### Commit Messages
```
<type>(<scope>): <subject>

<body>

Types: feat, fix, docs, style, refactor, test, chore
```

## 📊 Monitoring Your Work

### Local Testing
```bash
# Run all tests
pytest

# Run specific test file
pytest tests/unit/test_homeowner_agent.py

# Run with coverage
pytest --cov=instabids
```

### ADK Development UI
```bash
poetry run adk web
# Open http://localhost:8000
# Test your agents interactively
```

## 🆘 Getting Help

1. **Check Documentation**: This folder has everything you need
2. **Review Examples**: Look at existing agent implementations
3. **Consult Pitfalls**: Most issues are documented
4. **Use PromptSelector**: Let it guide you to the right instructions

## 🎯 Success Metrics

Your code is successful when:
- ✅ All tests pass (`pytest`, `adk test`)
- ✅ No pitfalls from `COMMON_PITFALLS.md` are triggered
- ✅ Code follows all conventions in `.prompts/conventions/`
- ✅ Documentation is updated for new features
- ✅ CI pipeline is green

Remember: You're building a production system. Every line of code matters!
