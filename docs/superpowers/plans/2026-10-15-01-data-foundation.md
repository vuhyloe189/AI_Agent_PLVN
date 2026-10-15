# PLVN Data Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Xây dựng data plane local, append-only và deterministic để kiểm tra nguồn hợp lệ, nạp/phiên bản hóa/parse văn bản pháp luật, tạo relation/diff và xuất SQLite FTS5 snapshot bất biến.

**Architecture:** `sources` quyết định source policy trước mọi HTTP call; `freshness` chỉ phát hiện delta; `ingestion` lưu artifact theo content hash; parse, relation, diff và index chỉ chạy trên dữ liệu local. Snapshot chỉ được publish atomically sau khi version và index hoàn tất.

**Tech Stack:** Python 3.12, SQLite WAL/FTS5, httpx, PyYAML, Beautiful Soup/lxml, pypdf, pytest, Ruff.

**Spec:** `docs/superpowers/specs/2026-10-15-plvn-orchestrator-design.md`

## Global Constraints

- Python `>=3.12,<3.14`; timezone pháp lý mặc định `Asia/Ho_Chi_Minh`.
- External network chỉ cho `GET`/`HEAD` tới domain/path trong registry; tối đa hai retry cho thao tác idempotent.
- Không login, upload, submit, gửi tin, giao dịch, `POST/PUT/PATCH/DELETE`, vượt robots/ToS/paywall/CAPTCHA hoặc gọi API nội bộ không công bố.
- Tier 3 (`thuvienphapluat.vn`, `luatvietnam.vn`) mặc định chỉ dùng discovery/metadata và không làm căn cứ quyết định.
- Raw content luôn là `UNTRUSTED_EVIDENCE`; raw artifact, DB, corpus, cache, index, log, secret và hồ sơ người dùng không được commit.
- Version append-only theo `source_id + canonical_url + legal_content_hash`; snapshot đã publish là immutable.
- CLI trả JSON ra stdout, log ra stderr; exit code `0/2/3/4/5` theo đặc tả.
- Test deterministic không gọi mạng thật hoặc model API.
- Không `git push`, deploy hoặc tạo external communication trong bất kỳ task nào.

## Review Focus

- Redirect từ URL hợp lệ sang domain ngoài allowlist phải bị chặn trước hop thứ hai — Task 2 có `test_fetch_blocks_cross_domain_redirect`.
- Method ghi hoặc path bị cấm phải fail trước transport — Task 2 có `test_policy_blocks_external_write_before_transport`.
- HTTP `304` không được tạo artifact, version hoặc snapshot mới — Task 4 có `test_not_modified_stops_before_ingest`.
- Parser phải giữ nguyên ngày, số tiền và locator tiếng Việt — Task 6 có `test_parser_preserves_money_date_and_hierarchy`.
- Boilerplate đổi nhưng nội dung pháp lý không đổi không được tạo version mới — Task 5 có `test_boilerplate_only_change_does_not_create_version`.

## File Map

- `config/sources.yaml`, `config/freshness.yaml`: source tiers, access/crawl policy and freshness SLA.
- `src/plvn/sources/`, `src/plvn/http/`, `src/plvn/freshness/`: registry, robots/ToS boundary, read-only transport and delta detection.
- `src/plvn/legal_store/`, `src/plvn/ingestion/`: SQLite schema, append-only repositories, artifact/provenance storage.
- `src/plvn/parsing/`, `src/plvn/relations/`, `src/plvn/diffing/`: deterministic extraction, legal hierarchy, lineage and structured diff.
- `src/plvn/indexing/`: exact/FTS index, reconciliation and immutable snapshot publishing.
- `scripts/`: thin JSON CLIs; no business logic.
- `tests/`, `tests/fixtures/`: offline deterministic tests and synthetic legal documents.

---

### Task 1: Repository, package contracts và source registry

**Files:**

- Create: `pyproject.toml`
- Create: `.gitignore`
- Create: `.env.example`
- Create: `src/plvn/__init__.py`
- Create: `src/plvn/sources/__init__.py`
- Create: `src/plvn/sources/registry.py`
- Create: `src/plvn/legal_store/models.py`
- Create: `config/sources.yaml`
- Create: `config/freshness.yaml`
- Test: `tests/sources/test_registry.py`

**Interfaces:**

- Consumes: đặc tả đã duyệt; không phụ thuộc code trước đó.
- Produces: `SourceDefinition`, `SourceRegistry`, `load_registry(path: Path) -> SourceRegistry`, `resolve_source_for_url(registry: SourceRegistry, url: str) -> SourceDefinition | None`; các dataclass `DocumentIdentity`, `DocumentVersion`, `LegalNode`, `LegalRelation`, `CorpusSnapshot`, `SearchHit` trong `legal_store/models.py`.

`DocumentVersion`, `LegalNode` và `SearchHit` phải dùng `effective_from: date | None` cùng `effective_to_exclusive: date | None`. `CorpusSnapshot` phải có `snapshot_id: str`, `created_at: datetime`, `version_ids: tuple[int, ...]`, `index_version: str`.

- [ ] **Step 1: Khởi tạo repository và test scaffold**

Run: `git init -b main` rồi tạo `pyproject.toml` với dependency runtime `PyYAML>=6.0.2,<7`, `httpx>=0.28,<1`, `beautifulsoup4>=4.12,<5`, `lxml>=5.3,<7`, `pypdf>=5,<7`; dependency dev `pytest>=8.3,<10`, `pytest-cov>=6,<8`, `ruff>=0.9,<1`, `respx>=0.22,<1`, `reportlab>=4.2,<5`. Cấu hình package layout `src/`, pytest và Ruff trong cùng file. `.gitignore` phải chặn `data/`, `*.db`, `*.sqlite*`, `.env*` trừ `.env.example`, `logs/`, cache/index, coverage và virtualenv.

- [ ] **Step 2: Viết test fail cho registry**

```python
def test_registry_rejects_unknown_tier(tmp_path: Path): ...

def test_resolve_source_requires_exact_domain_and_allowed_path(registry):
    assert resolve_source_for_url(registry, "https://evil-vbpl.vn/") is None
    assert resolve_source_for_url(registry, "https://vbpl.vn/api/private") is None
    assert registry.sources["vbpl"].tier == 1
```

Seed config phải có Tier 0 `congbao`, Tier 1 `vbpl`, Tier 2 cho cổng chính thức phù hợp và Tier 3 cho hai nguồn người dùng nêu. Tier 3 đặt `access_mode: discovery_only`, `authoritative_citation: false`.

- [ ] **Step 3: Chạy test và xác nhận RED**

Run: `python -m pytest tests/sources/test_registry.py -v`  
Expected: FAIL vì `plvn.sources.registry` chưa tồn tại.

- [ ] **Step 4: Implement registry và contracts tối thiểu**

Implement:

```python
@dataclass(frozen=True)
class SourceDefinition:
    source_id: str
    domains: tuple[str, ...]
    tier: Literal[0, 1, 2, 3]
    document_types: tuple[str, ...]
    authority_scope: str
    allowed_paths: tuple[str, ...]
    access_mode: Literal["public_read", "discovery_only", "manual_only", "disabled"]
    robots_policy: str
    terms_url: str | None
    licensing_notes: str
    allowed_use: tuple[str, ...]
    attribution_required: bool
    rate_limit_per_minute: int
    timeout_seconds: int
    parser_profile: str

@dataclass(frozen=True)
class SourceRegistry:
    sources: Mapping[str, SourceDefinition]
```

Reject duplicate `source_id`, wildcard domain, unknown tier, empty allowed path và config cho phép external write.

- [ ] **Step 5: Chạy test và lint**

Run: `python -m pytest tests/sources/test_registry.py -v`  
Expected: PASS.  
Run: `python -m ruff check src tests`  
Expected: PASS.

- [ ] **Step 6: Commit**

```powershell
git add pyproject.toml .gitignore .env.example src config tests docs
git commit -m "chore: scaffold PLVN data foundation"
```

### Task 2: External-read-only policy và HTTP adapter

**Files:**

- Create: `src/plvn/sources/policy.py`
- Create: `src/plvn/sources/robots.py`
- Create: `src/plvn/http/__init__.py`
- Create: `src/plvn/http/adapter.py`
- Test: `tests/sources/test_external_policy.py`
- Test: `tests/sources/test_robots.py`
- Test: `tests/http/test_adapter.py`

**Interfaces:**

- Consumes: `SourceRegistry`, `SourceDefinition` từ Task 1.
- Produces: `HttpRequest`, `HttpResponse`, `PolicyBlockedError`, `authorize_external_read(...) -> SourceDefinition`, `RobotsPolicyService.can_fetch(...)`, `ReadOnlyHttpClient.fetch(request: HttpRequest) -> HttpResponse`.

- [ ] **Step 1: Viết test fail cho policy và transport**

```python
def test_policy_blocks_external_write_before_transport(client, transport):
    with pytest.raises(PolicyBlockedError, match="EXTERNAL_ACTION_NOT_ALLOWED"):
        client.fetch(HttpRequest(method="POST", url="https://vbpl.vn/van-ban"))
    assert transport.calls == []

def test_fetch_blocks_cross_domain_redirect(client, transport):
    transport.redirect("https://vbpl.vn/a", "https://example.org/steal")
    with pytest.raises(PolicyBlockedError, match="REDIRECT_NOT_ALLOWED"):
        client.fetch(HttpRequest(method="GET", url="https://vbpl.vn/a"))

def test_unknown_or_disallowed_robots_policy_fails_closed(robots_service): ...
```

Thêm test `HEAD/GET` hợp lệ, `/api/` bị chặn, response quá 25 MiB bị `RESPONSE_TOO_LARGE`, retry không quá hai lần và conditional header được giữ.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/sources/test_external_policy.py tests/sources/test_robots.py tests/http/test_adapter.py -v`  
Expected: FAIL vì policy/adapter chưa tồn tại.

- [ ] **Step 3: Implement policy và HTTP adapter**

```python
@dataclass(frozen=True)
class HttpRequest:
    method: str
    url: str
    headers: Mapping[str, str] = field(default_factory=dict)

@dataclass(frozen=True)
class HttpResponse:
    status_code: int
    url: str
    headers: Mapping[str, str]
    body: bytes

def authorize_external_read(
    request: HttpRequest, registry: SourceRegistry
) -> SourceDefinition: ...
```

Validate method, scheme `https`, exact hostname, allowed path, robots policy và từng redirect hop trước khi gửi tiếp. `RobotsPolicyService` dùng public `/robots.txt`, cache theo source và fail closed khi policy chưa xác minh; không suy quyền caching/redistribution/training từ quyền đọc. Không giữ cookie/auth; User-Agent định danh; timeout/rate limit theo source; retry chỉ cho timeout/429/5xx của GET/HEAD.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/sources/test_external_policy.py tests/sources/test_robots.py tests/http/test_adapter.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/sources/policy.py src/plvn/sources/robots.py src/plvn/http tests/sources tests/http
git commit -m "feat: enforce external read-only source access"
```

### Task 3: SQLite schema, migrations và append-only repository

**Files:**

- Create: `src/plvn/legal_store/__init__.py`
- Create: `src/plvn/legal_store/database.py`
- Create: `src/plvn/legal_store/repositories.py`
- Create: `src/plvn/legal_store/migrations/001_initial.sql`
- Test: `tests/legal_store/test_migrations.py`
- Test: `tests/legal_store/test_repositories.py`

**Interfaces:**

- Consumes: legal-store dataclass contracts từ Task 1.
- Produces: `connect(path: Path) -> sqlite3.Connection`, `migrate(connection) -> None`, `LegalRepository`, `append_document_version(...) -> tuple[int, bool]`, `publish_snapshot(...) -> str` (stub persistence; atomic index publish hoàn thiện ở Task 8).

- [ ] **Step 1: Viết migration/repository tests**

```python
def test_migrate_enables_foreign_keys_wal_and_fts5(connection): ...

def test_append_version_is_idempotent(repository, version_input):
    first_id, first_created = repository.append_document_version(**version_input)
    second_id, second_created = repository.append_document_version(**version_input)
    assert (second_id, second_created) == (first_id, False)

def test_prior_version_cannot_be_updated(repository): ...
```

Schema: `schema_migrations`, `sources`, `legal_documents`, `artifacts`, `source_observations`, `document_versions`, `legal_nodes`, `legal_relations`, `freshness_receipts`, `corpus_snapshots`, `snapshot_versions`, `fts_legal_nodes`. `document_versions` và `legal_nodes` dùng `effective_from`/`effective_to_exclusive` theo interval half-open.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/legal_store -v`  
Expected: FAIL vì database/repository chưa tồn tại.

- [ ] **Step 3: Implement migration và append-only repository**

```python
def append_document_version(
    self, *, document_id: int, source_id: str, canonical_url: str,
    content_hash: str, legal_content_hash: str, artifact_hash: str,
    fetched_at: datetime, parser_version: str,
) -> tuple[int, bool]: ...
```

Migration chạy trong transaction, bật `foreign_keys=ON`, `journal_mode=WAL`; DB trigger chặn `UPDATE/DELETE` trên observation/version/node/snapshot tables. `source_observations` idempotent theo `source_id + canonical_url + content_hash`; legal version dedupe theo `document_id + legal_content_hash`. Duplicate key trả existing ID và `created=False`.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/legal_store -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/legal_store tests/legal_store
git commit -m "feat: add append-only SQLite legal store"
```

### Task 4: Freshness receipt và targeted update checks

**Files:**

- Create: `src/plvn/freshness/__init__.py`
- Create: `src/plvn/freshness/service.py`
- Create: `scripts/check_updates.py`
- Test: `tests/freshness/test_service.py`
- Test: `tests/scripts/test_check_updates.py`

**Interfaces:**

- Consumes: read-only HTTP adapter và repository.
- Produces: `FreshnessReceipt`, `receipt_is_fresh(...)`, `build_conditional_headers(...)`, `FreshnessService.check_resource(...) -> FreshnessReceipt`; JSON CLI `check_updates.py`.

- [ ] **Step 1: Viết freshness tests**

```python
def test_not_modified_produces_no_update_action(service):
    receipt = service.check_resource(resource_with_etag)
    assert receipt.changed is False
    assert receipt.status == "not_modified"
    assert service.plan_update(receipt).action == "none"

def test_recent_high_risk_receipt_expires_after_one_hour(): ...
def test_normal_receipt_expires_after_six_hours(): ...
def test_tier3_receipt_expires_after_twenty_four_hours(): ...
```

Thêm assertion feed poll 2 giờ, clock timezone-aware và source error tạo `status="error"` chứ không giả là fresh.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/freshness tests/scripts/test_check_updates.py -v`  
Expected: FAIL vì freshness service/CLI chưa tồn tại.

- [ ] **Step 3: Implement receipt và CLI**

```python
@dataclass(frozen=True)
class FreshnessReceipt:
    source_id: str
    resource_id: str
    checked_at: datetime
    validator_type: Literal["etag", "last_modified", "index_token", "content_hash", "none"]
    old_token: str | None
    new_token: str | None
    changed: bool
    status: Literal["ok", "not_modified", "error", "blocked"]
    valid_until: datetime
    checker_version: str
```

Signal priority: official index/feed → ETag/Last-Modified → document metadata → normalized legal-content hash → full fetch. CLI stdout là receipt JSON; lỗi input/network/policy dùng exit code đúng spec.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/freshness tests/scripts/test_check_updates.py -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/freshness scripts/check_updates.py tests/freshness tests/scripts/test_check_updates.py
git commit -m "feat: add targeted legal-source freshness checks"
```

### Task 5: Content-addressed artifacts và idempotent ingest

**Files:**

- Create: `src/plvn/ingestion/__init__.py`
- Create: `src/plvn/ingestion/artifacts.py`
- Create: `src/plvn/ingestion/ingest.py`
- Create: `scripts/crawl_updates.py`
- Create: `scripts/ingest_document.py`
- Test: `tests/ingestion/test_artifacts.py`
- Test: `tests/ingestion/test_ingest.py`

**Interfaces:**

- Consumes: changed freshness receipt, `HttpResponse`, append-only repository.
- Produces: `StoredArtifact`, `store_artifact(...)`, `legal_content_hash(...)`, `ingest_document(...) -> IngestResult`.

- [ ] **Step 1: Viết artifact/ingest tests**

```python
def test_store_artifact_is_content_addressed(tmp_path): ...
def test_ingest_same_source_url_and_hash_is_idempotent(repository): ...
def test_changed_legal_text_creates_new_version(repository): ...

def test_boilerplate_only_change_does_not_create_version(repository):
    first = ingest_document(html_with_banner_a, ...)
    second = ingest_document(html_with_banner_b_same_law, ...)
    assert first.version_id == second.version_id
    assert second.created is False
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/ingestion -v`  
Expected: FAIL vì ingestion package chưa tồn tại.

- [ ] **Step 3: Implement immutable artifacts và ingest**

```python
def store_artifact(root: Path, body: bytes, media_type: str) -> StoredArtifact: ...

def ingest_document(
    repository: LegalRepository, *, source: SourceDefinition,
    canonical_url: str, fetched_at: datetime, artifact: StoredArtifact,
    extracted_legal_text: str, parser_version: str,
) -> IngestResult: ...
```

Artifact path là `data/raw/sha256/<2-char-prefix>/<hash>`; write atomic và không overwrite. Lưu raw `content_hash` riêng với normalized `legal_content_hash`; mỗi fetch tạo/idempotently reuses `source_observation`, còn legal version dedupe theo hash pháp lý. Observation giữ provenance artifact/receipt kể cả khi không tạo legal version mới.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/ingestion -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/ingestion scripts/crawl_updates.py scripts/ingest_document.py tests/ingestion
git commit -m "feat: ingest immutable legal document artifacts"
```

### Task 6: Text/HTML/PDF extraction và legal-node parser

**Files:**

- Create: `src/plvn/parsing/__init__.py`
- Create: `src/plvn/parsing/extraction.py`
- Create: `src/plvn/parsing/legal_parser.py`
- Create: `scripts/parse_legal_document.py`
- Create: `tests/fixtures/legal/sample_document.txt`
- Create: `tests/fixtures/legal/sample_document.html`
- Test: `tests/parsing/test_extraction.py`
- Test: `tests/parsing/test_legal_parser.py`

**Interfaces:**

- Consumes: immutable raw artifact.
- Produces: `ExtractedText`, `extract_legal_text(...)`, `ParsedNode`, `parse_legal_document(...) -> tuple[ParsedNode, ...]`.

- [ ] **Step 1: Viết extraction/parser tests**

```python
def test_html_extraction_removes_script_style_comment_and_hidden_instructions(): ...
def test_text_layer_pdf_extracts_content(reportlab_pdf_bytes): ...
def test_scanned_pdf_without_text_returns_ocr_required(): ...

def test_parser_preserves_money_date_and_hierarchy(sample_text):
    nodes = parse_legal_document(sample_text, normalizer_version="v1")
    assert [n.locator for n in nodes if n.locator][:3] == (
        ["Điều 3", "Điều 3 Khoản 2", "Điều 3 Khoản 2 Điểm a"]
    )
    assert "500.000 đồng" in nodes[-1].text
    assert "01/07/2025" in nodes[-1].text
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/parsing -v`  
Expected: FAIL vì extraction/parser chưa tồn tại.

- [ ] **Step 3: Implement extraction và parser**

```python
@dataclass(frozen=True)
class ParsedNode:
    locator: str | None
    node_type: Literal["preamble", "chapter", "section", "article", "clause", "point"]
    parent_locator: str | None
    text: str
    node_hash: str
    ordinal: int
```

HTML extraction loại script/style/comment/hidden text. PDF chỉ hỗ trợ text layer bằng pypdf; không tự OCR. Nếu không đủ text, trả `ParseUnsupportedError(reason_code="OCR_REQUIRED")`. Parser deterministic nhận dạng Chương/Mục/Điều/Khoản/Điểm, giữ nguyên ký tự và thứ tự.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/parsing -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/parsing scripts/parse_legal_document.py tests/parsing tests/fixtures/legal
git commit -m "feat: extract and parse Vietnamese legal structure"
```

### Task 7: Legal relations và structured version diff

**Files:**

- Create: `src/plvn/relations/__init__.py`
- Create: `src/plvn/relations/builder.py`
- Create: `src/plvn/diffing/__init__.py`
- Create: `src/plvn/diffing/service.py`
- Create: `scripts/build_legal_relations.py`
- Create: `scripts/diff_versions.py`
- Test: `tests/relations/test_builder.py`
- Test: `tests/diffing/test_service.py`

**Interfaces:**

- Consumes: parsed nodes and document identities.
- Produces: `build_relations(...) -> tuple[LegalRelation, ...]`, `NodeDiff`, `diff_versions(...) -> tuple[NodeDiff, ...]`.

- [ ] **Step 1: Viết relation/diff tests**

```python
def test_relation_requires_explicit_basis_and_provenance(): ...

def test_diff_classifies_all_supported_changes(old_nodes, new_nodes):
    result = diff_versions(old_nodes, new_nodes)
    assert {item.kind for item in result} == {
        "unchanged", "amended", "new", "repealed", "moved"
    }
```

Thêm test thay đổi số tiền/thời hạn phải xuất `sensitive_change=True`; identity đích chưa resolve phải `verification_status="needs_review"`, không tự xác nhận.

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/relations tests/diffing -v`  
Expected: FAIL vì modules chưa tồn tại.

- [ ] **Step 3: Implement relation builder và node diff**

```python
def build_relations(
    source_version_id: int, nodes: Sequence[ParsedNode]
) -> tuple[LegalRelation, ...]: ...

def diff_versions(
    old_nodes: Sequence[ParsedNode], new_nodes: Sequence[ParsedNode]
) -> tuple[NodeDiff, ...]: ...
```

Chỉ extract quan hệ từ cụm rõ ràng: sửa đổi, bổ sung, thay thế, bãi bỏ, đình chỉ, hợp nhất, hướng dẫn; lưu locator/span làm provenance. Diff map locator trước, normalized similarity sau và không được suy luận ngày hiệu lực.

- [ ] **Step 4: Chạy test**

Run: `python -m pytest tests/relations tests/diffing -v`  
Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/plvn/relations src/plvn/diffing scripts/build_legal_relations.py scripts/diff_versions.py tests/relations tests/diffing
git commit -m "feat: derive legal relations and structured diffs"
```

### Task 8: Exact/FTS search và immutable corpus snapshots

**Files:**

- Create: `src/plvn/indexing/__init__.py`
- Create: `src/plvn/indexing/fts.py`
- Create: `src/plvn/indexing/snapshots.py`
- Create: `scripts/index_corpus.py`
- Create: `scripts/search_corpus.py`
- Create: `scripts/reconcile_corpus.py`
- Test: `tests/indexing/test_fts.py`
- Test: `tests/indexing/test_snapshots.py`
- Test: `tests/integration/test_data_pipeline.py`

**Interfaces:**

- Consumes: stored versions, nodes, relations and diff from Tasks 3–7.
- Produces: `rebuild_version_index(...)`, `search_nodes(...) -> tuple[SearchHit, ...]`, `publish_snapshot(...) -> str`, `get_snapshot_version_ids(...) -> tuple[int, ...]`, `reconcile_corpus(...) -> ReconciliationReport` plus offline CLI pipeline.

- [ ] **Step 1: Viết index/snapshot/integration tests**

```python
def test_exact_document_number_precedes_bm25_matches(): ...
def test_snapshot_remains_immutable_after_later_ingest(): ...
def test_snapshot_publish_rolls_back_when_index_write_fails(): ...
def test_reconciliation_detects_validator_miss_from_legal_content_hash(): ...

def test_fixture_pipeline_ingests_parses_indexes_and_searches(tmp_path):
    snapshot_id = run_fixture_pipeline(tmp_path)
    hits = search_nodes(..., snapshot_id=snapshot_id, limit=12)
    assert hits[0].locator == "Điều 3 Khoản 2 Điểm a"
```

- [ ] **Step 2: Chạy test và xác nhận RED**

Run: `python -m pytest tests/indexing tests/integration/test_data_pipeline.py -v`  
Expected: FAIL vì indexing/snapshot chưa tồn tại.

- [ ] **Step 3: Implement index và atomic snapshot publish**

```python
def search_nodes(
    connection: sqlite3.Connection, query: str, *, snapshot_id: str,
    as_of: date | None = None, limit: int = 12,
) -> tuple[SearchHit, ...]: ...

def publish_snapshot(
    connection: sqlite3.Connection, *, version_ids: Sequence[int],
    index_version: str, created_at: datetime,
) -> str: ...
```

Exact normalized number/locator trước FTS5/BM25. Snapshot manifest là SHA-256 của ordered version IDs + index version; publish transaction rollback toàn bộ nếu index lỗi; trigger chặn mutation sau publish.

- [ ] **Step 4: Chạy full deterministic verification**

Run: `python -m pytest tests -v`  
Expected: PASS, không network/model call.  
Run: `python -m ruff check src tests scripts`  
Expected: PASS.  
Run: `python scripts/ingest_document.py --help` và `python scripts/index_corpus.py --help`  
Expected: exit `0` và help text.

- [ ] **Step 5: Kiểm tra repository hygiene**

Run: `git status --ignored --short`  
Expected: mọi `data/`, SQLite, raw artifact, cache và log chỉ hiện dưới dạng ignored; không có secret/user document tracked.  
Run: `git diff --check`  
Expected: không có output.

- [ ] **Step 6: Commit**

```powershell
git add src/plvn/indexing scripts/index_corpus.py scripts/search_corpus.py scripts/reconcile_corpus.py tests/indexing tests/integration/test_data_pipeline.py
git commit -m "feat: publish immutable searchable corpus snapshots"
```

## Plan 1 Completion Gate

- Mọi test/lint trong plan pass offline.
- Fixture đi qua `check → ingest → parse → relation/diff → index → snapshot → search` và có thể tái lập.
- External method/path/redirect không hợp lệ fail trước transport.
- Corpus version và snapshot cũ không thể bị overwrite/delete qua repository API.
- Không có model, runtime subagent, external write, communication, deploy hoặc push.
