From Zero to GenLayer: Complete Contract Guide
==============================================

Audience and Outcome
--------------------
- New builders: follow the numbered steps to get a Python Intelligent Contract (PIC) live on GenLayer with Studio and make your first calls.
- Experienced builders: jump to the code, determinism checklist, and hardening tips.

What You Will Ship
------------------
- A minimal dispute-resolution PIC in Python.
- Deployed to GenLayer via Studio.
- Invocable from Studio UI or from a script / `genlayer-js`.

Prerequisites
-------------
- Python 3.10+ and Node.js 18+ installed.
- A GenLayer Studio account and API key.
- Git (optional) and a code editor.

GenLayer Concepts (brief)
-------------------------
- Optimistic Democracy Consensus (ODC): State updates are accepted optimistically; the network can challenge incorrect transitions. Make your transitions simple to verify and guarded with assertions.
- Equivalence Principle (EP): Local execution must match chain execution. Keep PIC code deterministic—no randomness, no time-based logic, no external I/O.
- Python Intelligent Contracts: Deterministic Python state machines executed by GenLayer nodes. Methods are exposed as actions callable via Studio or SDKs.

Fast Path (TL;DR)
-----------------
1) Create folder and venv.
2) Drop the PIC code into `contracts/dispute.py`.
3) Run the local check snippet.
4) Deploy via Studio (upload the file, get `contractId`).
5) Call methods in Studio or via `genlayer-js` using your `contractId` and API key.

Project Layout
--------------
```
genlayer-dispute/
  contracts/
    dispute.py        # the PIC
  .env (optional)     # holds GENLAYER_API_KEY and CONTRACT_ID for scripts
```

Step 1: Set Up Environment
--------------------------
```bash
mkdir genlayer-dispute && cd genlayer-dispute
python -m venv .venv
.\.venv\Scripts\activate  # Windows PowerShell
pip install genlayer        # or: pip install -r requirements.txt if using the boilerplate
mkdir contracts
```

Step 2: Write the PIC (contracts/dispute.py)
--------------------------------------------
```python
from genlayer.contract import Contract, state, action


class DisputeResolver(Contract):
    dispute_id: int = state(default=0)
    claimant: str = state(default="")
    respondent: str = state(default="")
    resolver: str = state(default="")
    status: str = state(default="idle")  # idle|open|resolved
    verdict: str = state(default="")

    @action
    def open(self, claimant: str, respondent: str, resolver: str):
        assert self.status in ("idle", "resolved"), "dispute active"
        self.dispute_id += 1
        self.claimant = claimant
        self.respondent = respondent
        self.resolver = resolver
        self.status = "open"
        self.verdict = ""

    @action
    def decide(self, caller: str, verdict: str):
        assert self.status == "open", "no open dispute"
        assert caller == self.resolver, "only resolver can decide"
        self.verdict = verdict
        self.status = "resolved"


contract = DisputeResolver()
```

Why this works
--------------
- Assertions keep ODC-friendly, easy-to-verify transitions.
- Deterministic updates (pure state changes, no external calls) satisfy EP.
- A single active dispute keeps the example simple; you can extend to multiple IDs later.

Step 3: Local Determinism Check
--------------------------------
Quick sanity run:
```bash
python - <<"PY"
from contracts.dispute import contract

contract.open("alice", "bob", "judge")
contract.decide("judge", "claimant_wins")
print(contract.status, contract.verdict)
PY
```
Expected output: `resolved claimant_wins`.

If you see errors
-----------------
- "dispute active": you called `open` twice without resolving.
- "only resolver can decide": `caller` did not match the stored `resolver`.
- Any randomness/time usage: remove it; the PIC must stay deterministic.

Step 4: Deploy with GenLayer Studio
-----------------------------------
1) Open Studio → New Contract.
2) Upload `contracts/dispute.py`.
3) Choose network (use testnet to start) and deploy.
4) Copy the returned `contractId` (and endpoint if shown). Save it in `.env` as `CONTRACT_ID` for scripts.
5) In Studio’s Call tab:
   - Run `open` with `claimant`, `respondent`, `resolver`.
   - Run `decide` with `caller` set to the resolver and your `verdict`.
6) Check the state viewer to confirm fields updated as expected.

Step 5: Call the Contract (Studio UI or code)
---------------------------------------------
Studio UI
- Use the Call tab to invoke `open` and `decide` directly.

Using genlayer-js (minimal example)
```bash
npm install genlayer-js
```
```js
import { GenLayerClient } from "genlayer-js";

const client = new GenLayerClient({ apiKey: process.env.GENLAYER_API_KEY });
const contractId = process.env.CONTRACT_ID;

await client.callContract({
  contractId,
  method: "open",
  args: { claimant: "alice", respondent: "bob", resolver: "judge" },
});

await client.callContract({
  contractId,
  method: "decide",
  args: { caller: "judge", verdict: "claimant_wins" },
});
```
Set `GENLAYER_API_KEY` and `CONTRACT_ID` in your environment or `.env.local` before running.

Using Python (requests-style)
-----------------------------
If you prefer Python calls, adapt to the Studio REST endpoint (if available) or a simple SDK wrapper. Example shape:
```python
import os
import requests

API_KEY = os.getenv("GENLAYER_API_KEY")
CONTRACT_ID = os.getenv("CONTRACT_ID")
BASE_URL = "https://api.genlayer.dev"  # adjust if Studio shows a different endpoint

def call(method, args):
    res = requests.post(
        f"{BASE_URL}/contracts/{CONTRACT_ID}/call",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={"method": method, "args": args},
        timeout=10,
    )
    res.raise_for_status()
    return res.json()

print(call("open", {"claimant": "alice", "respondent": "bob", "resolver": "judge"}))
print(call("decide", {"caller": "judge", "verdict": "claimant_wins"}))
```

Determinism and Safety Checklist (do this before deploy)
-------------------------------------------------------
- No randomness, clocks, or external HTTP/file I/O inside actions.
- Assertions guard every state transition (caller checks, status checks).
- State is explicit and typed; defaults are set.
- Re-entrancy not applicable (PIC execution is single-threaded), but still avoid mutable globals outside state.
- Tests or dry runs reproduce the same outputs when inputs are the same.

Common Errors and Fixes
-----------------------
- Mismatched resolver: ensure `caller` equals the stored `resolver` when calling `decide`.
- "dispute active": resolve or add a multi-dispute registry (see extensions) before opening again.
- 401/403 from API: check API key and that it has access to the chosen network.
- 404/400 on call: confirm `contractId` and method name spelling; ensure deployment finished.

Production-Hardening Ideas (for pros)
-------------------------------------
- Multi-dispute registry: store disputes in a mapping keyed by id; return ids on `open`.
- Challenge window: add a `pending` status with a timestamp-like counter passed as an argument (avoid reading time; accept it as input and validate monotonicity via assertions).
- Bonds and slashing: require stake arguments and maintain balances; slash on invalid decisions.
- Events/logs: emit structured events for frontends to subscribe to (check GenLayer event support in your SDK version).
- Access control: support multiple resolvers or quorum-based decisions.

Upgrade and Versioning Tips
---------------------------
- Treat each PIC deployment as immutable; re-deploy updated code and rotate `CONTRACT_ID` in clients.
- Keep a `VERSION` field in state; fail calls that target an outdated version if you later migrate.
- Maintain migration scripts to copy state when moving to a new contract (export via Studio if supported).

Testing Recommendations
-----------------------
- Unit tests: simulate calls in-process (similar to the local check) for all branches.
- Negative tests: wrong caller to `decide`, double `open`, empty resolver.
- Property: id increments monotonically; status always in {idle, open, resolved}.
- Regression: freeze sample inputs/outputs to ensure EP holds across changes.

Troubleshooting
---------------
- Deployment fails: re-run with a clean venv and ensure `genlayer` package matches Studio expectations.
- Calls hang or error: verify network selection (testnet vs mainnet), API key scope, and endpoint URL.
- State not updating: ensure assertions are not firing; check Studio logs if available.

FAQ
---
- Why Python? Rapid iteration and clear determinism. The PIC is the single source of truth for both local and on-chain execution.
- Where does ODC matter here? Assertions make each optimistic step easy to verify or challenge.
- What breaks EP? Anything nondeterministic: time, random, external fetches, reading local files.

Next Extensions (pick one to learn more)
----------------------------------------
- Add a challenge phase with a finalization action after a delay provided as an argument.
- Add stakes for parties and resolver; release or slash based on verdict.
- Convert to a multi-dispute registry keyed by `dispute_id`.
- Add event emission for frontends to stream updates.From Zero to GenLayer: Contract-Only Guide
=========================================

Goal
----
Deploy a minimal Python Intelligent Contract (PIC) on GenLayer that models a simple dispute-resolution flow. This guide focuses only on the contract: author it, validate determinism, deploy via GenLayer Studio, and call it.

Prerequisites
-------------
- Python 3.10+
- Git (optional) and a text editor
- GenLayer Studio account + API key

Quick Concepts
--------------
- Optimistic Democracy Consensus (ODC): State updates are accepted optimistically; bad transitions can be challenged. Keep state transitions explicit and easy to verify.
- Equivalence Principle (EP): Local execution and on-chain execution must be identical. Write deterministic code: no randomness, no time-based logic, no external I/O.

Directory Setup
---------------
```bash
mkdir genlayer-dispute && cd genlayer-dispute
python -m venv .venv
.\.venv\Scripts\activate
pip install genlayer  # if the boilerplate uses a requirements.txt, install that instead
mkdir contracts
```

Write the Contract (contracts/dispute.py)
-----------------------------------------
```python
from genlayer.contract import Contract, state, action


class DisputeResolver(Contract):
		dispute_id: int = state(default=0)
		claimant: str = state(default="")
		respondent: str = state(default="")
		resolver: str = state(default="")
		status: str = state(default="idle")  # idle|open|resolved
		verdict: str = state(default="")

		@action
		def open(self, claimant: str, respondent: str, resolver: str):
				assert self.status in ("idle", "resolved"), "dispute active"
				self.dispute_id += 1
				self.claimant = claimant
				self.respondent = respondent
				self.resolver = resolver
				self.status = "open"
				self.verdict = ""

		@action
		def decide(self, caller: str, verdict: str):
				assert self.status == "open", "no open dispute"
				assert caller == self.resolver, "only resolver can decide"
				self.verdict = verdict
				self.status = "resolved"


contract = DisputeResolver()
```

Local Determinism Check
-----------------------
If you have the boilerplate runner, use it; otherwise a simple script works:
```bash
python - <<"PY"
from contracts.dispute import contract

contract.open("alice", "bob", "judge")
contract.decide("judge", "claimant_wins")
print(contract.status, contract.verdict)
PY
```
Expected: `resolved claimant_wins`. Any nondeterminism (time, random, external calls) must be removed before deploying.

Deploy via GenLayer Studio
--------------------------
1) Open Studio → New Contract.
2) Upload `contracts/dispute.py`.
3) Choose network (testnet recommended) → Deploy.
4) Record the returned `contractId` (and endpoint, if shown).
5) Use Studio’s Call tab to invoke:
	 - Method `open` with `claimant`, `respondent`, `resolver`.
	 - Method `decide` with `caller` = resolver, `verdict` = your decision.
6) Verify state in Studio’s state viewer (confirms EP: local vs chain match).

Call the Contract (two options)
-------------------------------
**Studio UI:** Use the Call tab as above.

**genlayer-js (minimal):**
```bash
npm install genlayer-js
```
```js
import { GenLayerClient } from "genlayer-js";

const client = new GenLayerClient({ apiKey: process.env.GENLAYER_API_KEY });
const contractId = process.env.CONTRACT_ID;

await client.callContract({
	contractId,
	method: "open",
	args: { claimant: "alice", respondent: "bob", resolver: "judge" },
});
