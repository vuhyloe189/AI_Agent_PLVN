# PLVN Orchestrator and Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Xây dựng một PLVN Orchestrator duy nhất điều phối bốn skill, model-effort routing, retrieval/evidence trên snapshot bất biến và API/CLI tiếng Việt mà không có runtime subagent hoặc external-action capability.

**Architecture:** Orchestrator là state machine tuần tự, chỉ giữ state và gọi typed ports. Skills cung cấp workflow ngữ nghĩa; script/data plane của Plan 1 cung cấp freshness, snapshot, search và diff. Model chỉ nhận context đã thu gọn và evidence từ snapshot; mọi tool call đi qua allowlist/policy gateway.

**Tech Stack:** Python 3.12, Pydantic v2, PyYAML, OpenAI Responses adapter, FastAPI, Typer, pytest/pytest-asyncio.

**Spec:** `docs/superpowers/specs/2026-10-15-plvn-orchestrator-design.md`

## Global Constraints

- Prerequisite: Plan 1 completion gate đã pass và các interface `SourceRegistry`, `FreshnessReceipt`, `CorpusSnapshot`, `SearchHit`, `LegalRepository`, `search_nodes()` tồn tại đúng tên.
- Chỉ một Orchestrator; không import/call API tạo agent/subagent, không consensus voting.
- Business logic dùng profile `D0_DETERMINISTIC`–`M4_CRITICAL`; model ID chỉ đến từ environment/config.
- `as_of` mặc định theo `Asia/Ho_Chi_Minh`; facts confirmed/assumed/disputed/missing không được trộn.
- Answer plane chỉ đọc một `snapshot_id` đã pin; snapshot mới giữa workflow không đổi state hiện tại.
- Model không nhận generic shell/browser/network handle. Mọi external access đi qua Plan-1 adapter; chỉ `GET/HEAD` allowlisted.
- Không gửi email/tin nhắn, upload/login/submit, giao dịch, đại diện/mạo danh, scam/lừa đảo hoặc hỗ trợ hành vi trái pháp luật.
- Tối đa 12 tool invocation, một answer revision, một profile upgrade và hai retry/network operation.
- Retrieval tối đa 12 candidate/issue, 8 evidence/issue, 32 evidence/request; exact locator luôn được giữ.
- Test không gọi model hoặc network thật; dùng fake/stub ports.
- Không `git push`, deploy hay kết nối trực tiếp với cá nhân/cơ quan.

## Review Focus

- Snapshot mới publish giữa workflow không được thay snapshot đã pin — Task 6 có `test_run_keeps_pinned_snapshot_after_store_changes`.
- Receipt hết hạn và official source lỗi không được render “mới nhất” — Task 8 có `test_expired_unverified_freshness_abstains`.
- Exact locator không bị top-k loại bỏ — Task 6 có `test_exact_locator_survives_candidate_limit`.
- Nguồn/model yêu cầu upload, login, gửi tin hoặc HTTP write phải bị chặn — Task 3 có `test_external_actions_are_always_denied`.
- Hết budget phải kết thúc có reason code, không loop — Task 8 có `test_budget_exhaustion_is_terminal`.

## File Map

- `src/plvn/contracts/`: immutable schemas shared by Orchestrator, skills and quality gates.
- `src/plvn/orchestrator/`: policy gateway, state transition table and coordinating service only.
- `src/plvn/routing/`, `config/routing.yaml`: logical profile/risk/effort decisions; no hard-coded model IDs.
- `src/plvn/providers/`: structured-output provider adapter without model-facing tools.
- `skills/`, `src/plvn/skills_runtime/`: four concise skills, policy manifests, loader and runner.
- `src/plvn/retrieval/`: snapshot pinning, evidence limits and packaging over Plan-1 ports.
- `src/plvn/api/`, `src/plvn/cli.py`, `scripts/ask_plvn.py`: local Vietnamese user interfaces.
- `tests/`: fake-port unit/integration coverage; no live model/network.

---

### Task 1: Orchestration contracts và typed ports

**Files:**

- Create: `src/plvn/contracts/__init__.py`
- Create: `src/plvn/contracts/common.py`
- Create: `src/plvn/contracts/orchestration.py`
- Create: `src/plvn/contracts/evidence.py`
- Create: `src/plvn/contracts/skills.py`
- Create: `src/plvn/orchestrator/__init__.py`
- Create: `src/plvn/orchestrator/ports.py`
- Test: `tests/contracts/test_orchestration_models.py`

**Interfaces:**

- Consumes: Plan-1 `FreshnessReceipt`, `CorpusSnapshot`, `SearchHit`, `NodeDiff`.
- Produces: enums and Pydantic models used unchanged by Tasks 2–9 and Plan 3; protocols `Clock`, `FreshnessPort`, `SnapshotPort`, `CorpusSearchPort`, `ModelPort`, `VerificationPort`, `AuditPort`.

- [ ] **Step 1: Viết contract tests**

```python
def test_question_request_uses_injected_clock_for_default_as_of(clock): ...
def test_run_state_cannot_replace_pinned_snapshot(): ...

def test_non_complete_terminal_answer_requires_reason_code():
    with pytest.raises(ValidationError):
        AnswerEnvelope(answer_type="abstain", reason_codes=())

def test_fact_categories_are_distinct(): ...
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/contracts/test_orchestration_models.py -v`  
Expected: FAIL vì contracts chưa tồn tại.

- [ ] **Step 3: Implement exact contracts**

```python
class RunPhase(StrEnum):
    INTAKE = "intake"
    RISK_AND_SCOPE = "risk_and_scope"
    FRESHNESS = "freshness"
    SNAPSHOT_PINNED = "snapshot_pinned"
    RETRIEVAL = "retrieval"
    REASONING = "reasoning"
    VERIFY = "verify"
    RENDER = "render"
    COMPLETE = "complete"
    ABSTAIN = "abstain"
    ESCALATE = "escalate"
    REFUSE = "refuse"

class ModelProfile(StrEnum):
    D0_DETERMINISTIC = "d0_deterministic"
    M1_FAST = "m1_fast"
    M2_BALANCED = "m2_balanced"
    M3_REASONING = "m3_reasoning"
    M4_CRITICAL = "m4_critical"

class QuestionRequest(BaseModel):
    question: str
    as_of: date | None = None
    jurisdiction: Literal["VN"] = "VN"
    force_refresh: bool = False

class RunState(BaseModel, frozen=True):
    request_id: UUID
    phase: RunPhase
    request: QuestionRequest
    snapshot_id: str | None = None
    tool_calls: int = 0
    revisions: int = 0
    profile_upgrades: int = 0
    # confirmed_facts, assumed_facts, disputed_facts, missing_facts, issues,
    # receipts, evidence_bundle, draft, verification and reason_codes.
```

`AnswerEnvelope` phải có `rendered_text`, `as_of`, `snapshot_id`, facts, issues, claims, uncertainties, freshness, confidence band, reason codes và next step. Define `Evidence`, `EvidenceBundle`, `SkillDefinition`, `SkillContext`, `SkillOutput` với `schema_version="1.0"`. `VerificationPort.verify(...)` là async và trả canonical `VerificationDecision`; mọi implementation ở Plan 3 phải giữ đúng signature này.

Canonical fields:

```python
class AnswerClaim(BaseModel):
    claim_id: str
    text: str
    criticality: Literal["critical", "material", "context"]
    evidence_ids: tuple[str, ...]
    support: Literal["supported", "partial", "unsupported", "conflicted"]
    temporal_status: Literal["valid", "superseded", "future", "unknown"]

class Evidence(BaseModel):
    evidence_id: str
    snapshot_id: str
    document_id: int
    version_id: int
    node_id: int
    locator: str | None
    text: str
    supporting_span: str
    span_hash: str
    source_url: str
    source_tier: Literal[0, 1, 2, 3]
    fetched_at: datetime
    effective_from: date | None
    effective_to_exclusive: date | None

class SearchRequest(BaseModel):
    issue_id: str
    query: str
    as_of: date
    snapshot_id: str
    exact_locator: str | None = None
    limit: int = Field(default=12, ge=1, le=12)

class ToolCallSummary(BaseModel):
    operation: str
    origin: InputOrigin
    parameters: Mapping[str, object]

class VerificationRequest(BaseModel):
    state: RunState
    draft: SkillOutput
    evidence: EvidenceBundle
    snapshot: CorpusSnapshot
    planned_actions: tuple[ToolCallSummary, ...]

class VerificationDecision(BaseModel):
    passed: bool
    terminal_phase: RunPhase | None
    reason_codes: tuple[ReasonCode, ...]
    finding_ids: tuple[str, ...]
```

`ReasonCode` phải khai báo ngay tại Task 1 tối thiểu các mã đang dùng trong ba plan: `INVALID_REQUEST`, `SOURCE_URL_NOT_ALLOWED`, `REDIRECT_NOT_ALLOWED`, `EXTERNAL_ACTION_NOT_ALLOWED`, `UNLAWFUL_ASSISTANCE_REQUEST`, `FRESHNESS_UNVERIFIED`, `SNAPSHOT_MISMATCH`, `INSUFFICIENT_EVIDENCE`, `CITATION_MISSING`, `EVIDENCE_NOT_FOUND`, `LOCATOR_MISMATCH`, `EVIDENCE_HASH_MISMATCH`, `OFFICIAL_SOURCE_REQUIRED`, `GROUNDING_UNVERIFIED`, `TEMPORAL_INVALID`, `TRANSITION_UNRESOLVED`, `PII_LEAK_DETECTED`, `PROMPT_INJECTION_DETECTED`, `TOOL_BUDGET_EXCEEDED`, `REVISION_EXHAUSTED`, `PROFILE_UPGRADE_EXHAUSTED`, `PROVIDER_ERROR`, `VERIFIER_INTERNAL_ERROR`, `HUMAN_REVIEW_REQUIRED`, `SYSTEM_ERROR`.

Port signatures:

```python
class FreshnessPort(Protocol):
    async def ensure_fresh(
        self, request: QuestionRequest, issues: tuple[Issue, ...]
    ) -> tuple[FreshnessReceipt, ...]: ...

class SnapshotPort(Protocol):
    async def latest_published(self) -> CorpusSnapshot: ...

class CorpusSearchPort(Protocol):
    async def search(self, request: SearchRequest) -> tuple[SearchHit, ...]: ...

class ModelPort(Protocol):
    async def generate(
        self, *, instructions: str, payload: Mapping[str, object],
        profile: ModelProfile, output_type: type[T]
    ) -> T: ...

class VerificationPort(Protocol):
    async def verify(
        self, request: VerificationRequest
    ) -> VerificationDecision: ...

class AuditPort(Protocol):
    def append(self, event: Mapping[str, object]) -> None: ...
```

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/contracts/test_orchestration_models.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/contracts src/plvn/orchestrator tests/contracts pyproject.toml
git commit -m "feat: define PLVN orchestration contracts"
```

### Task 2: Bốn skill packages và validated skill registry

**Files:**

- Create: `skills/plvn-tra-cuu/SKILL.md`
- Create: `skills/plvn-tra-cuu/policy.yaml`
- Create: `skills/plvn-tra-cuu/references/output-contract.md`
- Create: `skills/plvn-tra-cuu/agents/openai.yaml`
- Create: matching files for `skills/plvn-so-sanh/`
- Create: matching files for `skills/plvn-phan-tich/`
- Create: matching files for `skills/plvn-bao-tri/`
- Create: `src/plvn/skills_runtime/__init__.py`
- Create: `src/plvn/skills_runtime/loader.py`
- Test: `tests/skills_runtime/test_loader.py`
- Test fixture: `tests/fixtures/skills/invalid_external_write/`

**Interfaces:**

- Consumes: `SkillDefinition`/`SkillName` from Task 1.
- Produces: `load_skill_definition(skill_dir: Path) -> SkillDefinition`, `load_skill_registry(root: Path) -> Mapping[SkillName, SkillDefinition]`.

- [ ] **Step 1: Viết loader/validation tests**

```python
def test_registry_loads_exactly_four_plvn_skills(skill_root):
    registry = load_skill_registry(skill_root)
    assert set(registry) == {
        SkillName.TRA_CUU, SkillName.SO_SANH,
        SkillName.PHAN_TICH, SkillName.BAO_TRI,
    }

def test_skill_declaring_external_write_or_subagent_is_rejected(): ...
def test_maintenance_skill_requires_trusted_operator_origin(): ...
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/skills_runtime/test_loader.py -v`  
Expected: FAIL vì skills và loader chưa tồn tại.

- [ ] **Step 3: Tạo skill bằng skill-authoring workflow**

Khi thực thi step này, đọc và dùng `skill-creator` cùng `writing-skills`; scaffold từng skill bằng script được skill hướng dẫn, đọc đầy đủ chuẩn `agents/openai.yaml`, rồi validate bằng `quick_validate.py`. `SKILL.md` chỉ dùng frontmatter chuẩn `name`/`description`; policy runtime riêng nằm trong `policy.yaml` với `version`, `trusted_origins`, `allowed_operations`, `forbidden_capabilities`, `output_schema`.

Skill requirements:

- `plvn-tra-cuu`: current/historical lookup, bắt buộc `as_of` và source hierarchy.
- `plvn-so-sanh`: lineage trước diff, statuses `unchanged/amended/new/repealed/moved`.
- `plvn-phan-tich`: facts → issue → rule → exception/transition → application → counterargument → conditional conclusion.
- `plvn-bao-tri`: deterministic operator workflow; không kích hoạt từ model/web evidence.

- [ ] **Step 4: Implement loader và policy validation**

Reject skill nếu tên thư mục/frontmatter khác nhau, thiếu output contract, có generic browser/shell, external write, communication, payment, impersonation hoặc subagent capability.

- [ ] **Step 5: Chạy tests và skill validators**

Run: `python -m pytest tests/skills_runtime/test_loader.py -v`  
Expected: PASS.  
Run validator command do `skill-creator` cung cấp cho cả bốn thư mục.  
Expected: tất cả valid.

- [ ] **Step 6: Commit**

```powershell
git add skills src/plvn/skills_runtime/loader.py tests/skills_runtime tests/fixtures/skills
git commit -m "feat: add validated Vietnamese-law skills"
```

### Task 3: External-action deny gate và typed tool gateway

**Files:**

- Create: `src/plvn/orchestrator/policy.py`
- Create: `src/plvn/orchestrator/tool_gateway.py`
- Create: `config/policy.yaml`
- Test: `tests/orchestrator/test_policy.py`
- Test: `tests/orchestrator/test_tool_gateway.py`

**Interfaces:**

- Consumes: Task-1 ports and Plan-1 data-plane functions.
- Produces: `ToolOperation`, `ToolCall`, `ToolResult`, `PolicyGate.authorize(...)`, `ToolGateway.invoke(...)`.

- [ ] **Step 1: Viết policy/gateway tests**

```python
@pytest.mark.parametrize("operation", [
    "http_post", "upload", "login", "send_email", "send_message",
    "submit_form", "payment", "impersonate", "external_delete",
])
def test_external_actions_are_always_denied(policy, operation):
    decision = policy.authorize_requested_action(operation, origin=InputOrigin.USER)
    assert decision.allowed is False
    assert decision.reason_code == ReasonCode.EXTERNAL_ACTION_NOT_ALLOWED

def test_thirteenth_tool_invocation_is_rejected(gateway, state_at_12): ...
```

Thêm test maintenance từ `UNTRUSTED_EVIDENCE` bị block, model text không thể tạo enum operation, local append-only operation hợp lệ và redirect policy vẫn do Plan 1 xử lý.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/orchestrator/test_policy.py tests/orchestrator/test_tool_gateway.py -v`  
Expected: FAIL vì modules chưa tồn tại.

- [ ] **Step 3: Implement closed operation enum và gateway**

```python
class ToolOperation(StrEnum):
    CHECK_FRESHNESS = "check_freshness"
    PIN_SNAPSHOT = "pin_snapshot"
    SEARCH_CORPUS = "search_corpus"
    GET_EVIDENCE = "get_evidence"
    DIFF_VERSIONS = "diff_versions"
    LOCAL_MAINTENANCE = "local_maintenance"

class PolicyGate:
    def authorize(self, call: ToolCall, *, origin: InputOrigin) -> PolicyDecision: ...
    def authorize_requested_action(
        self, action: str, *, origin: InputOrigin
    ) -> PolicyDecision: ...

class ToolGateway:
    async def invoke(self, call: ToolCall, state: RunState) -> ToolResult: ...
```

Không có generic URL, shell command hoặc arbitrary function call trong model-facing schema. `LOCAL_MAINTENANCE` chỉ cho trusted operator và không có destructive operation.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/orchestrator/test_policy.py tests/orchestrator/test_tool_gateway.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/orchestrator/policy.py src/plvn/orchestrator/tool_gateway.py config/policy.yaml tests/orchestrator
git commit -m "feat: enforce bounded read-only tool policy"
```

### Task 4: Intent, risk và reasoning-effort routing

**Files:**

- Create: `src/plvn/routing/__init__.py`
- Create: `src/plvn/routing/config.py`
- Create: `src/plvn/routing/router.py`
- Create: `config/routing.yaml`
- Test: `tests/routing/test_router.py`

**Interfaces:**

- Consumes: `IntakeResult`, `RiskLevel`, `SkillName`, `ModelProfile`.
- Produces: `RoutingConfig`, `ProfileDecision`, `classify_skill(...)`, `select_profile(...)`, `upgrade_profile(...)`.

- [ ] **Step 1: Viết routing tests**

```python
def test_simple_lookup_routes_to_tra_cuu_m1(): ...
def test_version_comparison_routes_to_so_sanh_at_least_m2(): ...
def test_transition_or_exception_routes_to_phan_tich_m3(): ...
def test_critical_deadline_or_large_sanction_routes_m4(): ...
def test_max_effort_rejected_for_noncritical_unapproved_route(): ...
def test_second_profile_upgrade_is_exhausted(): ...
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/routing/test_router.py -v`  
Expected: FAIL vì routing modules chưa tồn tại.

- [ ] **Step 3: Implement logical routing config**

```python
def classify_skill(intake: IntakeResult) -> SkillName: ...
def select_profile(context: RoutingContext, config: RoutingConfig) -> ProfileDecision: ...
def upgrade_profile(
    current: ProfileDecision, state: RunState, trigger: UpgradeTrigger
) -> ProfileDecision: ...
```

`routing.yaml` chỉ chứa profile, allowed effort, timeout/token budget, risk triggers và `model_ref_env`; không chứa model ID trong code. `D0` bắt buộc cho crawl/parse/index/diff/citation-locator checks.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/routing/test_router.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/routing config/routing.yaml tests/routing
git commit -m "feat: route PLVN skills and reasoning effort"
```

### Task 5: Structured model port và OpenAI Responses adapter

**Files:**

- Create: `src/plvn/providers/__init__.py`
- Create: `src/plvn/providers/openai_responses.py`
- Create: `tests/fixtures/fake_ports.py`
- Test: `tests/providers/test_openai_responses.py`
- Modify: `pyproject.toml`
- Modify: `.env.example`

**Interfaces:**

- Consumes: `ModelPort`, `ModelProfile`, Pydantic output types, `routing.yaml`.
- Produces: `OpenAIResponsesModel.generate(...)`; `FakeModelPort` for all tests.

- [ ] **Step 1: Viết adapter tests với stub client**

```python
async def test_adapter_uses_configured_model_ref_and_effort(stub_client): ...
async def test_adapter_requests_structured_output_schema(stub_client): ...

async def test_adapter_never_exposes_web_shell_or_external_write_tools(stub_client):
    await adapter.generate(...)
    request = stub_client.last_request
    assert request.get("tools", []) == []
```

Thêm test API key không log, timeout/budget map đúng profile và error được chuyển thành stable provider reason code.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/providers/test_openai_responses.py -v`  
Expected: FAIL vì provider adapter chưa tồn tại.

- [ ] **Step 3: Thêm dependencies và implement adapter**

Thêm `pydantic>=2.10,<3`, `pydantic-settings>=2.7,<3`, `openai>=2,<3`, `tzdata>=2025.1`, `pytest-asyncio>=0.25,<2`. Adapter gọi Responses API qua injected client, không truyền tools; model output phải parse vào `output_type` hoặc fail closed. `.env.example` chỉ liệt kê `OPENAI_API_KEY`, `PLVN_MODEL_FAST`, `PLVN_MODEL_BALANCED`, `PLVN_MODEL_REASONING`, `PLVN_MODEL_CRITICAL` không có secret/value thật.

```python
class OpenAIResponsesModel(ModelPort):
    async def generate(
        self, *, instructions: str, payload: Mapping[str, object],
        profile: ModelProfile, output_type: type[T]
    ) -> T: ...
```

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/providers/test_openai_responses.py -v`  
Expected: PASS, không network call.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/providers tests/providers tests/fixtures/fake_ports.py pyproject.toml .env.example
git commit -m "feat: add structured model provider adapter"
```

### Task 6: Snapshot pinning, retrieval và evidence packaging

**Files:**

- Create: `src/plvn/retrieval/__init__.py`
- Create: `src/plvn/retrieval/ports.py`
- Create: `src/plvn/retrieval/service.py`
- Create: `scripts/get_evidence.py`
- Test: `tests/retrieval/test_service.py`
- Create: `tests/fixtures/evidence.py`

**Interfaces:**

- Consumes: Plan-1 snapshot/search APIs and Task-1 `Issue`, `Evidence`, `EvidenceBundle`.
- Produces: `RetrievalService.pin_snapshot(...)`, `RetrievalService.retrieve(...) -> EvidenceBundle`.

- [ ] **Step 1: Viết retrieval tests**

```python
async def test_run_keeps_pinned_snapshot_after_store_changes(service, state): ...
async def test_expired_hit_is_excluded_for_as_of_date(service): ...

async def test_exact_locator_survives_candidate_limit(service):
    bundle = await service.retrieve(issues, as_of=AS_OF, snapshot=SNAPSHOT)
    assert "exact-locator-id" in bundle.evidence_ids
```

Thêm test limits 12/8/32, snapshot mismatch reason, parent/child context và insufficient coverage flag thay vì silent truncation.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/retrieval/test_service.py -v`  
Expected: FAIL vì retrieval service chưa tồn tại.

- [ ] **Step 3: Implement snapshot-bound retrieval**

```python
class RetrievalService:
    async def pin_snapshot(self, state: RunState) -> RunState: ...

    async def retrieve(
        self, issues: tuple[Issue, ...], *, as_of: date,
        snapshot: CorpusSnapshot,
    ) -> EvidenceBundle: ...
```

Mọi query/evidence request bắt buộc chứa snapshot ID. Rerank exact locator → official tier → temporal validity → BM25; preserve supporting span/hash/source URL/fetched time.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/retrieval/test_service.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/retrieval scripts/get_evidence.py tests/retrieval tests/fixtures/evidence.py
git commit -m "feat: retrieve evidence from pinned legal snapshots"
```

### Task 7: Single-Orchestrator skill runner

**Files:**

- Create: `src/plvn/skills_runtime/runner.py`
- Test: `tests/skills_runtime/test_runner.py`
- Modify: `tests/fixtures/fake_ports.py`

**Interfaces:**

- Consumes: skill definitions, `SkillContext`, `ModelPort`, `EvidenceBundle`, routing decision.
- Produces: `SkillRunner.run(...) -> SkillOutput`.

- [ ] **Step 1: Viết skill-runner tests**

```python
async def test_output_cannot_reference_evidence_outside_bundle(): ...
async def test_assumption_cannot_become_confirmed_fact(): ...
async def test_untrusted_evidence_remains_delimited_data(): ...
async def test_maintenance_uses_deterministic_operation_without_model_when_possible(): ...
async def test_runner_makes_one_sequential_model_call_and_no_subagent_call(): ...
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/skills_runtime/test_runner.py -v`  
Expected: FAIL vì runner chưa tồn tại.

- [ ] **Step 3: Implement bounded runner**

```python
class SkillRunner:
    async def run(
        self, definition: SkillDefinition, context: SkillContext,
        profile: ModelProfile,
    ) -> SkillOutput: ...
```

Context chỉ chứa redacted facts, `as_of`, snapshot ID, evidence/diff được phép. Wrap evidence trong `UNTRUSTED_EVIDENCE`; validate evidence IDs, fact lineage và output schema sau model call.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/skills_runtime/test_runner.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/skills_runtime/runner.py tests/skills_runtime/test_runner.py tests/fixtures/fake_ports.py
git commit -m "feat: run PLVN skills through one bounded model port"
```

### Task 8: Bounded state machine và PLVN Orchestrator

**Files:**

- Create: `src/plvn/orchestrator/machine.py`
- Create: `src/plvn/orchestrator/service.py`
- Test: `tests/orchestrator/test_machine.py`
- Test: `tests/integration/test_orchestrator_workflow.py`

**Interfaces:**

- Consumes: all ports/services from Tasks 1–7; Plan-3 verifier may initially be injected as strict test double.
- Produces: `transition(state: RunState, event: RunEvent) -> RunState`, `PLVNOrchestrator.answer(request: QuestionRequest) -> AnswerEnvelope`.

- [ ] **Step 1: Viết machine/workflow tests**

```python
def test_transition_whitelist_rejects_retrieval_before_snapshot(): ...

async def test_expired_unverified_freshness_abstains(orchestrator):
    result = await orchestrator.answer(CURRENT_LAW_QUESTION)
    assert result.answer_type == "abstain"
    assert ReasonCode.FRESHNESS_UNVERIFIED in result.reason_codes
    assert "mới nhất" not in result.rendered_text.lower()

async def test_budget_exhaustion_is_terminal(orchestrator): ...
```

Thêm happy path, one-revision-only, one-upgrade-only, external action → refuse, scam/impersonation/forgery request → refuse with lawful alternative, provider error → reason-coded abstain, no subagent trace.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/orchestrator/test_machine.py tests/integration/test_orchestrator_workflow.py -v`  
Expected: FAIL vì machine/service chưa tồn tại.

- [ ] **Step 3: Implement transition table và orchestrator**

Allowed flow:

```text
INTAKE → RISK_AND_SCOPE → FRESHNESS → SNAPSHOT_PINNED
→ RETRIEVAL → REASONING → VERIFY → RENDER
→ COMPLETE | ABSTAIN | ESCALATE | REFUSE
```

`PLVNOrchestrator` chỉ điều phối; không parse, search, sửa evidence hoặc render thêm legal claim. Persist checkpoint sau mỗi phase qua AuditPort; enforce 12/1/1 budgets.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/orchestrator/test_machine.py tests/integration/test_orchestrator_workflow.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/orchestrator/machine.py src/plvn/orchestrator/service.py tests/orchestrator tests/integration/test_orchestrator_workflow.py
git commit -m "feat: implement bounded PLVN orchestration workflow"
```

### Task 9: Vietnamese API/CLI và offline end-to-end

**Files:**

- Create: `src/plvn/api/__init__.py`
- Create: `src/plvn/api/dependencies.py`
- Create: `src/plvn/api/app.py`
- Create: `src/plvn/cli.py`
- Create: `scripts/ask_plvn.py`
- Test: `tests/integration/test_hoi_dap_api.py`
- Test: `tests/integration/test_cli.py`
- Modify: `pyproject.toml`

**Interfaces:**

- Consumes: `PLVNOrchestrator` and typed contracts.
- Produces: `create_app(orchestrator) -> FastAPI`, `build_cli(orchestrator_factory) -> typer.Typer`, endpoint `POST /v1/hoi-dap`, command `plvn hoi`.

- [ ] **Step 1: Viết API/CLI tests**

```python
def test_api_returns_vietnamese_answer_as_of_snapshot_claims_and_freshness(client): ...
def test_api_rejects_external_action_with_stable_reason(client): ...
def test_cli_help_and_validation_are_vietnamese(runner): ...
def test_cli_renders_abstain_escalate_refuse_without_external_side_effect(runner): ...
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/integration/test_hoi_dap_api.py tests/integration/test_cli.py -v`  
Expected: FAIL vì API/CLI chưa tồn tại.

- [ ] **Step 3: Implement API/CLI composition root**

Thêm `fastapi>=0.115,<1`, `uvicorn>=0.34,<1`, `typer>=0.15,<1`. External JSON dùng alias `cau_hoi`, `ngay_ap_dung`; internal fields giữ tên chuẩn. `POST /v1/hoi-dap` là inbound call vào local service, không phải outbound write tới bên thứ ba. API chỉ phục vụ người dùng hiện tại, không có endpoint gửi/nộp/giao dịch. CLI command:

```powershell
plvn hoi --cau-hoi "..." --ngay-ap-dung 2026-10-15
```

- [ ] **Step 4: Chạy full verification**

Run: `python -m pytest tests -v`  
Expected: PASS, không network/model thật.  
Run: `python -m ruff check src tests scripts`  
Expected: PASS.  
Run: `python -m plvn.cli --help`  
Expected: exit `0`, help tiếng Việt.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/api src/plvn/cli.py scripts/ask_plvn.py tests/integration pyproject.toml
git commit -m "feat: expose local Vietnamese PLVN API and CLI"
```

## Plan 2 Completion Gate

- Một request fixture đi trọn workflow bằng fake model/ports, snapshot được pin và output có `as_of`, citations, freshness, reason code khi cần.
- Bốn skill pass official skill validation và runtime policy validation.
- Không skill/model nào có generic browser/shell, external-write, communication, payment, impersonation hoặc subagent capability.
- Tool, revision và profile budgets đều fail closed.
- API/CLI hoàn toàn tiếng Việt ở bề mặt người dùng và không có endpoint hành động thay người dùng.
- Citation/temporal/privacy/safety release gates đầy đủ vẫn thuộc Plan 3; Plan 2 không được tuyên bố production-ready.
