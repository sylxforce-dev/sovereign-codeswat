# Benchmark Telemetry: Run A — User Session Tracker

```text
Pipeline Status: PASS (0 retries)
Total Duration:  ~30s
Task:            FastAPI In-Memory User Session Tracker
Model:           Qwen2.5.1-Coder-7B-Instruct (Q6_K_L GGUF)
Hardware:        NVIDIA RTX 5060 Ti 8GB
```

---

## 1. Task Specification & Constraints

```text
Create a FastAPI application for an in-memory user session tracker.

ENDPOINTS:
POST '/sessions/create'
Accepts a JSON body with 'user_id' (string) and 'ip_address' (string).
Generates a unique session_id using uuid.uuid4().
Stores the session with active status.
Returns {'session_id': ..., 'user_id': ..., 'is_active': true}.

POST '/sessions/{session_id}/terminate'
Terminates the specified session by setting its status to inactive.
Returns {'session_id': ..., 'is_active': false} if found.
Returns {'session_id': ..., 'is_active': false, 'error': 'not_found'} if the session does not exist, without raising exceptions.

GET '/sessions/user/{user_id}'
Returns a list of all active session objects for that specific user: [{'session_id': ..., 'ip_address': ...}, ...].
If no active sessions exist, returns an empty list: [].
This endpoint must not modify state.

CRITICAL REQUIREMENTS:
- Store all mutable session states inside a singleton class named SessionRegistry.
- SessionRegistry must be instantiated exactly once.
- All endpoints must access state exclusively via SessionRegistry injected through FastAPI Depends().
- Do not instantiate SessionRegistry inside endpoint functions.
- Do not use global mutable containers (such as global dicts or lists) outside the registry class.
- Do not use an external database or persistent disk storage.
- Endpoints must validate incoming JSON using dedicated Pydantic model subclasses (e.g. SessionCreateRequest inheriting from BaseModel) with explicit fields, not raw BaseModel.
- Do not mutate incoming request objects directly.
- Do not use HTTPException, eval(), exec(), dict(), globals(), or locals().
- Do not add authentication, background workers, middleware, or unrelated libraries.

ALLOWED IMPORTS:
from fastapi import FastAPI, Depends
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
import uuid
```

---

## 2. Pipeline Execution Telemetry

| Pipeline Stage | Status | Duration | Resource Metrics |
|---|---|---|---|
| **Planner Phase** | ✅ Contract Audited | 17.93s | CPU: 10.8% \| RAM: 65.4% \| GPU: 94% \| VRAM: 7.26 GB |
| **Group: models** | ✅ Green Light (AST + Semantic) | 6.69s | CPU: 8.3% \| RAM: 65.1% \| GPU: 96% \| VRAM: 7.27 GB |
| **Group: dependencies** | ✅ Green Light (AST + Semantic) | 1.94s | CPU: 10.1% \| RAM: 65.1% \| GPU: 98% \| VRAM: 7.27 GB |
| **Group: endpoints** | ✅ Green Light (AST + Semantic) | 3.74s | CPU: 8.3% \| RAM: 65.0% \| GPU: 97% \| VRAM: 7.26 GB |
| **Full Audit & Assembly** | 🟢 PASS (Integrated) | 0.00s | CPU: 7.5% \| RAM: 65.0% \| GPU: 97% \| VRAM: 7.26 GB |

<details>
<summary><b>View Raw Terminal Log</b></summary>

```text
🔥 Kütus GPU-sse... Laen mudeli: C:\Projektid\CodeSwat\models\Qwen2.5.1-Coder-7B-Instruct-Q6_K_L.gguf
✅ Mudel laetud ja lukus.
   🧹 [LLMLoader] Windows kernel RAM flush: Surnud kaal eemaldatud.

⏳ Planner analüüsib ja loob JSON lepingu (Katse 1/3)...
✅ Leping käes ja auditeeritud: Deterministlik Audit: Spetsiifilisi nõuete plokke ei tuvastatud. Leping läbis kontrolli.
   ⏱️  Planner: 17.93s
   📊 CPU: 10.8% | RAM: 65.4% | GPU: 94% | VRAM: 7.26 GB / 7.96 GB

-> Töötlen gruppi: models
   ✅ Green Light! (AST, Arhitektuuriline ja Semantiline audit edukas. Kood on puhas.)
   ⏱️  Grupp 'models': 6.69s
   📊 CPU: 8.3% | RAM: 65.1% | GPU: 96% | VRAM: 7.27 GB / 7.96 GB

-> Töötlen gruppi: dependencies
   ✅ Green Light! (AST, Arhitektuuriline ja Semantiline audit edukas. Kood on puhas.)
   ⏱️  Grupp 'dependencies': 1.94s
   📊 CPU: 10.1% | RAM: 65.1% | GPU: 98% | VRAM: 7.27 GB / 7.96 GB

-> Töötlen gruppi: endpoints
   ✅ Green Light! (AST, Arhitektuuriline ja Semantiline audit edukas. Kood on puhas.)
   ⏱️  Grupp 'endpoints': 3.74s
   📊 CPU: 8.3% | RAM: 65.0% | GPU: 97% | VRAM: 7.26 GB / 7.96 GB

🔬 Terviklik arhitektuurne audit ja kokkupanek...
🟢 PASS (Kõik sümbolid lahendatud ja impordid vastavad lepingule.)
   ⏱️  Full audit: 0.00s
   📊 CPU: 7.5% | RAM: 65.0% | GPU: 97% | VRAM: 7.26 GB / 7.96 GB
```
</details>

---

## 3. Final Assembled & Audited Output

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
import uuid

app = FastAPI()


# === GROUP: models ===
class SessionCreateRequest(BaseModel):
    user_id: str = Field(...)
    ip_address: str = Field(...)

class SessionRegistry:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.sessions = {}
        return cls._instance

    def create_session(self, user_id: str, ip_address: str) -> Dict[str, Any]:
        session_id = str(uuid.uuid4())
        self.sessions[session_id] = {
            "user_id": user_id,
            "ip_address": ip_address,
            "is_active": True
        }
        return {
            "session_id": session_id,
            "user_id": user_id,
            "is_active": True
        }

    def terminate_session(self, session_id: str) -> Dict[str, Any]:
        if session_id in self.sessions:
            session = self.sessions[session_id]
            session["is_active"] = False
            return {
                "session_id": session_id,
                "is_active": False
            }
        else:
            return {
                "session_id": session_id,
                "is_active": False,
                "error": "not_found"
            }

    def get_active_sessions_for_user(self, user_id: str) -> List[Dict[str, Any]]:
        return [
            session for session in self.sessions.values()
            if session["user_id"] == user_id and session["is_active"]
        ]
# === END GROUP: models ===

# === GROUP: dependencies ===
def get_session_registry() -> SessionRegistry:
    return SessionRegistry()
# === END GROUP: dependencies ===

# === GROUP: endpoints ===
@app.post("/sessions/create")
async def create_session(payload: SessionCreateRequest, session_registry: SessionRegistry = Depends(get_session_registry)):
    return session_registry.create_session(payload.user_id, payload.ip_address)

@app.post("/sessions/{session_id}/terminate")
async def terminate_session(session_id: str, session_registry: SessionRegistry = Depends(get_session_registry)):
    return session_registry.terminate_session(session_id)

@app.get("/sessions/user/{user_id}")
async def get_active_sessions_for_user(user_id: str, session_registry: SessionRegistry = Depends(get_session_registry)):
    return session_registry.get_active_sessions_for_user(user_id)
# === END GROUP: endpoints ===
```