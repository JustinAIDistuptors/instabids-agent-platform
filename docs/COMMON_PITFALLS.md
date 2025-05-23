# 🛑️ Common Pitfalls & Instant Fixes — Google ADK 1.0.0 + InstaBids

> **CRITICAL**: Review this BEFORE writing any code. Updated May 22, 2025.

## 🔴 Top 30 ADK 1.0.0 Pitfalls

| # | Pitfall (≈ probability) | Symptom / Bad Code | Fix | Refs |
|---|------------------------|-------------------|-----|------|
| **1** | **Wrong import** (90%) | `import google.generativeai as genai` | `from google import genai` | [Quickstart](https://github.com/google/adk-python) |
| **2** | **Ghost Git wheel** (90%) | `ModuleNotFoundError: google.adk.cli.main` | `poetry cache clear pypi --all && rm -rf .venv && poetry install` | GH #73 |
| **3** | **Missing [vertexai]** (85%) | Live model call errors | `pip install "google-adk[vertexai]~=1.0.0"` | [Install Docs](https://pypi.org/project/google-adk/) |
| **4** | **Protobuf mismatch** (80%) | `ImportError: _pb2` | Pin `protobuf==5.29.4` | #618 |
| **5** | **Wrong agent export** (80%) | `.adk scan skips module` | Export as `agent = MyAgent()` NOT `root_agent` | ADK samples |
| **6** | **ToolContext missing** (75%) | `TypeError: missing tool_context` | First param: `tool_context: ToolContext` | Tool guide |
| **7** | **Live model ID gap** (70%) | `Model gemini-2.0-flash-live-001 not found` | Use `"gemini-2.0-flash-exp"` | #866 |
| **8** | **Weak docstrings** (70%) | LLM ignores tool | ≥5-line docstring with Args/Returns | Best practices |
| **9** | **Bad import paths** (65%) | `from google.adk.runtime import Event` | Use `google.adk.events` | Module guide |
| **10** | **No state prefix** (60%) | Memory collision | Use `user:` `app:` `temp:` prefixes | State docs |
| **11** | **Positional args** (55%) | Params shift after refactor | Always `name=..., model=...` | Style guide |
| **12** | **Port 8000 busy** (55%) | `[Errno 10048]` Windows | Use `ports.pick_free()` helper | #296 |
| **13** | **Bad tool return** (50%) | `ToolCallValidationError` | Return `{"status": "success", ...}` | Tool schema |
| **14** | **Global adk shadows** (45%) | Wrong Python used | `pipx uninstall adk` | PATH issue |
| **15** | **No Proactor loop** (40%) | `RuntimeError: Event loop closed` | Add Windows fix to `__init__.py` | #296 |
| **16** | **Wrong `__init__.py`** (40%) | Components scan fails | `from .agent import agent` | Export pattern |
| **17** | **Model misuse** (35%) | Slow/expensive | Flash for speed, Pro for reasoning | Model guide |
| **18** | **Env var typo** (35%) | `PermissionDenied` | Must be `GOOGLE_API_KEY` | #693 |
| **19** | **A2A/MCP mixup** (30%) | Wrong protocol | A2A = events, MCP = code interface | Docs |
| **20** | **Old streaming** (30%) | `.start_stream()` | Use `run_live()` | New API |
| **21** | **CallbackContext** (25%) | `AttributeError` | Match exact signature | #202 |
| **22** | **Artifact confusion** (20%) | Binary inline → OOM | Use artifact store | Storage docs |
| **23** | **Docker missing** (20%) | `ModuleNotFoundError` | Add `[vertexai]` to Dockerfile | Deploy guide |
| **24** | **Wrong CLI** (15%) | `adk run --script` | Use `adk test` or `adk api_server` | CLI ref |
| **25** | **sub_agents misuse** (15%) | Can't delegate | List OR components.json, not both | MAS guide |
| **26** | **Live quota hit** (15%) | `429 RESOURCE_EXHAUSTED` | Throttle tests, mock in CI | Quota docs |
| **27** | **Supabase RLS** (10%) | 403 on Storage | Add service_role policy | RLS guide |
| **28** | **Circular import** (10%) | After AI rewrites | Restart Python after edits | #61 |
| **29** | **grpcio mismatch** (5%) | `_Call` object error | Match versions exactly | Compat table |
| **30** | **Model cache stale** (5%) | Model still not found | `rm ~/.cache/adk/model_catalog.json` | Cache issue |

## 🎯 InstaBids-Specific Pitfalls

### 💾 Database Issues

**Pitfall**: Forgetting RLS policies
```sql
-- ❌ WRONG: Table without RLS
CREATE TABLE projects (...);

-- ✅ CORRECT: Enable RLS
CREATE TABLE projects (...);
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own projects"
  ON projects FOR SELECT
  USING (auth.uid() = owner_id);
```

**Pitfall**: Not handling Supabase service role
```python
# ❌ WRONG: Using anon key for admin operations
client = create_client(url, anon_key)

# ✅ CORRECT: Use service key when needed
if admin_operation:
    client = create_client(url, service_key)
```

### 🤖 Agent Communication

**Pitfall**: Direct agent calls
```python
# ❌ WRONG: Tight coupling
result = bid_card_agent.generate_card(data)

# ✅ CORRECT: Use events
await emit_event(BidCardRequestedEvent(project_id=project_id))
```

### 🔧 Tool Patterns

**Pitfall**: Synchronous Supabase calls in async context
```python
# ❌ WRONG: Blocks event loop
def get_project(tool_context, project_id):
    return supabase.table("projects").select("*").eq("id", project_id).execute()

# ✅ CORRECT: Use async client
async def get_project(tool_context, project_id):
    return await supabase.table("projects").select("*").eq("id", project_id).execute()
```

## 💚 One-Time Setup Scripts

### Reset Environment (`scripts/reset_env.ps1`)
```powershell
# Windows PowerShell
Write-Host "Resetting environment..." -ForegroundColor Green
poetry cache clear pypi --all
Remove-Item -Recurse -Force .venv -ErrorAction SilentlyContinue
poetry lock --no-update
poetry install --sync
Remove-Item "$env:USERPROFILE\.cache\adk\model_catalog.json" -ErrorAction SilentlyContinue
Write-Host "Environment reset complete!" -ForegroundColor Green
```

### Reset Environment (`scripts/reset_env.sh`)
```bash
#!/bin/bash
set -e
echo "Resetting environment..."
poetry cache clear pypi --all
rm -rf .venv
poetry lock --no-update
poetry install --sync
rm -f ~/.cache/adk/model_catalog.json
echo "Environment reset complete!"
```

### Port Helper (`src/instabids/utils/ports.py`)
```python
import socket
import random
import contextlib

def pick_free_port() -> int:
    """Return an unused TCP port (avoids Errno 10048 on Windows)."""
    for _ in range(20):
        port = random.randint(8100, 9000)
        with contextlib.closing(socket.socket()) as s:
            if s.connect_ex(("127.0.0.1", port)) != 0:
                return port
    raise RuntimeError("No free port found")
```

## 🧪 Quick Validation Tests

### Test 1: Imports
```python
# Run: python -c "exec(open('test_imports.py').read())"
try:
    from google import genai
    from google.adk.agents.llm_agent import CallbackContext
    from google.adk.events import Event
    from google.adk.types import Content, Part
    print("✅ All imports OK")
except ImportError as e:
    print(f"❌ Import failed: {e}")
```

### Test 2: Model Registry
```python
from google import genai
models = [m.name for m in genai.discovery.list_models() if "flash-exp" in m.name]
if models:
    print(f"✅ Live model available: {models[0]}")
else:
    print("❌ No flash-exp model found - install from GitHub main")
```

### Test 3: Doctor Route
```python
# Add to src/instabids/api/routes/health.py
from fastapi import APIRouter
from google import genai
import pkg_resources

router = APIRouter(prefix="/healthz", tags=["health"])

@router.get("/doctor")
def doctor_check():
    """Comprehensive health check for debugging."""
    try:
        models = [m.name for m in genai.discovery.list_models()]
        packages = {
            p.key: p.version 
            for p in pkg_resources.working_set 
            if "google" in p.key or "instabids" in p.key
        }
        
        return {
            "status": "healthy",
            "models_available": len(models),
            "packages": packages,
            "checks": {
                "imports": "pass",
                "database": "pending",
                "agents": "pending"
            }
        }
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

## 📝 Checklist Before Committing

- [ ] Run `./scripts/reset_env.sh` to ensure clean dependencies
- [ ] Verify imports: `from google import genai` (not `google.generativeai`)
- [ ] Check agent export: `agent = MyAgent()` at module level
- [ ] Validate tool signatures: `tool_context: ToolContext` as first param
- [ ] Use state prefixes: `user:`, `app:`, `temp:`
- [ ] Test locally: `poetry run adk web` and `pytest`
- [ ] Review this file for any matching patterns

## 🆘 Emergency Fixes

### "Module not found" after update
```bash
poetry cache clear pypi --all
rm -rf .venv
rm poetry.lock
poetry install
```

### "Model not found" with latest ADK
```bash
# Option 1: Use experimental model
model = "gemini-2.0-flash-exp"

# Option 2: Install pre-release
pip install --pre google-adk
```

### Windows "Event loop closed"
```python
# Add to src/instabids/__init__.py
import asyncio
import sys

if sys.platform.startswith('win'):
    asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
```

### Port already in use
```python
# Use the port helper
from instabids.utils.ports import pick_free_port

app = FastAPI()
port = pick_free_port()
uvicorn.run(app, host="0.0.0.0", port=port)
```

---

**Remember**: This document is maintained by both human and AI developers. If you discover a new pitfall, add it here with a clear example and fix!
