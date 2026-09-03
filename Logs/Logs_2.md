# Benchmark Telemetry: Run B — Customer Loyalty Service

```text
Pipeline Status: PASS with Dead Code (Auditor Gap Benchmark)
Total Duration:  ~72s (Planner: ~60s)
Task:            FastAPI In-Memory Customer Loyalty Points Service
Model:           Qwen2.5.1-Coder-7B-Instruct (Q6_K_L GGUF)
Hardware:        NVIDIA RTX 5060 Ti 8GB
```

---

## 1. Task Specification & Constraints

```text
Create a FastAPI application for an in-memory customer loyalty points service.

ENDPOINTS:
POST '/accounts/{customer_id}/earn'
Accepts a points amount (integer).
Adds points to the customer's balance.
Returns {'balance': ...}.

POST '/accounts/{customer_id}/redeem'
Accepts a points amount (integer).
Attempts to deduct points from the customer's balance.
Returns {'success': true, 'balance': ...} if the customer had enough points.
Returns {'success': false, 'balance': ...} if insufficient points are available, without altering state incorrectly.

GET '/accounts/{customer_id}/balance'
Returns the customer's current balance: {'customer_id': ..., 'balance': ...}.
This endpoint must not modify state.

CRITICAL REQUIREMENTS:
- Store all mutable application state inside a singleton class named LoyaltyStore.
- LoyaltyStore must be instantiated exactly once.
- All endpoints must receive LoyaltyStore through FastAPI Depends().
- Do not instantiate LoyaltyStore inside endpoint functions.
- Do not use global mutable containers for application state.
- Do not use a database or external storage.
- Each customer_id must have an independent points balance.
- A new customer_id must start with a balance of zero.
- Balance must never drop below zero.
- GET endpoints must never modify application state.
- Do not mutate incoming Pydantic request objects.
- Do not use eval(), exec(), dict(), globals(), or locals().
- Do not use HTTPException().
- Do not add authentication, logging, persistence, background workers, caching, external services, or unrelated features.

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
| **Planner Phase** | ✅ Contract Audited | 60.73s | CPU: 10.6% \| RAM: 63.9% \| GPU: 94% \| VRAM: 7.46 GB |
| **Group: models** | ✅ Green Light (AST + Semantic) | 4.51s | CPU: 10.8% \| RAM: 63.7% \| GPU: 97% \| VRAM: 7.38 GB |
| **Group: dependencies** | ✅ Green Light (AST + Semantic) | 1.34s | CPU: 13.4% \| RAM: 63.8% \| GPU: 98% \| VRAM: 7.38 GB |
| **Group: endpoints** | ✅ Green Light (AST + Semantic) | 5.21s | CPU: 12.4% \| RAM: 63.6% \| GPU: 96% \| VRAM: 7.37 GB |
| **Full Audit & Assembly** | 🟢 PASS (Integrated) | 0.00s | CPU: 6.2% \| RAM: 63.6% \| GPU: 96% \| VRAM: 7.37 GB |

<details>
<summary><b>View Raw Terminal Log</b></summary>

```text
🔥 Kütus GPU-sse... Laen mudeli: C:\Projektid\CodeSwat\models\Qwen2.5.1-Coder-7B-Instruct-Q6_K_L.gguf
✅ Mudel laetud ja lukus.
   🧹 [LLMLoader] Windows kernel RAM flush: Surnud kaal eemaldatud.

⏳ Planner analüüsib ja loob JSON lepingu (Katse 1/3)...
✅ Leping käes ja auditeeritud: Deterministlik Audit: Spetsiifilisi nõuete plokke ei tuvastatud. Leping läbis kontrolli.
   ⏱️  Planner: 60.73s
   📊 CPU: 10.6% | RAM: 63.9% | GPU: 94% | VRAM: 7.46 GB / 7.96 GB

-> Töötlen gruppi: models
   ✅ Green Light! (AST, Arhitektuuriline ja Semantiline audit edukas. Kood on puhas.)
   ⏱️  Grupp 'models': 4.51s
   📊 CPU: 10.8% | RAM: 63.7% | GPU: 97% | VRAM: 7.38 GB / 7.96 GB

-> Töötlen gruppi: dependencies
   ✅ Green Light! (AST, Arhitektuuriline ja Semantiline audit edukas. Kood on puhas.)
   ⏱️  Grupp 'dependencies': 1.34s
   📊 CPU: 13.4% | RAM: 63.8% | GPU: 98% | VRAM: 7.38 GB / 7.96 GB

-> Töötlen gruppi: endpoints
   ✅ Green Light! (AST, Arhitektuuriline ja Semantiline audit edukas. Kood on puhas.)
   ⏱️  Grupp 'endpoints': 5.21s
   📊 CPU: 12.4% | RAM: 63.6% | GPU: 96% | VRAM: 7.37 GB / 7.96 GB

🔬 Terviklik arhitektuurne audit ja kokkupanek...
🟢 PASS (Kõik sümbolid lahendatud ja impordid vastavad lepingule.)
   ⏱️  Full audit: 0.00s
   📊 CPU: 6.2% | RAM: 63.6% | GPU: 96% | VRAM: 7.37 GB / 7.96 GB
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
class LoyaltyStore:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.balances = {}
        return cls._instance

    def get_balance(self, customer_id: str) -> int:
        return self.balances.get(customer_id, 0)

    def add_points(self, customer_id: str, points: int) -> None:
        # Note: Redundant defensive check (Pydantic Field(..., gt=0) already prevents negative inputs)
        if points < 0:
            raise ValueError("Points cannot be negative")
        if customer_id not in self.balances:
            self.balances[customer_id] = 0
        self.balances[customer_id] += points

    def redeem_points(self, customer_id: str, points: int) -> bool:
        # Note: Redundant defensive check (Pydantic Field(..., gt=0) already prevents negative inputs)
        if points < 0:
            raise ValueError("Points cannot be negative")
        if customer_id not in self.balances or self.balances[customer_id] < points:
            return False
        self.balances[customer_id] -= points
        return True
# === END GROUP: models ===

# === GROUP: dependencies ===
def get_loyalty_store() -> LoyaltyStore:
    return LoyaltyStore()
# === END GROUP: dependencies ===

# === GROUP: endpoints ===
class EarnRequest(BaseModel):
    points: int = Field(..., gt=0)

class RedeemRequest(BaseModel):
    points: int = Field(..., gt=0)

@app.post("/accounts/{customer_id}/earn")
async def earn_points(customer_id: str, payload: EarnRequest, loyalty_store: LoyaltyStore = Depends(get_loyalty_store)):
    loyalty_store.add_points(customer_id, payload.points)
    balance = loyalty_store.get_balance(customer_id)
    return {"balance": balance}

@app.post("/accounts/{customer_id}/redeem")
async def redeem_points(customer_id: str, payload: RedeemRequest, loyalty_store: LoyaltyStore = Depends(get_loyalty_store)):
    success = loyalty_store.redeem_points(customer_id, payload.points)
    balance = loyalty_store.get_balance(customer_id)
    return {"success": success, "balance": balance}

@app.get("/accounts/{customer_id}/balance")
async def get_balance(customer_id: str, loyalty_store: LoyaltyStore = Depends(get_loyalty_store)):
    balance = loyalty_store.get_balance(customer_id)
    return {"customer_id": customer_id, "balance": balance}
# === END GROUP: endpoints ===
```