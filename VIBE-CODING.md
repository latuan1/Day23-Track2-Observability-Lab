# VIBE-CODING.md — Sổ Tay Vibe Coding cho Kỷ Nguyên Software 3.0

> Tech tips cho lập trình viên trong kỷ nguyên agentic. Distill từ tư tưởng của **Andrej Karpathy** (Software 3.0, Vibe Coding), **Boris Cherny** (Claude Code, Anthropic), và **Cat Wu** (Head of Product, Claude Code).
>
> Mục tiêu: một tài liệu standalone, có thể đem áp dụng vào bất kỳ codebase nào — từ một file `.tex` Beamer cho tới một service Kubernetes ở production.

---

## I. Tuyên Ngôn (The Manifesto)

### 1.1 — Cuộc dịch chuyển: Software 1.0 → 2.0 → 3.0

Karpathy chia lịch sử phần mềm làm ba thời đại:

| Thời đại | Vật liệu lập trình | Cách viết |
|---|---|---|
| **Software 1.0** | Mã nguồn (C, Python, Go, Rust...) | Con người gõ từng dòng |
| **Software 2.0** | Trọng số mạng thần kinh (neural weights) | Con người *huấn luyện* trên dữ liệu |
| **Software 3.0** | **Ngữ cảnh** (prompts, tools, context) | Con người *truyền đạt vibe* cho LLM |

Trong Software 3.0, **LLM là một hệ điều hành mới** — orchestrate compute và memory để giải quyết bài toán dựa trên prompt. Tiếng Anh trở thành ngôn ngữ lập trình "nóng" nhất hành tinh.

### 1.2 — "Coding is solved" — và những gì điều đó thực sự nghĩa là

Boris Cherny tuyên bố: **100% mã của Claude Code hiện được sinh bởi AI**. Bản thân ông đã không gõ tay một dòng code nào kể từ tháng 11 năm trước, nhưng vẫn ship 30–150 PR mỗi ngày — phần lớn từ điện thoại di động.

"Solved" ở đây **không** có nghĩa là "không cần con người". Nó có nghĩa là: việc *gõ cú pháp* đã trở thành phần rẻ nhất của quy trình. Phần đắt giá còn lại là **quyết định nên xây gì, và làm sao biết khi nào nó đã đúng**.

### 1.3 — Hai mặt của một đồng xu: *Vibe Coding* & *Agentic Engineering*

Karpathy phân biệt rạch ròi hai khái niệm — đừng nhầm lẫn:

- **Vibe Coding — *nâng nền* (raising the floor)**: bất kỳ ai (kể cả người chưa từng học code) cũng có thể tạo phần mềm, chỉ bằng cách mô tả ý tưởng và cảm nhận. Đây là sự *dân chủ hoá*.
- **Agentic Engineering — *nới trần* (extrapolating the ceiling)**: kỷ luật của kỹ sư chuyên nghiệp khi điều phối một *đội ngũ* tác nhân AI — vẫn giữ chuẩn bảo mật, chất lượng, khả năng mở rộng, nhưng tốc độ gấp 10×–100×.

Vibe Coding làm rộng đáy của kim tự tháp lập trình; Agentic Engineering kéo dài đỉnh của nó.

### 1.4 — Thanh trượt tự trị (The Autonomy Slider)

Mỗi task có một con trượt từ trái sang phải:

```
[Autocomplete] ─ [Inline edit] ─ [Multi-file edit] ─ [Background agent] ─ [Let it rip]
   con người                                                          con người chỉ
   lái chính                                                          duyệt kết quả
```

Vibe coder thành thạo là người **biết khi nào trượt phải, khi nào kéo trái**. Trượt phải khi rủi ro thấp, có test bảo vệ, scope nhỏ. Kéo trái khi đụng tới shared infra, security, hoặc khi bạn chưa hiểu vấn đề đủ sâu.

### 1.5 — Bài học máy in Gutenberg (1440)

Cherny so sánh AI với máy in Gutenberg. Trước máy in, *Sử gia* (scribe) dành cả đời chép sách thủ công. Khi máy in xuất hiện, họ **không** đau khổ — họ được giải phóng để tập trung vào phần *nghệ thuật*: minh hoạ, đóng bìa, thiết kế.

Lập trình viên bây giờ cũng vậy. AI giải phóng bạn khỏi boilerplate, khỏi việc tra `argparse` lần thứ 200. Phần còn lại — *kiến trúc, gu, phán đoán* — mới là phần đáng giá.

### 1.6 — "Builder" thay cho "Engineer"

Tại Anthropic, Cat Wu áp dụng chiến lược **Underfunding** (đầu tư thiếu một cách có chủ đích): khi một người phải làm việc của 10 người, họ buộc phải *Claudify* — tự động hoá mọi quy trình bằng AI.

Kết quả: ranh giới giữa Engineer / PM / Designer / Researcher mờ dần. Tất cả đều biết code, đều biết prompt, đều biết ship. Chức danh "Software Engineer" đang chết — thay bằng **"Builder"**: người có domain knowledge sâu và biết dùng AI như đôi tay vạn năng.

### 1.7 — Trí tuệ răng cưa (Jagged Intelligence) — sự thật phũ phàng

AI hiện tại có thể refactor 100.000 dòng code, tìm zero-day exploit, viết một service production trong 5 phút — *và đồng thời* khuyên bạn đi bộ 50 mét để rửa xe thay vì lái vào tiệm.

Nó không có *động lực nội tại*, không có *common sense* nhất quán. Nó là một **bóng ma** triệu hồi từ dữ liệu — siêu nhân ở chỗ này, ngớ ngẩn ở chỗ kia. Tin nó *có điều kiện*.

### 1.8 — Cạnh con người (The Human Edge): *Gu*, *Phán đoán*, *Street Smarts*

Karpathy đóng lại bằng một câu cô đọng:

> **"You can outsource execution, but you can't outsource understanding."**
> *"Bạn có thể thuê ngoài việc thực thi, nhưng không thể thuê ngoài sự thấu hiểu."*

Khi cú pháp rẻ về 0, **5% còn lại của con người** — gu thẩm mỹ, đạo đức, tư duy phản biện, sự nhạy bén thực tế — được khuếch đại lên hàng nghìn lần. Đó là tấm vé duy nhất giữ bạn không bị cuốn trôi.

---

## II. Tips — 20 nguyên tắc hành nghề

### Phần A · Mindset & Cách tiếp cận

#### Tip 1 — Ngữ cảnh là vua (*Context is king*)

Một câu prompt giàu ngữ cảnh đáng giá hơn mười câu lệnh chính xác. Hãy đính kèm file, dán log đầy đủ (không paraphrase), chỉ cụ thể commit hash, line number, version. Đừng viết *"có lỗi khi parse JSON"* — hãy viết *"file `src/parser.py` line 42, input là payload từ webhook ở 14:23:11 UTC, traceback đính kèm dưới"*.

> **Quy tắc đo:** nếu một junior dev đọc prompt của bạn mà không hỏi lại được câu nào, thì agent cũng sẽ không hỏi lại. Nếu junior cần hỏi — bổ sung trước khi gửi.

#### Tip 2 — Mô tả *ý định*, đừng mô tả *cú pháp*

"Add a `try/except` around the `requests.get()` call" là cú pháp. "Make the API call resilient to 5xx errors and rate limits, with exponential backoff capped at 30s" là ý định.

Mô tả ý định cho phép agent chọn pattern phù hợp (retry library, tenacity, urllib3 retry) thay vì sao chép mù snippet bạn nhớ mang máng từ Stack Overflow 2019.

#### Tip 3 — Coi tiếng Anh là ngôn ngữ lập trình chính

Trong Software 3.0, prompt **là** code — và nó rotted, leaked, refactored, tested, version-controlled như mọi codebase khác. Lưu prompt vào file (`.md`, `.txt`, hoặc skill files), commit chúng, review chúng trong PR.

> **Anti-pattern:** prompt sống trong clipboard và Slack DM. Khi đồng nghiệp hỏi *"bạn dùng prompt nào để sinh cái này?"* mà bạn không trả lời được — bạn vừa mất một phần codebase.

#### Tip 4 — Bắt đầu mỗi task bằng câu hỏi: *"Có cần code không?"*

Karpathy kể chuyện ông từng dày công xây ứng dụng MenuGen (nhận diện menu nhà hàng) — rồi nhận ra: chỉ cần đưa ảnh menu cho Gemini/Claude và nói *"overlay dữ liệu này lên menu"*, model làm trực tiếp **không cần app**.

Trước khi viết file đầu tiên, hỏi: liệu một LLM call đơn giản có giải quyết được không? Liệu một spreadsheet + một prompt có đủ không? Software 3.0 đang khiến rất nhiều phần mềm trở nên *thừa thãi* — đừng xây thêm cái thừa nữa.

---

### Phần B · Workflow & Cadence

#### Tip 5 — Trượt thanh tự trị có chủ đích

Quy ước cá nhân (chỉ là gợi ý — bạn nên có version riêng):

| Tình huống | Vị trí slider |
|---|---|
| Đụng vào file production một mình mình maintain | Tự động cao — let it rip |
| Đụng vào shared library nhiều team dùng | Trung bình — review từng file |
| Đụng vào auth, payments, secrets, migrations | Thấp — agent đề xuất, người phê duyệt từng diff |
| Khi *chưa hiểu* domain | Tắt agent — đọc code trước, hỏi sau |

#### Tip 6 — Loop nền (*Autonomous loops*)

Cherny mô tả một quy trình ở Anthropic: **agent loops** chạy liên tục — monitor CI, fix flaky tests, scrape user feedback từ GitHub Issues, đề xuất bug fix tự động. Bạn không *gọi* agent; agent *đến với bạn* qua PR.

Áp dụng nhỏ: trước khi đi ngủ, kích hoạt một background agent với scope rõ ràng (*"refactor module X theo style của module Y, mở PR với test"*). Sáng hôm sau bạn review thay vì viết.

#### Tip 7 — Ship 30 PR/ngày, mỗi PR *nhỏ* và *rõ ràng*

PR lớn = review-time tỉ lệ thuận với bình phương số dòng. Trong vibe coding, PR phải nhỏ tới mức bạn có thể review trên màn hình điện thoại 6 inch — vì đôi khi bạn *thực sự* sẽ review trên đó.

> **Heuristic:** một PR = một intent. Nếu bạn cần dùng từ "and" trong tiêu đề PR, hãy tách đôi.

#### Tip 8 — Mobile-first ops

Nếu workflow của bạn cần một laptop + tmux + 4 tab terminal, bạn đang khoá khả năng vận hành xuống một thiết bị duy nhất. Cherny ship 30–150 PR/ngày từ điện thoại — vì mọi thứ critical đều phải có mobile path: review, merge, rollback, kill switch.

Bài kiểm tra: nếu sếp bạn đang trên máy bay và một incident xảy ra — họ có thể xử lý từ điện thoại không? Nếu không, infrastructure của bạn còn nợ một bước thiết kế.

#### Tip 9 — Underfunding chiến thuật

Khi một task được giao đủ người, mọi người sẽ làm theo cách cũ. Khi được giao *thiếu* người (nhưng hợp lý), team buộc phải Claudify — tự động hoá để sống sót.

Áp dụng cá nhân: với task tiếp theo, đặt deadline ngắn hơn 50% so với *"thực tế cần"*. Ép bản thân hỏi *"chỗ nào AI có thể làm thay?"* trước khi bắt đầu.

---

### Phần C · Chất lượng & Kiểm chứng

#### Tip 10 — Jagged intelligence: *tin nhưng kiểm tra*

AI sai theo những kiểu *cụ thể*: `confident wrong > confused`. Nó hiếm khi nói *"tôi không biết"* — nó sẽ chế ra một API trông cực hợp lý mà không tồn tại. Đừng deploy code AI sinh ra mà chưa chạy nó **một lần** trong môi trường thật.

Heuristic phát hiện hallucination:
- API/method tên *quá hợp lý* nhưng bạn chưa từng nghe → grep docs trước khi tin.
- Magic numbers không có comment giải thích → hỏi lại.
- Code "sửa" một bug bằng cách *bao bọc* nó trong try/except trống → đó không phải fix, đó là che giấu.

#### Tip 11 — Test là *hợp đồng* bạn ký với agent

Trong vibe coding, test không phải là "phần phải viết sau". Test là *spec* — agent đọc test để hiểu bạn muốn gì.

> **Quy tắc:** prompt cho agent một task non-trivial → kèm test (đã viết hoặc đã skeleton). Agent sẽ implement đến khi test pass. Không có test = không có hợp đồng = bạn nhận về bất kỳ thứ gì compile được.

#### Tip 12 — Đọc "tầng dưới của tầng" (*the layer under the layer*)

CEO Anthropic cảnh báo: nếu bạn để AI sinh code mà không bao giờ đọc, bạn đang bị **deskill**. Sau 6 tháng, bạn sẽ mất khả năng tự debug, tự thiết kế, tự đánh giá AI có nói đúng không.

Quy tắc cá nhân: với mọi feature non-trivial, đọc *ít nhất* 1 file AI vừa sinh — line by line, hiểu *tại sao* nó chọn pattern đó. Bạn có thể không gõ code, nhưng bạn phải đọc được code.

#### Tip 13 — Đừng nhường security review cho AI

AI sinh code ở quy mô con người *trong một ngày*. Nếu nó introduce một SQL injection — nó introduce ở quy mô con người làm trong 6 tháng. Security review **vẫn** phải có người đọc, ít nhất ở các điểm:

- input parsing / deserialization
- auth / session / token handling
- file I/O với user-controlled paths
- subprocess / shell execution
- crypto, randomness, secrets

#### Tip 14 — Cảnh giác với "hallucinated APIs"

Pattern lặp lại: agent gọi `library.method()` không tồn tại, hoặc gọi đúng tên nhưng sai signature. Chuyện này xảy ra nhiều nhất với:

- Library mới (< 1 năm tuổi) — model chưa thấy đủ.
- Library vừa breaking-change major version — model học version cũ.
- Library hiếm hoặc internal company library — model đoán dựa trên tên.

Phòng vệ: ép agent dùng **MCP doc fetcher** (Context7, Mintlify) hoặc dán snippet docs vào context trước khi sinh code.

---

### Phần D · Anti-patterns cần tránh

#### Tip 15 — Đừng xây phần mềm thừa thãi

Trước khi tạo file, hỏi:
1. Liệu một prompt có giải quyết được không? (xem Tip 4)
2. Liệu một spreadsheet + một công thức có đủ không?
3. Liệu một bash script 10 dòng có thay được app 1000 dòng không?

Trong Software 3.0, **mọi phần mềm bạn xây đều cạnh tranh với phương án "không xây gì"**. Phần mềm cá nhân hoá (personalized micro-software) đang nuốt phần mềm "general-purpose" — đừng đi ngược dòng.

#### Tip 16 — Tránh deskilling

AI tốt giống như xe ô tô: tiện đến mức người ta quên đi bộ. Nếu bạn không bao giờ:

- viết một function từ đầu mà không gợi ý,
- debug một bug mà không paste vào AI,
- thiết kế một schema mà không hỏi AI có ý gì,

… thì sau 1 năm, bạn sẽ không còn là kỹ sư — bạn là người vận hành AI. Đó không phải tương lai an toàn.

> **Nguyên tắc 80/20:** 80% task — let AI rip. 20% task — tự làm hoàn toàn, không AI assist. Giữ cơ bắp lập trình sống.

#### Tip 17 — Đừng để agent chạm vào *shared state* không qua cổng

Shared state = production database, deployed config, billing, người dùng thật. Agent **luôn** phải qua một human-in-the-loop gate cho các action này. Cụ thể:

- DB migrations: agent đề xuất, người chạy.
- Production deploy: agent build artifact, người approve release.
- Force push, branch delete, rebase main: agent **không bao giờ** tự làm.
- Gửi email/Slack/notification ra ngoài: agent draft, người gửi.

Quy tắc một dòng: **reversible → agent tự lo. Irreversible → người phê duyệt.**

#### Tip 18 — Đừng "thuê ngoài *sự thấu hiểu*"

Bạn có thể nhờ agent viết code, viết test, viết doc, viết PR description. Bạn **không** thể nhờ agent:

- Hiểu *tại sao* product này tồn tại.
- Hiểu khách hàng *thực sự* đau ở đâu (không phải họ *nói* họ đau ở đâu).
- Quyết định trade-off khi 3 stakeholder muốn 3 thứ khác nhau.
- Gọi tên thứ *không nên* xây.

Đây là 5% còn lại. Giữ chặt nó.

---

### Phần E · Kỹ năng cần nuôi dưỡng

#### Tip 19 — Nuôi *Gu* (Product Taste)

Khi code rẻ về 0, *taste* — khả năng quyết định đáng xây gì và xây sao cho đẹp — là kỹ năng giá trị nhất. Cách rèn:

- Mỗi tuần, dùng một sản phẩm *ngoài* lĩnh vực của bạn (game, app design, công cụ tài chính). Hỏi: *"Tại sao họ chọn UX này, không phải UX kia?"*
- Mỗi PR review, hỏi: *"Đây có phải cách dễ nhất user đạt được intent của họ không?"* — không phải *"code có chạy không?"*.
- Đọc design docs / RFC của các công ty top (Stripe, Linear, Figma, Anthropic). Note xuống các quyết định bạn *bất ngờ*.

#### Tip 20 — Street smarts > IQ thuần

CEO Anthropic gợi ý: nghề nghiệp an toàn nhất tương lai là những nghề **heavily human-centered** — interpersonal relationships, vật lý thế giới thực, judgment dưới uncertainty.

Trong lập trình, "street smarts" là:
- Đọc Slack thread và đoán đúng *điều người ta không nói thẳng*.
- Biết khi nào nên gọi điện thay vì viết doc.
- Biết khi nào một bug "kỹ thuật" thực ra là một bug **organizational** (process, ownership, incentive).

AI không có cái này. Bạn có. Đừng để mất.

---

## III. Câu Hỏi Đóng

> *"When the cost of execution approaches zero, how large does the value of your ideas and your understanding become?"*
>
> **"Khi cái giá của việc thực thi tiến về bằng không, giá trị của ý tưởng và sự thấu hiểu của bạn sẽ lớn đến mức nào?"**
>
> — Andrej Karpathy

---

## Phụ lục — Đọc thêm

| Nguồn | Chủ đề |
|---|---|
| Andrej Karpathy — *Software 3.0* (talk, 2025) | Khái niệm Software 3.0, autonomy slider, jagged intelligence |
| Andrej Karpathy — *Vibe Coding* (X/Twitter post, 2025) | Định nghĩa gốc của thuật ngữ |
| Boris Cherny — *Building Claude Code* (Anthropic, 2025) | 100% AI-generated code, 30–150 PR/ngày, printing press analogy |
| Cat Wu — *Product Taste in the Agentic Era* (Anthropic, 2025) | Underfunding strategy, dissolving roles, taste as moat |
| Anthropic — *Co-work* (product, 2025) | Agentic principles ngoài engineering — knowledge work tự động hoá |
| [NotebookLM — Vibe Coding companion](https://notebooklm.google.com/notebook/55558a1b-4565-4622-9dfb-e05fe4b08f45) | Tổng hợp + audio overview của các nguồn ở trên — nghe trong lúc đi/chạy bộ |

---

*File này là tài liệu sống. Mỗi 3–6 tháng, review lại — vì paradigm này đang dịch chuyển nhanh hơn bất kỳ thứ gì trong lịch sử ngành.*
