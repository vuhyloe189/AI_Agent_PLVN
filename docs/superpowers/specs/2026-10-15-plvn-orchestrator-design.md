# Đặc tả thiết kế PLVN Orchestrator

**Ngày:** 2026-10-15  
**Trạng thái:** Đã được người dùng duyệt; bổ sung ranh giới không vượt quyền/không giao tiếp hoặc giao dịch trực tiếp ngày 2026-10-15  
**Phạm vi:** MVP chạy nội bộ/local, trả lời hoàn toàn bằng tiếng Việt

## 1. Mục tiêu

Xây dựng một hệ thống hỏi đáp pháp luật Việt Nam có một **PLVN Orchestrator** duy nhất. Orchestrator chọn skill phù hợp, gọi các script deterministic để kiểm tra/cập nhật/tra cứu dữ liệu và chỉ dùng mô hình ngôn ngữ cho những việc cần hiểu hoặc lập luận.

Hệ thống phải:

- tra cứu quy định hiện hành hoặc tại một thời điểm lịch sử;
- so sánh phiên bản và chỉ ra thay đổi nếu có dữ liệu phù hợp;
- phân tích tình huống giả định hoặc thực tế theo dữ kiện được cung cấp;
- nêu rõ căn cứ đến Điều/Khoản/Điểm, nguồn và ngày áp dụng;
- luôn kiểm tra độ mới của dữ liệu nhưng không tìm lại toàn bộ web cho mỗi câu hỏi;
- biết dừng, nói rõ bất định hoặc đề nghị chuyên gia pháp lý khi không đủ căn cứ.

Đây là hệ thống cung cấp thông tin và hỗ trợ phân tích pháp lý. Nó không tự nhận là luật sư, không tạo quan hệ luật sư–khách hàng và không thay thế việc rà soát hồ sơ gốc trong vụ việc có hậu quả nghiêm trọng.

## 2. Phạm vi MVP và ngoài phạm vi

### Trong phạm vi

- Văn bản quy phạm pháp luật Việt Nam cấp trung ương; kiến trúc dữ liệu cho phép bổ sung văn bản địa phương.
- Nạp văn bản từ URL/ID đã được phê duyệt và theo dõi thay đổi theo watchlist.
- Lưu nhiều phiên bản, quan hệ sửa đổi/bổ sung/thay thế/bãi bỏ/đình chỉ/hợp nhất và hiệu lực theo thời gian.
- Tra cứu exact + full-text theo số hiệu, tên văn bản, Điều/Khoản/Điểm và nội dung.
- So sánh phiên bản có cấu trúc.
- Phân tích tình huống theo mẫu vấn đề → quy tắc → áp dụng → phản biện → kết luận có điều kiện.
- CLI cho tác vụ vận hành và một API nội bộ tối thiểu cho hỏi đáp.
- Bộ kiểm thử deterministic, bộ eval cho mô hình, nhật ký kiểm toán không lưu dữ liệu cá nhân quá mức cần thiết.

Corpus khởi đầu tập trung vào nhóm văn bản nền tảng có nhu cầu cao: dân sự, lao động, doanh nghiệp, đất đai, xử lý vi phạm hành chính và các văn bản hướng dẫn trực tiếp. Pipeline vẫn nhận được văn bản thuộc lĩnh vực khác; nếu corpus chưa đủ, hệ thống phải dùng live fallback có kiểm soát hoặc nói rõ giới hạn, không giả vờ bao phủ toàn bộ pháp luật Việt Nam.

Với mỗi văn bản seed, MVP thu thập các phiên bản lịch sử chính thức còn truy cập được và công khai mức bao phủ theo thời gian. “Read-only” áp dụng với hệ thống nguồn bên ngoài; hệ thống được ghi local theo kiểu append-only vào corpus, receipt và audit store. Chỉ workflow bảo trì đã xác thực mới được publish snapshot local mới.

### Ngoài phạm vi MVP

- Website công khai hoàn chỉnh, tài khoản người dùng, thanh toán hoặc quản trị đa tổ chức.
- Tự động gửi hồ sơ, giao tiếp với cơ quan nhà nước hoặc hành động thay người dùng.
- Gửi email/tin nhắn, gọi điện, đăng bài/bình luận, nộp biểu mẫu, tải hồ sơ lên dịch vụ bên thứ ba, ký/chấp thuận, thanh toán hoặc thực hiện bất kỳ giao dịch trực tiếp nào.
- Đại diện, mạo danh hoặc tuyên bố có thẩm quyền thay người dùng, luật sư, cơ quan nhà nước hay tổ chức khác.
- Thu thập vượt paywall, CAPTCHA, cơ chế chống bot, robots.txt, điều khoản sử dụng hoặc giấy phép nội dung.
- Fine-tuning mô hình.
- Semantic/vector search bắt buộc. Chỉ bổ sung khi eval chứng minh full-text/exact search chưa đạt ngưỡng.
- Cam kết kết quả tranh tụng hoặc đưa ra chỉ dẫn cá biệt có rủi ro cao mà không có human gate.

## 3. Nguyên tắc thiết kế

1. **Một Orchestrator, không có runtime subagent mặc định.** Mọi nhánh công việc được điều phối bằng state machine và tool call rõ ràng.
2. **Script trước, mô hình sau.** Crawl, hash, parse, diff, index, search và kiểm tra trích dẫn là code deterministic, không gọi API mô hình.
3. **Nguồn chính thức quyết định.** Nguồn thương mại/thứ cấp hỗ trợ discovery và diễn giải; claim quan trọng phải được xác minh bằng nguồn chính thức khi nguồn đó tồn tại.
4. **Snapshot bất biến.** Một lượt trả lời chỉ đọc một `corpus_snapshot_id`; phiên bản mới phát hiện giữa lượt chỉ áp dụng cho lượt sau.
5. **Có thời điểm áp dụng.** Mọi yêu cầu có `as_of`; mặc định là ngày hiện tại tại múi giờ `Asia/Ho_Chi_Minh` và phải hiện rõ trong câu trả lời.
6. **Không trộn dữ kiện với giả định.** Dữ kiện đã biết, giả định, điểm tranh chấp và thông tin còn thiếu được lưu riêng.
7. **Verifier có quyền phủ quyết.** Không xuất kết luận pháp lý trọng yếu nếu căn cứ, hiệu lực hoặc freshness chưa qua gate.
8. **Không bỏ phiếu để xác định pháp luật.** Xung đột được xử lý bằng nguồn, thẩm quyền, thời gian, phạm vi và điều khoản chuyển tiếp.
9. **Tối ưu theo chất lượng tối thiểu trước, chi phí sau.** Chỉ dùng effort `max` khi mức rủi ro/độ phức tạp hoặc kết quả eval biện minh.
10. **Least privilege và external read-only.** Tool runtime chỉ được đọc nguồn pháp luật công khai bằng `GET`/`HEAD` theo allowlist; không có tool gửi thông tin, giao dịch hoặc thay mặt người dùng tới bên thứ ba.
11. **Không hỗ trợ lừa đảo hoặc vượt quyền.** Yêu cầu giả mạo, lừa dối, scam, tạo/chỉnh chứng cứ giả, che giấu hoặc tiêu hủy chứng cứ, né nghĩa vụ hay truy cập trái phép phải bị chặn; hệ thống có thể giải thích hậu quả pháp lý và lựa chọn hợp pháp an toàn.

## 4. Kiến trúc tổng thể

```text
Người dùng / API
       |
       v
PLVN Orchestrator
  |-- Intake + phân loại rủi ro
  |-- Chọn skill
  |-- Chọn model profile / reasoning effort
  |-- Điều phối tool calls theo state machine
  |-- Render câu trả lời tiếng Việt
       |
       +--> Skill: tra cứu / so sánh / phân tích / bảo trì
       |
       +--> Tool adapters (JSON vào/ra)
                |
                +--> Script deterministic
                |      check -> crawl -> parse -> diff -> index -> search
                |
                +--> Legal Store + Freshness Store + Audit Store
       |
       +--> Evidence/Citation/Temporal/Safety gates
```

Kiến trúc có hai mặt phẳng tách biệt:

- **Data plane:** script kiểm tra nguồn, nạp, parse, phiên bản hóa và lập chỉ mục. Chỉ data plane được tạo snapshot mới.
- **Answer plane:** Orchestrator và skill chỉ đọc snapshot đã pin, thu evidence, lập luận và tạo đáp án.

Ranh giới này ngăn một câu trả lời dùng lẫn phiên bản cũ/mới và giúp tái lập chính xác đầu ra.

## 5. PLVN Orchestrator

Orchestrator là state machine có các trạng thái tối thiểu:

```text
INTAKE
-> RISK_AND_SCOPE
-> FRESHNESS
-> SNAPSHOT_PINNED
-> RETRIEVAL
-> REASONING
-> VERIFY
-> RENDER
-> COMPLETE | ABSTAIN | ESCALATE | REFUSE
```

Trách nhiệm:

- chuẩn hóa câu hỏi, ngày áp dụng, phạm vi lãnh thổ, chủ thể và lĩnh vực;
- phân loại yêu cầu là tra cứu, so sánh, phân tích hay bảo trì;
- xác định dữ kiện đã biết/giả định/thiếu;
- lấy hoặc làm mới freshness receipt;
- pin snapshot;
- gọi đúng skill và tool, với giới hạn thời gian/số lần gọi;
- chọn model profile từ cấu hình, không hard-code model ID trong business logic;
- chạy gate bắt buộc và chỉ render từ evidence đã xác minh;
- lưu trace đã redaction để audit.

Giới hạn vòng lặp:

- tối đa một lần sửa đáp án sau khi verifier từ chối;
- tối đa một lần nâng model profile trong một request;
- tối đa 12 tool invocation ở answer plane; tác vụ ingest nhiều tài liệu phải batch trong data plane;
- tối đa 2 lần retry cho một thao tác mạng idempotent;
- không retry vô hạn; lỗi nguồn/model phải có reason code;
- retry HTTP chỉ nằm trong script, có exponential backoff, jitter và giới hạn cấu hình.

## 6. Các skill

Skill chỉ chứa hướng dẫn nghiệp vụ ngắn, điều kiện kích hoạt, output contract và liên kết đến tài liệu tham chiếu. Skill không tự crawl, không chứa corpus lớn và không thay thế code deterministic.

### `plvn-tra-cuu`

- Dùng cho câu hỏi về quy định hiện hành hoặc lịch sử.
- Bắt buộc xác định `as_of`, lấy exact locator nếu người dùng nêu Điều/Khoản/Điểm và ưu tiên nguồn chính thức.
- Trả về quy định, phạm vi áp dụng, trạng thái hiệu lực, căn cứ, bất định và freshness.

### `plvn-so-sanh`

- Dùng khi hỏi trước/sau, cũ/mới, thay đổi hoặc tác động của văn bản sửa đổi.
- Dựng lineage trước khi diff.
- Phân loại node là `unchanged`, `amended`, `new`, `repealed` hoặc `moved`; nêu ngày thay đổi có hiệu lực và điều khoản chuyển tiếp nếu có.

### `plvn-phan-tich`

- Dùng cho tình huống giả định hoặc thực tế.
- Phân tích theo từng issue: dữ kiện → câu hỏi pháp lý → quy tắc/điều kiện → ngoại lệ/chuyển tiếp → áp dụng → phản biện → kết luận có điều kiện.
- Không biến giả định thành sự kiện. Với dữ kiện thiếu có thể đảo kết luận, trả về các nhánh `nếu...thì...` hoặc hỏi thêm.

### `plvn-bao-tri`

- Dùng cho người vận hành để xem source health, chạy targeted refresh, reconcile, reindex và eval.
- Không được kích hoạt từ nội dung không tin cậy được crawl về.
- Thao tác ghi phải dùng allowlist lệnh, idempotency key và audit log.

### Gate không phải skill

Các kiểm tra citation, hiệu lực, snapshot, privacy, prompt injection và policy là code gate bắt buộc. Orchestrator không được bỏ qua gate chỉ vì một skill hoặc mô hình nói rằng câu trả lời đã đúng.

## 7. Script deterministic và giao diện

Mỗi script có CLI mỏng gọi logic trong package `src/plvn`, nhận tham số rõ ràng và trả JSON có schema version. Các script dữ liệu/vận hành không gọi LLM API. `run_evals.py` là test harness riêng và chỉ được gọi workflow/model khi người vận hành bật rõ chế độ eval.

| Script | Trách nhiệm chính |
|---|---|
| `check_updates.py` | Kiểm tra delta cho nguồn/watchlist và tạo freshness receipt |
| `crawl_updates.py` | Tải đúng tài nguyên đã thay đổi theo source policy |
| `ingest_document.py` | Lưu raw artifact, hash, provenance và tạo document version |
| `parse_legal_document.py` | Chuẩn hóa, tách Chương/Mục/Điều/Khoản/Điểm |
| `build_legal_relations.py` | Tạo quan hệ sửa đổi/thay thế/bãi bỏ/hợp nhất/hướng dẫn |
| `diff_versions.py` | Diff ở cấp legal node và dữ liệu nhạy như số tiền/thời hạn |
| `index_corpus.py` | Cập nhật exact index và SQLite FTS5 theo snapshot |
| `search_corpus.py` | Tìm ứng viên theo query, `as_of`, source tier và snapshot |
| `get_evidence.py` | Đóng gói evidence object có supporting span |
| `verify_answer.py` | Kiểm tra locator, nội dung hỗ trợ, temporal status và source policy |
| `reconcile_corpus.py` | Full-fetch/rehash định kỳ để bắt thay đổi bị bỏ sót |
| `run_evals.py` | Chạy eval suite có cấu hình; đây là script duy nhất được phép gọi workflow/model qua adapter riêng |

Quy ước chung:

- JSON stdout; log người vận hành đi stderr;
- exit code `0` thành công, `2` dữ liệu/đầu vào không hợp lệ, `3` nguồn tạm lỗi, `4` policy block, `5` lỗi nội bộ;
- thao tác ghi idempotent theo `source_id + canonical_url + content_hash`;
- timeout, retry, rate limit và user-agent định danh được cấu hình theo nguồn;
- không tải URL ngoài registry/allowlist sau redirect;
- raw content luôn được xem là `UNTRUSTED_EVIDENCE`.

## 8. Nguồn và chính sách xác minh

Đây là thứ tự tin cậy để xác minh nguồn, không phải bảng thứ bậc hiệu lực pháp lý:

1. **Tier 0 — artifact chính thức gốc:** bản ký số/bản đăng chính thức của cơ quan ban hành và Công báo điện tử. Đây là evidence cuối cùng về toàn văn, đính chính và văn bản công bố bãi bỏ.
2. **Tier 1 — CSDL quốc gia:** `vbpl.vn` dùng cho metadata, trạng thái, lược đồ, quan hệ và văn bản hợp nhất; đối chiếu Tier 0 khi rủi ro cao hoặc có bất nhất.
3. **Tier 2 — cổng chính thức theo thẩm quyền:** Quốc hội, Chính phủ, bộ/ngành, địa phương và TANDTC. Chỉ dùng trong đúng phạm vi loại tài liệu của cơ quan đó.
4. **Tier 3 — nguồn tổng hợp uy tín:** THƯ VIỆN PHÁP LUẬT, LuatVietnam và nguồn chuyên ngành đã phê duyệt. Dùng cho discovery, synonym và quan hệ khả dĩ. Mọi kết luận pháp lý lấy từ Tier 3 phải được Tier 0–2 xác minh trước khi xuất.
5. **Nguồn tham khảo ngoài registry:** bài báo, blog, diễn đàn không được dùng làm evidence. Chúng chỉ có thể tạo đề xuất nguồn để người vận hành xem xét và thêm vào registry.

Registry của mỗi nguồn chứa:

```text
source_id, domains, tier, document_types, authority_scope,
allowed_paths, robots_policy, terms/licensing_notes,
allowed_use, attribution_required, robots_checked_at,
rate_limit, timeout, validators, parser_profile,
health_status, last_success_at, next_check_at
```

Các nguồn ưu tiên ban đầu:

- `congbao.chinhphu.vn`, `vbpl.vn`, `vanban.chinhphu.vn`, `chinhphu.vn`;
- cổng Quốc hội, bộ/ngành và TANDTC khi tài liệu thuộc thẩm quyền tương ứng;
- `thuvienphapluat.vn` và `luatvietnam.vn` ở Tier 3.

Không sử dụng API nội bộ không được công bố, không né paywall/CAPTCHA và không crawl khi robots/điều khoản/giấy phép không cho phép. Khi không được phép tự động thu thập, hệ thống chỉ lưu metadata/link hợp lệ hoặc yêu cầu người vận hành cung cấp tài liệu có quyền sử dụng.

## 9. Freshness, cache và cập nhật tăng dần

“Luôn kiểm tra” được thực hiện bằng một freshness receipt có thời hạn, không đồng nghĩa luôn tải toàn văn.

```text
source_id, resource_id, checked_at, validator_type,
old_token, new_token, changed, status, error,
valid_until, checker_version
```

Thứ tự tín hiệu:

1. feed/index/sitemap hoặc danh mục văn bản mới chính thức;
2. ETag/Last-Modified qua conditional request nếu endpoint hỗ trợ;
3. metadata/lược đồ/trạng thái của document watchlist;
4. hash nội dung pháp lý đã loại boilerplate;
5. full-fetch và so hash;
6. full reconciliation định kỳ.

Default SLA của MVP:

- poll index/feed chính thức mỗi 2 giờ;
- receipt tối đa 6 giờ cho câu hỏi thông thường về pháp luật hiện hành;
- receipt tối đa 1 giờ cho văn bản mới/thay đổi trong 30 ngày gần nhất hoặc câu hỏi rủi ro cao;
- receipt tối đa 24 giờ cho nguồn Tier 3 vì không dùng làm căn cứ quyết định;
- nightly incremental index;
- full reconciliation mỗi tuần;
- người vận hành có thể ép `--force-refresh` cho một nguồn/tài liệu cụ thể.

Khi request đến:

1. tìm ứng viên trong local index;
2. kiểm tra receipt của đúng tài liệu/quan hệ nguồn;
3. nếu còn hạn, dùng lại và không gọi web;
4. nếu hết hạn, chạy targeted `check_updates`;
5. chỉ tải/parse/reindex tài liệu có token hoặc legal-content hash thay đổi;
6. publish snapshot mới theo kiểu atomic;
7. pin snapshot rồi mới retrieval/analysis.

Cache key tối thiểu:

- raw: `source_id + content_hash` (immutable);
- parsed: `content_hash + parser_version + normalizer_version`;
- index node: `version_id + node_id + index_version`;
- retrieval: `normalized_query + as_of + snapshot_id + retrieval_config_version`;
- analysis: `issue_signature + redacted_fact_hash + snapshot_id + policy_version + model_profile`.

Negative result cache dùng `normalized_query + source_scope + snapshot_id`, TTL tối đa 1 giờ. Một kết quả “không tìm thấy” không được tái sử dụng sau khi source token hoặc snapshot thay đổi.

Không cache đáp án tình huống thực tế có PII ở dạng raw. Invalidation theo dependency graph `source revision → legal nodes → retrieval → analysis`, tránh xóa toàn bộ cache vì thay đổi giao diện không liên quan nội dung pháp lý.

## 10. Kho dữ liệu và mô hình phiên bản

MVP dùng Python 3.12 và SQLite có WAL + FTS5 để giảm chi phí vận hành local. Raw artifacts được lưu content-addressed trong thư mục `data/` bị gitignore. Repository dùng storage interface để có thể chuyển sang PostgreSQL/pgvector sau này mà không đổi hợp đồng Orchestrator.

Thực thể lõi:

- `LegalDocument`: loại, số/ký hiệu, tên, cơ quan, phạm vi, ngày ban hành.
- `DocumentVersion`: source URL, fetched time, content hash, parser version, raw artifact, effective interval, status.
- `LegalNode`: Chương/Mục/Điều/Khoản/Điểm, stable locator, text, node hash, effective interval.
- `LegalRelation`: source/target, relation type, legal basis, effective date, confidence, provenance.
- `CorpusSnapshot`: immutable manifest của document/version/index.
- `FreshnessReceipt`: bằng chứng kiểm tra độ mới.
- `Evidence`: document/version/node, supporting span, source, temporal status.
- `RunManifest`: request ID, snapshot, policy/prompt/model config, tool calls, latency, cost/usage và kết quả gate.

Không ghi đè phiên bản. Văn bản hợp nhất là một artifact hỗ trợ đọc và tra cứu, không được mô hình hóa như văn bản tự tạo ra thay đổi pháp luật.

Mọi khoảng hiệu lực nội bộ dùng quy ước half-open `[effective_from, effective_to_exclusive)`. Trường kết thúc để `null` khi chưa xác định ngày hết hiệu lực. Adapter nguồn phải chuyển metadata của nguồn về quy ước này trước khi temporal retrieval hoặc verifier sử dụng.

## 11. Retrieval và evidence

Pipeline MVP:

1. exact match số/ký hiệu và locator;
2. filter theo `as_of`, trạng thái hiệu lực, phạm vi và source tier;
3. SQLite FTS5/BM25;
4. rerank deterministic theo exactness, authority, temporal validity và độ gần locator;
5. mở rộng parent/child context để không tách Điểm khỏi Khoản/Điều;
6. tạo evidence object có hash và supporting span.

Mỗi issue lấy tối đa 12 candidate node sau rerank và tối đa 8 evidence node vào context; toàn request tối đa 32 evidence node. Exact locator do người dùng chỉ định luôn được giữ, không bị loại chỉ vì giới hạn top-k. Nếu giới hạn làm coverage chưa đủ, Orchestrator chia issue hoặc abstain thay vì cắt ngầm căn cứ.

Semantic search chỉ được thêm sau khi có baseline eval và chứng minh cải thiện recall mà không tăng sai citation vượt ngưỡng. Live web fallback phải đi qua cùng source registry, parser, evidence schema và verifier; mô hình không được tự mở web rồi trích dẫn trực tiếp ngoài pipeline.

## 12. Routing model và reasoning effort

Business logic dùng profile, còn model ID cụ thể nằm trong `config/routing.yaml` và được chọn bằng benchmark trên eval set.

| Profile | Công việc | Effort mặc định |
|---|---|---|
| `D0_DETERMINISTIC` | crawl, parse, hash, diff, index, locator verification | không dùng model |
| `M1_FAST` | FAQ rõ, một nguồn sơ cấp, rủi ro thấp | low |
| `M2_BALANCED` | nhiều văn bản, query expansion, tóm tắt diff | medium |
| `M3_REASONING` | áp dụng tình huống, ngoại lệ, chuyển tiếp | high |
| `M4_CRITICAL` | hình sự/tố tụng/deadline/chế tài lớn, nguồn xung đột, verifier fail | xhigh hoặc max |

Trigger nâng tầng gồm: hậu quả mất tự do/quyền/tài sản lớn; thời hạn gần; văn bản vừa đổi; điều khoản chuyển tiếp; nguồn chính thức xung đột; citation hoặc temporal gate thất bại; kết luận phụ thuộc dữ kiện chưa rõ.

`max` không dùng cho crawl, retrieval hoặc câu hỏi đơn giản. Nó chỉ được kích hoạt khi `risk_level=critical`, hoặc khi route thấp hơn thất bại và eval cho thấy `max` nâng chất lượng đáng kể. Sau một lần sửa/nâng tầng mà gate vẫn fail, hệ thống abstain/escalate.

## 13. Hợp đồng đầu ra

Đầu ra nội bộ có schema máy kiểm được; phần hiển thị được render bằng tiếng Việt:

```json
{
  "answer_type": "general_info|contextual_analysis|abstain|escalate|refuse",
  "jurisdiction": "VN",
  "as_of": "YYYY-MM-DD",
  "snapshot_id": "...",
  "facts_confirmed": [],
  "facts_assumed": [],
  "missing_facts": [],
  "issues": [],
  "claims": [
    {
      "text": "...",
      "criticality": "critical|material|context",
      "evidence_ids": [],
      "support": "supported|partial|unsupported|conflicted",
      "temporal_status": "valid|superseded|future|unknown"
    }
  ],
  "uncertainties": [],
  "freshness": {},
  "confidence_band": "high|medium|low",
  "recommended_next_step": "..."
}
```

Câu trả lời cho người dùng có thứ tự mặc định:

1. kết luận ngắn và có điều kiện;
2. mốc thời gian/phạm vi đang áp dụng;
3. phân tích theo từng vấn đề;
4. căn cứ trực tiếp đến Điều/Khoản/Điểm và URL;
5. so sánh cũ/mới khi liên quan;
6. giả định, bất định và dữ kiện còn thiếu;
7. trạng thái kiểm tra độ mới;
8. bước tiếp theo an toàn.

Confidence không phải phần trăm do model tự đoán; nó được suy ra từ coverage nguồn, temporal validity, độ đủ dữ kiện, xung đột và gate.

## 14. An toàn, privacy và prompt injection

Mức xử lý:

- `L0`: thông tin pháp luật chung — trả lời trực tiếp có nguồn.
- `L1`: định hướng theo tình huống — phân tích có điều kiện, nêu dữ kiện thiếu.
- `L2`: quyết định cá biệt/rủi ro cao — cung cấp nền tảng, bất định và human escalation; không cam kết kết quả.
- `L3`: làm giả, che giấu/tiêu hủy chứng cứ, doxxing, vượt quyền, mạo danh — từ chối phần không an toàn và chuyển sang lựa chọn hợp pháp.

Hệ thống chỉ tạo **thông tin để người dùng tự xem xét**. Nó không được trao đổi trực tiếp với cá nhân/cơ quan, không gửi hoặc nộp nội dung, không tham gia giao dịch, không nhận/chuyển tiền, không tạo danh tính giả và không tuyên bố đại diện người dùng. Tool policy không cấp external write capability; nếu nội dung nguồn hoặc output mô hình yêu cầu `POST`, `PUT`, `PATCH`, `DELETE`, upload, login hoặc gửi tin thì Orchestrator phải chặn với reason code `EXTERNAL_ACTION_NOT_ALLOWED`.

Các tác vụ ghi hợp lệ chỉ giới hạn ở repository, corpus append-only, cache và audit store local trong phạm vi người dùng đã cho phép. Mọi thay đổi local có tính phá hủy vẫn cần target rõ ràng và quy trình phê duyệt riêng.

Mọi HTML/PDF/OCR/metadata là dữ liệu không tin cậy. Pipeline loại script/style/comment, đánh dấu `UNTRUSTED_EVIDENCE`, chặn redirect ngoài allowlist và không bao giờ thực thi lệnh nằm trong nội dung nguồn. Model output không được truyền thẳng vào shell, SQL hoặc API ghi.

Đầu vào có thông tin cá nhân được redaction trước model call, trace và eval. Mặc định chỉ log metadata; raw hồ sơ/tài liệu người dùng không được commit và phải có TTL nếu được lưu tạm. Eval dùng PII tổng hợp và canary.

## 15. Trạng thái dừng và xử lý lỗi

- **Abstain:** không xác minh được nguồn/hiệu lực/phiên bản, evidence thiếu hoặc dữ kiện thiếu có thể đảo kết luận.
- **Escalate:** vẫn cung cấp thông tin nền nhưng khuyến nghị luật sư/cơ quan chuyên môn vì rủi ro cao, thời hạn gần, cần xem hồ sơ gốc hoặc còn cách hiểu cạnh tranh.
- **Refuse:** yêu cầu hành vi trái pháp luật/không an toàn hoặc vượt quyền hệ thống.

Nếu nguồn chính thức tạm lỗi và receipt hết hạn, hệ thống không được tuyên bố “đã cập nhật mới nhất”. Có thể trả dữ liệu snapshot gần nhất kèm thời điểm kiểm tra cuối và cảnh báo rõ, hoặc abstain nếu claim trọng yếu phụ thuộc bản mới.

Nếu hai nguồn khác nhau:

1. xác nhận cùng document identity, snapshot và `as_of`;
2. ưu tiên artifact chính thức phù hợp và so hash;
3. kiểm tra sửa đổi, thay thế, bãi bỏ, đình chỉ và chuyển tiếp;
4. áp dụng authority policy được phiên bản hóa;
5. nếu vẫn không giải quyết được, giữ hai cách hiểu và escalate, không chọn theo đa số.

## 16. `tests` và `evals`

### `tests/`

Kiểm tra code deterministic, chạy nhanh/local và mặc định không gọi model API:

- parser tách đúng Điều/Khoản/Điểm và giữ nguyên số tiền/thời hạn;
- unchanged receipt không tải lại, changed hash tạo version mới;
- thao tác ingest idempotent;
- diff phân loại đúng node thêm/sửa/xóa/di chuyển;
- retrieval không dùng văn bản hết hiệu lực cho query hiện hành;
- snapshot không đổi giữa một workflow;
- verifier từ chối locator sai, claim thiếu căn cứ và redirect ngoài allowlist;
- redaction không để canary xuất hiện trong log/output.

`tests` tồn tại để phát hiện lỗi code, lỗi parser, cache invalidation và regression mà không cần trả tiền cho model.

### `evals/`

Đo chất lượng toàn workflow có mô hình:

- frozen regression set với source snapshot/hash cố định;
- live freshness set;
- holdout red-team set cho prompt injection, PII, nguồn giả và context dài;
- nhóm normal, edge và adversarial;
- chấm groundedness, citation correctness, effective-date correctness, version comparison, routing, refusal/abstention, latency và cost per successful task.

Mô hình tạo đáp án không được là judge duy nhất. Case critical phải có gold label được chuyên gia pháp lý rà soát trước khi dùng làm release gate.

Release suite ban đầu có tối thiểu 120 case: 40 normal, 30 edge và 50 adversarial; ít nhất 20% là holdout khác cả câu hỏi lẫn tài liệu. Ngưỡng sàn: task success toàn bộ `>=90%`, normal `>=95%`, edge `>=85%`, weighted groundedness `>=95%`, citation entailment `>=98%`, effective-date correctness toàn bộ `>=98%`, false refusal `<=5%`; các chỉ số critical vẫn phải đạt 100% theo release gate bên dưới.

## 17. Release gates của MVP

Một release chỉ được tạo khi:

- toàn bộ deterministic tests và schema/contract tests đều pass;
- 100% claim `critical` trong eval release có citation và temporal status đã xác minh;
- 0 citation bịa hoặc locator critical sai;
- 0 lỗi critical về hiệu lực/ngày áp dụng/chuyển tiếp trong release set;
- 0 lần làm theo prompt injection trong red-team set và báo cáo rõ cỡ mẫu;
- 0 PII/secret canary xuất hiện trong output, log hoặc URL;
- mọi abstain/refuse/escalate có reason code;
- retry bị giới hạn và không có workflow loop vô hạn;
- p95 latency và cost per successful task nằm trong ngân sách cấu hình đã chốt từ benchmark ban đầu;
- khi chưa chốt ngân sách tuyệt đối, release candidate không được làm p95 latency hoặc cost per successful task tăng quá 10% so với baseline nếu không có cải thiện chất lượng được duyệt;
- secret scan sạch; corpus, cache, index, log, `.env` và hồ sơ thật không nằm trong Git.

Không được diễn giải “0 lỗi trong tập kiểm thử” thành “rủi ro bằng 0”. Báo cáo release phải ghi số case, miền luật, snapshot và giới hạn bao phủ.

## 18. Cấu trúc repository dự kiến

```text
AI_Agent_PLVN/
|-- src/plvn/
|   |-- orchestrator/
|   |-- routing/
|   |-- skills_runtime/
|   |-- sources/
|   |-- freshness/
|   |-- ingestion/
|   |-- legal_store/
|   |-- retrieval/
|   |-- verification/
|   `-- api/
|-- skills/
|   |-- plvn-tra-cuu/
|   |-- plvn-so-sanh/
|   |-- plvn-phan-tich/
|   `-- plvn-bao-tri/
|-- scripts/
|-- config/
|   |-- sources.yaml
|   |-- freshness.yaml
|   |-- routing.yaml
|   `-- policy.yaml
|-- tests/
|-- evals/
|-- docs/
|-- data/                  # gitignored
|-- .env.example
|-- pyproject.toml
`-- docker-compose.yml     # chỉ thêm khi có dependency cần container
```

Logic nằm trong `src/plvn`; `scripts` chỉ là entrypoint mỏng cho Orchestrator, Windows Task Scheduler và CI.

## 19. Vận hành local và Git

- Windows Task Scheduler gọi script theo lịch; không cần daemon riêng trong MVP.
- Cấu hình và migration được version control; raw corpus/index/cache/log/secret bị gitignore.
- Repository local sẽ được khởi tạo Git ở giai đoạn implementation sau khi kế hoạch được duyệt.
- Chỉ push khi test/eval/release gates và secret scan pass. Việc push cần URL remote và quyền truy cập do người dùng cung cấp hoặc đã cấu hình sẵn.

## 20. Phân kỳ triển khai

Do hệ thống gồm nhiều subsystem độc lập, sau khi đặc tả này được duyệt sẽ có ba implementation plan riêng, mỗi plan tạo ra phần mềm chạy và kiểm thử được:

1. **Data foundation:** source registry, SQLite schema, freshness, crawl/ingest/parse/version/diff/index và deterministic tests.
2. **Orchestrator + skills:** state machine, routing, retrieval/evidence, bốn skill, API/CLI và snapshot pinning.
3. **Quality + release:** citation/temporal/safety gates, eval harness, red-team/privacy, observability, scheduler, CI và release checklist.

Mỗi task trong các plan dùng TDD: viết test fail → chạy xác nhận fail → code tối thiểu → chạy pass → commit nhỏ. Việc triển khai chỉ bắt đầu sau khi người dùng duyệt đặc tả, duyệt các plan và chọn cách thực thi.

## 21. Tiêu chí hoàn thành sản phẩm MVP

MVP được coi là hoàn thành khi người vận hành có thể:

1. thêm một văn bản/nguồn hợp lệ vào registry/watchlist;
2. chạy check và chỉ tải lại khi nội dung pháp lý thay đổi;
3. thấy lịch sử phiên bản, quan hệ và structured diff;
4. hỏi bằng tiếng Việt về quy định hiện hành hoặc lịch sử;
5. nhận phân tích tình huống có giả định, căn cứ, ngày áp dụng và freshness;
6. thấy hệ thống abstain/escalate đúng khi thiếu nguồn hoặc rủi ro cao;
7. tái lập câu trả lời từ snapshot/run manifest;
8. chạy toàn bộ tests/evals và nhận báo cáo chất lượng, latency, chi phí;
9. đẩy repository sạch lên remote mà không chứa corpus, secret hay hồ sơ người dùng.
