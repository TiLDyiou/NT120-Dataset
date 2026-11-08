# Bộ quy tắc gán nhãn & quy trình làm việc — NT120 Dataset

> Tài liệu dành cho người/agent tham gia gán nhãn, duyệt nhãn và làm sạch dữ liệu bình luận mạng tiếng Việt của dự án NT120.
> Cập nhật: 2026-11-08. Mọi mục ghi **[ĐÃ CHỐT]** là quyết định của chủ dự án — bắt buộc tuân theo. Mục ghi **[QUY ƯỚC DỮ LIỆU]** là cách gán đa số trong train/valid, dùng khi quy tắc chưa nói rõ. Mục ghi **[CHƯA CHỐT]** thì **phải hỏi chủ dự án**, không tự quyết.

---

## Mục lục

1. [Bối cảnh & dữ liệu](#1-bối-cảnh--dữ-liệu)
2. [Nguyên tắc vàng](#2-nguyên-tắc-vàng)
3. [Nhãn Sentiment](#3-nhãn-sentiment)
4. [Nhãn Toxicity](#4-nhãn-toxicity)
5. [Từ khoá nhạy cảm & mức tối thiểu](#5-từ-khoá-nhạy-cảm--mức-tối-thiểu)
6. [Thuật toán quyết định nhãn (từng bước)](#6-thuật-toán-quyết-định-nhãn-từng-bước)
7. [Casebook — các ca đã chốt](#7-casebook--các-ca-đã-chốt)
8. [Các vùng chưa chốt — phải hỏi](#8-các-vùng-chưa-chốt--phải-hỏi)
9. [Quy trình làm việc](#9-quy-trình-làm-việc)
10. [Vệ sinh dữ liệu: trùng lặp & leak](#10-vệ-sinh-dữ-liệu-trùng-lặp--leak)
11. [Định dạng file & cách ghi an toàn](#11-định-dạng-file--cách-ghi-an-toàn)
12. [Những lỗi đã mắc — đừng lặp lại](#12-những-lỗi-đã-mắc--đừng-lặp-lại)
13. [Phụ lục: file báo cáo & backup hiện có](#13-phụ-lục-file-báo-cáo--backup-hiện-có)

---

## 1. Bối cảnh & dữ liệu

### 1.1. Các tập dữ liệu (thư mục `merged/`)

| File | Vai trò | Số dòng (2026-11-08) | Định dạng |
|---|---|---|---|
| `merged/train/train.csv` | Huấn luyện (bản chính thức, trước đây là `train_after.csv`) | 29.474 | UTF-8 **có BOM**, xuống dòng **LF** |
| `merged/valid/valid.csv` | Kiểm định | 7.756 | UTF-8 không BOM, **CRLF** |
| `merged/challenge/challenge.csv` | **Benchmark** — quan trọng nhất | 3.237 | UTF-8 không BOM, **CRLF** |

Cột: `Comment,Sentiment,Toxicity`. Một comment có thể chứa xuống dòng (được quote theo chuẩn CSV) — **số dòng vật lý ≠ số bản ghi**.

### 1.2. Nguồn comment (để hiểu ngữ cảnh)

- Video/stream giải trí: **Thầy Giáo Ba**, **Abe**, **Sela**, **Độ Mixi** ("anh Độ", "tộc trưởng"), Trần Dần ("quốc sư", "nhà tiên tri vũ trụ"). Fan trêu nhau, tag tên bạn bè rất phổ biến.
- Tin tức (VnExpress…): tai nạn, bạo hành trẻ em, tội phạm, dịch bệnh, xe cộ, bóng đá.
- Chính trị/xã hội: tranh cãi "dlv", "3 que", "bò đỏ", Trung Quốc, vùng miền.

### 1.3. Ràng buộc bất biến

- Không có comment nào xuất hiện ở hơn một tập (xem §10).
- Không có comment trùng trong cùng một tập.
- Nhãn hợp lệ: Sentiment ∈ {-1, 0, 1}, Toxicity ∈ {0, 1, 2, 3}.

---

## 2. Nguyên tắc vàng

1. **Đọc trọn câu trong ngữ cảnh.** Từ khoá chỉ là gợi ý mặc định; giọng điệu tổng thể có quyền ghi đè. Không bao giờ gán nhãn hàng loạt theo nhóm/từ khoá mà không đọc lại từng câu. **[ĐÃ CHỐT]**
2. **Mỉa mai → Sentiment = -1**, kể cả khi bề mặt câu trung tính hoặc tích cực. **[ĐÃ CHỐT]**
3. **Nhãn hiện có là do người gán.** Khi duyệt, chỉ đề xuất đổi khi nhãn **sai rõ** theo tài liệu này; không đổi chỉ vì "cũng có thể khác".
4. **Không chắc thì hỏi.** Ghi vào danh sách `hỏi`, kèm phương án bạn nghiêng về và lý do.
5. **Không sửa dữ liệu khi chưa backup**, và mọi thay đổi phải có log (§9.4).
6. **Challenge là benchmark** — ưu tiên độ chính xác ở đây hơn mọi tập khác.
7. **Đánh giá linh hoạt dựa trên mức độ công kích thực tế.** Không hoàn toàn dựa vào từ khoá hay quy tắc cứng máy móc; luôn phân tích ngữ cảnh, đối tượng bị nhắm tới và mức độ sát thương để gán nhãn chính xác. **[ĐÃ CHỐT]**

---

## 3. Nhãn Sentiment

### 3.1. Định nghĩa gốc **[ĐÃ CHỐT]**

| Nhãn | Ý nghĩa |
|---|---|
| **-1** Tiêu cực | Chê bai, bức xúc, thất vọng, phàn nàn, mỉa mai cay độc, lên án. |
| **0** Trung tính | Đưa tin tức, trích dẫn khách quan, hỏi han thuần túy, câu cụt lủn không rõ sắc thái. |
| **1** Tích cực | Khen ngợi, cảm kích, yêu thích, ủng hộ, chúc mừng, vui vẻ. |

### 3.2. Quy tắc chi tiết

| Tình huống | Sentiment | Nguồn |
|---|---|---|
| Có ý **mỉa mai/châm chọc**: giả vờ hỏi, khen/tâng bốc giả, chào mừng giả, "=))", ":v", "😂" đặt sau một câu châm chọc | **-1** | [ĐÃ CHỐT] |
| Lên án tội phạm, kẻ bạo hành | **-1** | [ĐÃ CHỐT] |
| Câu cụt chỉ có emoticon / không rõ sắc thái (":)))))", "Nam mô :)))") | **0** (và T=0) | [ĐÃ CHỐT] |
| Chê sự vật/sự việc (không dung tục) | **-1** | [ĐÃ CHỐT] |
| Chê con người nhưng lịch sự/góp ý | **-1** | [ĐÃ CHỐT] |
| **Thuần thương xót, cảm thông** với nạn nhân/người gặp nạn: "tội nghiệp bé quá", "thương con quá", "đau lòng quá", "xót xa", "thương tâm", "chia buồn cùng gia đình", "cầu mong các con siêu thoát" | **1** | [ĐÃ CHỐT] — train/valid hiện đa số gán -1, cần sửa thành 1 |
| Thương xót **kèm đề xuất/kiến nghị** (vd: "nên thành lập một tổ chức để những người có bỏ con có chỗ mà cho chứ như vầy quài đau lòng chết", "cần rà soát lại...") | **0** | [ĐÃ CHỐT] |
| Thương xót **kèm truy cứu trách nhiệm** ("Thật thương tâm. Trách nhiệm phải làm rõ đến cùng từ cấp quản lý đến đơn vị thi công...", "các cơ quan chức năng cần vào cuộc...") | **0 (T=0)** | [ĐÃ CHỐT] |
| Thương xót **kèm lên án tội phạm/kẻ thủ ác** ("cha mẹ nào mà ác quá, tội nghiệp con", "tội nghiệp con... tên súc vật") | **-1** (T=1 theo quy tắc lên án tội phạm) | [ĐÃ CHỐT] |
| "Tội nghiệp/thương" dùng để **mỉa** ("tội nghiệp con nhỏ, khùng lâu năm", "thương anh quá, ngu đoé gì mà ngu thế") | **-1** | quy tắc mỉa mai |
| "Xót" vì của cải/sự việc ("xót xa khi phải vứt hộp bánh") | **-1** (phàn nàn) | — |
| "Thương [ai đó]" mang nghĩa quý mến ("thương a ba :(") | **1** | đã gán trong challenge |
| Chúc (bình an, cố lên, chúc mừng, sớm khoẻ) | **1** | [QUY ƯỚC DỮ LIỆU] |
| Khẩu hiệu fan Thầy Giáo Ba "Cực kì thuyết phục" | **1** (T=0) trừ khi rõ là mỉa | [QUY ƯỚC DỮ LIỆU] — 124/132 dòng train |
| Câu có từ tục nhưng ý khen/vui ("đẹp trai vcl", "đcm cười 33 giây") | **1** (T theo §4) | [ĐÃ CHỐT] |

### 3.3. Dấu hiệu nhận biết mỉa mai

- Câu hỏi không cần trả lời, hỏi để hạ thấp: *"Dư Luận Viên là gì ?? … Họ là người bảo vệ chất độc hã ???"* → (-1,0).
- Danh xưng tâng bốc phóng đại: *"Nhà tiên tri vũ trụ, đại quốc sư Ba sọc quốc lưu vong Trần Dần😂"* → (-1,0).
- Khen cường điệu kèm ":v": *"Giọng ca của B52 phải vượt qua tầm vũ trụ :v"* → (-1,0).
- Giả vờ không biết: *"3/// là nước gì lạ vậy"* → (-1,0).
- Đùa giễu bằng doạ báo: *"Tao báo dlv 😂😂😂"*, *"tao méc mấy a dlv tụi bây phản động nè"* → (-1,0).
- "Thiên đường", "vĩ đại", "dân biết dân bàn", "nhờ ơn đảng" dùng ngược nghĩa → -1.

**Ngược lại** — câu nhìn như chê nhưng thật ra khen/trìu mến (vd. *"dân nam kỳ nói chuyện là cưng hà, chửi cũng cưng"* → (1,0)): đọc kỹ, đừng gán -1 máy móc.

---

## 4. Nhãn Toxicity

### 4.1. Định nghĩa gốc **[ĐÃ CHỐT]**

| Nhãn | Ý nghĩa |
|---|---|
| **0** Không độc hại | Lịch sự, tranh luận văn minh; chê bai sự việc nhưng không xúc phạm cá nhân. |
| **1** Thiếu lịch sự | Chửi thề buột miệng, từ đệm cảm thán (vcl, cay vl…), cộc cằn nhưng chưa thóa mạ ai. |
| **2** Xúc phạm / công kích cá nhân | Chửi thẳng mặt người khác, body shaming, miệt thị trí tuệ/nhân phẩm (ngu, óc chó, mặt dày…). |
| **3** Thù ghét cực đoan / đe doạ | Phân biệt vùng miền, tôn giáo, sắc tộc; đe doạ bạo lực (bắn, chém, giết); trù ẻo cái chết. |

Bổ sung **[ĐÃ CHỐT]**:
- Bình phẩm/chê **sự vật, sự việc** (không phải con người) mà không dung tục → T=0, S=-1.
- Nhắm vào **con người** nhưng lịch sự/góp ý → T=0, S=-1.
- Công kích con người **không chửi thề nhưng có sát thương** → **T=1**.
- Cẩn thận: comment trêu đùa, lên án tội phạm, và câu tưởng chê nhưng thật ra khen.

### 4.2. Từ tục: viết đầy đủ vs viết tắt **[ĐÃ CHỐT]**

Đây là điểm phân biệt T=1 và T=2 hay bị sai nhất.

| Dạng | Ví dụ | Toxicity |
|---|---|---|
| **Viết đầy đủ** | lồn, cặc, buồi, địt, đụ má, "cái máu lồn", "con cặc gì", "xạo lồn" | **2** — câu nào từ tục được viết đầy đủ thì gán T=2 [ĐÃ CHỐT] |
| **Viết tắt / né chữ** | cc, đm, dm, đmm, đcm, dcm, clm, cmm, cmn, vl, vcl, vkl, vc, đ (=đéo), "L" thay lồn, lol, loz | Xem xét kĩ mức độ công kích: cảm thán/vui vẻ/chê nhẹ (như "anh lại xạo loz") → 1; có công kích/miệt thị nặng → 2; ca lưng chừng không chắc chắn → gom lại để chủ dự án tự gán [ĐÃ CHỐT] |
| **Từ đệm cảm thán nhẹ** | "vãi" đứng một mình làm từ đệm cảm thán thuần túy ("hay vãi", "đẹp vãi") | **0** — chỉ đệm "vãi" thì không tính toxic [ĐÃ CHỐT] |
| Từ đệm trong câu khen/vui | "đẹp trai vcl", "gắt vl", "hợp lí vl", "đcm cười 33 giây" | **1** (S theo ý câu) |
| Cảm thán không nhắm ai | "Nứng", "chết mẹ", "Ây long bùi => lông buồi" | **1** |

Ví dụ đã chốt:
- *"vĩ đại cái máu lồn =)))"* → **(-1,2)** — vì "lồn" viết thẳng.
- *"Cc thầy eie"* → **(-1,1)**; *"kệ mẹ thầy"* → **(-1,1)** — viết tắt/cộc cằn, giữ 1.
- *"Bê L chạy quanh nhà :))"* → **(1,1)**; *"… bê l chạy xung quanh 🙂"* → T=1.
- *"3/// zui, khác vs 3/// lol kia"* → **(1,1)** — "lol" là từ tục viết tắt, rành rành.

Lưu ý: từ tục viết đầy đủ nhưng chỉ cảm thán, không nhắm ai (vd. "Vãi thị lồn" đứng một mình) → xem §8.

### 4.3. Từ suồng sã KHÔNG phải từ tục **[ĐÃ CHỐT]**

"thằng", "tao", "mày", "cưng", "con này", "bọn", "tụi bây", "mu" (trong ngữ cảnh kể chuyện) **tự chúng không** đủ để T=1.
- *"Dư Luận Viên là gì ?? Mấy thằng này là ai ?…"* → T=0 (chủ dự án sửa từ 1 về 0).
- *"Hà Nguyễn Khánh Duy Gái Nhật đi tàu… ngứa mu quá… Nhìn ghê lắm kìa"* → **(-1,0)**.

### 4.4. Công kích cá nhân (T=2)

- Miệt thị trí tuệ: **chửi người khác là ngu** ("thằng ngu", "ad đúng là ngu", "ngu gia truyền", "ngu bền vững"), óc chó (ochco), não ngắn, không có não, thiểu năng, dốt nát → **2** [ĐÃ CHỐT]. (Lưu ý: tự nhận/hỏi thăm vô hại như "cho hỏi ngu chút" không áp).
- Chửi thẳng: "thằng ngu", "mấy thằng ngu tin đảng", "mang cái mặt của mày", "bớt cái mồn lại … bà già" → **2**.
- Ví người với con vật để hạ nhục: "con bò động đực", "lũ chó đói", "dlv sủa" → **2** (hoặc 3 nếu là từ ở §5 có mức 3).
- **Ngoại hình — ranh giới [ĐÃ CHỐT]:**
  - Nhận xét có thể đúng sự thật hoặc góp ý ("Mũi lệch cô ơi", "Cạo râu đi chị ba", "chẳng đẹp, chẳng sang") → **0**.
  - Bình phẩm nhẹ có sát thương ("Mắt ông này già trước tướng") → **1**.
  - **Chế giễu**, tỏ ra ghê tởm, nghi ngờ giới tính ("râu ria đầy mồm", "Nhìn con này thấy gê quá", "đàn bà thiệt hay đàn ông vậy chời?", "xấu như con quỷ") → **2**.

### 4.5. Lên án tội phạm **[ĐÃ CHỐT]**

Câu lên án **nhắm vào người** với: "súc vật", "khốn nạn", "không phải (là) con người", "quỷ dữ chứ không phải con người", "đồ khốn nạn", "cầm thú" → **T=1, S=-1** [ĐÃ CHỐT] (chủ dự án hạ mức toxic thành 1 để phân biệt với hành vi lăng mạ/công kích vô cớ).
- *"…cho đến khi tên súc vật này vô tù"* → (-1,1).
- *"Đồ khốn nạn, không phải con người…"* → (-1,1).

**Lưu ý câu chửi cụt lủn [ĐÃ CHỐT]:** Các từ như *"Súc vật"*, *"Khốn nạn"* đứng một mình quá cụt lủn không rõ ngữ cảnh $ightarrow$ gán **T=2** vì có khả năng cao là công kích cá nhân trực diện (khác với câu kể/lên án hành vi cụ thể $ightarrow$ T=1).

**Không áp** khi đối tượng là xã hội/tổ chức/sự việc hoặc nghĩa đen:
- *"xã hội khốn nạn vl"* → (-1,1); *"cái dn khốn nạn…"* (doanh nghiệp) → (-1,1).
- *"Cuộc sống này khốn nạn hơn mình tưởng"*, *"tận cùng của sự khốn nạn"* (sự việc) → giữ T=0.
- *"xe chở súc vật"* (nghĩa đen), *"giọng cười Abe nghe khốn nạn vl"* (đùa) → không áp.

### 4.6. Đe doạ & trù ẻo (T=3)

- Đe doạ bạo lực: "quất chết mẹ nó", "chặt… tạt axit", "chở bom với mìn cho cả nhà bà nổ", "đấm vào mỏm cmm giờ" → **3**.
- Trù ẻo cái chết/tiệt chủng: "lũ này chết đi cho đỡ chật đất", "sao nó k chết thay ng khác", "bọn 3/// liệu có bị tiệt chủng không", "3/// mà chết hết" → **3**.
- Chửi nặng nhưng **không** đe doạ/không phân biệt: "Lũ chó đói", "Kon mẹ nhà nước lũ hút máu dân" → **2**, không phải 3.
- Kêu gọi tử hình trong ngữ cảnh pháp lý ("nên tử hình kẻ hiếp dâm") — xem §8.

---

## 5. Từ khoá nhạy cảm & mức tối thiểu

### 5.1. Mức Toxicity tối thiểu **[ĐÃ CHỐT]**

Áp dụng khi từ **được dùng để miệt thị**. Đây là **mức sàn**: nếu câu có thêm chửi/đe doạ thì nâng cao hơn; không bao giờ hạ dưới mức này **khi từ được dùng để miệt thị**.

| Nhóm | Từ | Toxicity tối thiểu |
|---|---|---|
| Trung Quốc | khựa, tàu khựa, tàu cộng | **1** |
| | trung cẩu | **2** |
| | chệt | **không phải từ miệt thị** |
| Sắc tộc khác | mọi rợ, bọn mọi | **2** |
| | mẽo, thái lọ, mỹ đen | **1** |
| Chính trị | bò đỏ | **3** |
| | dlv, dư luận viên, dư lợn viên (gọi ai đó là dlv) | **3** |
| | đu càng | **2** |
| | cơm sườn, nhà cơm sườn (mỉa mai cộng sản/chế độ) | **3** |
| | 3 que / ba que / 3/// | theo ngữ cảnh — §5.2 |
| | ngụy, việt cộng | [CHƯA CHỐT] — §8 |
| Vùng miền | bắc kỳ/kì, bake, bắc cụ, nam kỳ/kì, parky, packy, trung kỳ | **3** |

### 5.2. "3 que / ba que / 3/// / ba sọc" — theo ngữ cảnh **[ĐÃ CHỐT]**

| Ngữ cảnh | Nhãn (S,T) |
|---|---|
| Nhắc trung tính, hỏi nghĩa, nói về lá cờ | (0,0) |
| Đùa theo meme Trần Dần, không ác ý | (0,0) |
| Gọi người/nhóm là 3 que để chê, mỉa mai, không tục | (-1,0) |
| Kèm chửi/miệt thị nặng (chó, sủa, ngu, lợn, ăn cứt, thất học) | (-1,1) hoặc (-1,2) — chọn theo mức công kích |
| Kèm trù chết, tiệt chủng, mong bị dịch | (-1,2) hoặc (-1,3) — nói thẳng thì 3, ngụ ý thì 2 |
| Thành ngữ "ba que xỏ lá", "ba que ba xạo" (gian xảo) | (-1,1) nếu công kích người, (-1,0) nếu chê sự việc |
| Nghĩa đen ("con rắn ba sọc") | (0,0) |

**Ghi đè theo ngữ cảnh** (chủ dự án đã sửa):
- Câu nhắc "3///" nhưng tổng thể là khen → (1,0). Vd. *"ông có tuổi rùi! cố giữ gìn sức khỏe… giác ngộ vầy là tốt rùi, chứ nhìu ông già 3/// cố chấp lắm"* → **(1,0)**.
- Câu "đùa" nhưng có từ tục → T theo từ tục. Vd. *"Ông Dần là 3/// nhưng là 3/// zui, khác vs 3/// lol kia"* → **(1,1)**.
- Câu có thêm từ ở §5.1 (vd. "bò đỏ") → giữ mức sàn của từ đó.

### 5.3. Khi KHÔNG áp mức sàn

Từ chỉ được **nhắc tới**, không dùng để miệt thị ai:
- Hỏi nghĩa: *"Dư Luận Viên là gì ??"*
- Tự nhận để châm biếm: *"…từ hôm nay mình đã gia nhập đội ngũ DLV…"* → (-1,0).
- Bênh vực/phản đối kỳ thị: *"Bắc kỳ cũng là người vn mà nó chửi như có sách…"* → (-1,1).
- Kể lại lời người khác: *"Mở mồm ra chửi Bắc kỳ rồi lại bị mấy anh Bắc tìm thôi"* → (-1,1).
- Tự nhận vùng miền: *"Tui cũng dân bắc Kỳ mà thấy thằng dân bắc này… khốn nạn quá"* → (-1,2) (2 do "khốn nạn", không phải do "bắc Kỳ").
- Nghĩa không miệt thị: *"anh trai Mỹ đen"* (muốn có bạn trai) → (1,0); *"dân nam kỳ… cưng hà"* → (1,0).
- Từ trùng chữ: "triền miên/thôi miên" (≠ miên), "da đen" (rám nắng), "chệch", "ngụy biện/ngụy trang" (≠ ngụy).

---

## 6. Thuật toán quyết định nhãn (từng bước)

Làm lần lượt cho **mỗi** comment:

1. **Đọc trọn câu**, xác định: ai/cái gì là đối tượng? người nói đang khen, chê, hỏi, kể, đùa hay mỉa?
2. **Sentiment**
   1. Có mỉa mai/châm chọc? → **-1**, sang bước 3.
   2. Lên án / chê / phàn nàn? → **-1**.
   3. Khen / chúc / ủng hộ / vui vẻ / quý mến / **thương xót, cảm thông** (không kèm lên án)? → **1**.
   4. Đưa tin, hỏi thuần, câu cụt không rõ sắc thái → **0**.
3. **Toxicity** — lấy mức **cao nhất** thoả mãn:
   1. Đe doạ bạo lực, trù ẻo chết, hoặc từ có mức sàn 3 dùng để miệt thị → **3**.
   2. Từ tục **viết đầy đủ** trong câu mỉa/công kích; chửi thẳng người; miệt thị trí tuệ/nhân phẩm; body shaming rõ; lên án bằng "súc vật/khốn nạn/không phải con người"; từ có mức sàn 2 → **2**.
   3. Từ tục **viết tắt**/từ đệm; cảm thán tục không nhắm ai; công kích không chửi nhưng có sát thương; từ có mức sàn 1 → **1**.
   4. Còn lại → **0**. (Từ suồng sã "thằng/tao/mày" không tính.)
4. **Kiểm tra chéo**: nhãn có mâu thuẫn với casebook (§7) không? Câu có thuộc vùng chưa chốt (§8) không? → nếu có, đánh dấu `hỏi`.
5. **Sanity check**: S=1 & T≥2 là hợp lệ nhưng hiếm (khen kèm chửi tục) — đọc lại lần nữa. S=0 & T≥2 thường là sai Sentiment.

---

## 7. Casebook — các ca đã chốt

| Comment (rút gọn) | Nhãn | Ghi chú |
|---|---|---|
| vĩ đại cái máu lồn =))) | -1,2 | từ tục viết đầy đủ |
| Cc thầy eie | -1,1 | viết tắt |
| kệ mẹ thầy | -1,1 | cộc cằn |
| đcm cười 33 giây | 1,1 | vui + chửi cảm thán |
| Đẹp trai tài năng… thầy dell có nha… | 1,1 | "dell" = đéo |
| Biết bao nhiêu vụ… đm… chửi sml ra à ? | -1,1 | bức xúc, chửi không nhắm ai |
| abe ochco =))) | -1,2 | "óc chó" nhắm người |
| … abe tuổi lol mà so đc | -1,2 | miệt thị cá nhân |
| Võ Công Hoàng cái lon má | -1,2 | chửi tục vào người được tag |
| Mắt ông này già trước tướng | -1,1 | ngoại hình, có sát thương |
| :))))) | 0,0 | câu cụt |
| Nam mô :))) | 0,0 | câu cụt |
| 😪😪 | -1,0 | emoji buồn |
| Nứng | 0,1 | tục buột miệng |
| Nhà vô địch | 1,0 | khen |
| thương a ba :( | 1,0 | quý mến |
| Tội nghiệp bé quá! / Đau lòng quá đi. / Quá đau xót! Chia buồn cùng gia đình! | 1,0 | thương xót |
| Thật thương tâm. Trách nhiệm phải làm rõ đến cùng… Xin được chia buồn | -1,0 | thương xót **kèm** chỉ trích → -1 |
| …cha mẹ nào mà ác quá, tội nghiệp con, chúc con khỏe mạnh | -1,0 | thương xót kèm lên án → -1 |
| 2 chị em chắc mất hết nhân cách con người. Thương các cháu… | -1,0 | "mất nhân cách/nhân tính" **không** tính như "không phải con người" → T=0 |
| xin gởi lời chia buồn…, anh chàng rể này quá mất nhân tính, đê hèn… | -1,0 | như trên |
| Xã tắc mang dâng cho ác quỷ… Đê hèn như một lũ điên | -1,0 | thơ chính trị, không chửi tục → T=0 |
| Lần này nhìn cô ấy, rất bùn. | 0,0 | không rõ sắc thái |
| Tk trọng tài m ráng núp kính đi nhe 🤣 t bắn á | -1,0 | doạ đùa → **không** phải đe doạ (T=0) |
| Cấm tụi bây đụng vô cô minh hiếu nhé… fan nem🔪nhé mầy | -1,1 | doạ kiểu fan → T=1, không phải 3 |
| Tôi gọi là Chínese virus | -1,0 | không tính miệt thị |
| Yêu Bác, yêu Nước, yêu Đảng… yêu cả hội con dân Đông Lào♡ | 1,0 | "Đông Lào" dùng trìu mến |
| …lãnh đạo đông lào thì lại muốn gắn cả việt nam vào dưới đít trung qq !!! | -1,3 | "đông lào/trung qq" dùng miệt thị |
| Quang Quang Tường mấy lol đó mà. Lạc hậu vãi lol | -1,2 | "lol" dùng làm **danh từ gọi người** → 2 (khác "lol" làm từ đệm = 1) |
| Nhìn mặt xanh xao mà thương bác thật sự … nhắc đến lại DM NHUNG | -1,2 | chửi viết tắt nhưng **nhắm thẳng người có tên** → 2 |
| Anh độ chửi con nhân vật ngu mọi người ạ. Trong khi anh ấy… hiểu ý tôi chứ | -1,2 | ngầm chê người là ngu |
| Bởi dừa lờ t lắm cho chừa cái tật… :)))))) | -1,1 | "lờ" = L viết tắt |
| soái ca trong ngôn lù đây sao | -1,1 | "ngôn lù" chơi chữ tục |
| Chi ba chui cho hon cho con no nhuc | 0,1 | cộc cằn |
| Mũi lệch cô ơi / Cô này chẳng đẹp, chẳng sang… / Cạo râu đi chị ba | -1,0 | nhận xét ngoại hình **có thể đúng sự thật** ("lỡ mũi lệch thật thì sao?"), không tục → T=0 [ĐÃ CHỐT] |
| Cô ba gì đâu mà râu ria đầy mồm | -1,2 | **chế giễu** ngoại hình → body shaming (khác "Cạo râu đi chị ba" = góp ý/nhận xét → 0) [ĐÃ CHỐT] |
| Chị ba này là đàn bà thiệt hay đàn ông vậy chời? Hông hỏi thiệt | -1,2 | chế giễu giới tính/ngoại hình → 2 [ĐÃ CHỐT] |
| Nhìn con này thấy gê quá | -1,2 | tỏ ra ghê tởm một người → 2 [ĐÃ CHỐT] |
| …hiểu biết nông cạn ghê | -1,1 | chê hiểu biết, có sát thương → 1 [ĐÃ CHỐT] |
| Hay vai lon anh à | 1,1 | "vãi lồn" làm từ đệm trong câu khen → T=1 [ĐÃ CHỐT] |
| V im lặng đi. Than than cái j | -1,0 | bảo người khác im, không tục → **không hề** toxic [ĐÃ CHỐT] |
| Dug la con lih cho do lam tu cach kho ua va | -1,0 | chê người, không tục → T=0 |
| Mỗi đợt iphone ra… (có "méo") | 1,0 | "méo" thay "đéo" **không** tính toxic |
| Đc ván hay ông giáo gáy khét vc=)) 🥰 | 1,1 | khen + từ đệm |
| Bê L chạy quanh nhà :)) | 1,1 | "L" viết tắt |
| Cực kỳ thuyết phục🙂 | 1,0 | khẩu hiệu fan |
| Cầu mong mọi người bình an! Việt Nam cố lên! | 1,0 | chúc |
| oki thầy / Cây nè đập hộp đi thầy | 0,0 | đáp lời / đề nghị |
| Thật là một thiếu sót | -1,0 | phàn nàn lịch sự |
| …tên súc vật này vô tù | -1,2 | lên án tội phạm |
| Đồ khốn nạn, không phải con người… | -1,2 | lên án tội phạm |
| Gái Nhật đi tàu… ngứa mu… Nhìn ghê lắm kìa | -1,0 | không có từ tục |
| ông có tuổi rùi!… nhìu ông già 3/// cố chấp lắm | 1,0 | tổng thể là khen |
| Ông Dần là 3/// nhưng là 3/// zui, khác vs 3/// lol kia | 1,1 | có "lol" |
| Dư Luận Viên là gì ?? Mấy thằng này là ai ?… | -1,0 | mỉa mai, "thằng" không tính |
| Tao báo dlv 😂😂😂 | -1,0 | giễu cợt |
| dân nam kỳ nói chuyện là cưng hà, chửi cũng cưng | 1,0 | quý mến |
| Lũ chó đói | -1,2 | xúc phạm, không đe doạ |
| lũ này chết đi cho đỡ chật đất | -1,3 | trù ẻo |
| Có những thằng Bắc Cụ đang dùng cái óc toàn Bắc… | -1,3 | vùng miền |
| anh lại xạo loz | -1,1 | trêu, viết tắt "loz" [ĐÃ CHỐT] |
| Súc vật (đứng một mình cụt lủn) | -1,2 | cụt lủn, nghi công kích cá nhân [ĐÃ CHỐT] |
| Công thức đập của thầy ba hayy vãi ae à | 1,0 | chỉ đệm "vãi" → T=0 [ĐÃ CHỐT] |
| Mày có loz hả | -1,2 | công kích bằng từ tục viết trại [ĐÃ CHỐT] |
| Bảo yêu nước lắm yêu nước vừa toàn làm mấy kiểu... sự ngu : ) | -1,2 | mỉa mai + chê ngu [ĐÃ CHỐT] |
| Ad méo biết thì thôi. Tầm nhìn nhà cơm sườn... | -1,3 | "nhà cơm sườn" mỉa mai chính trị [ĐÃ CHỐT] |
| Cường học ANND là hiểu rồi. dm đám côn đồ tụi mày là tay sai... | -1,3 | công kích chính trị/côn đồ tay sai [ĐÃ CHỐT] |
| dm cứ đi vay ODA của TQ vs NHật... mấy thằng bụng bà đẻ... | -1,3 | chửi bới chính trị nặng [ĐÃ CHỐT] |

---

## 8. Các vùng chưa chốt — phải hỏi

Gặp các ca sau thì **ghi `hỏi`**, đề xuất phương án nghiêng về kèm lý do, **không tự áp**:

| Vùng | Ví dụ | Câu hỏi mở |
|---|---|---|
| Từ tục viết đầy đủ nhưng chỉ cảm thán, không nhắm ai | "Vãi thị lồn", "Vãi cả lồn" | 1 hay 2? |
| Từ tục né chữ/biến thể | "xamlone", "l.o.n", "lồnn", "vlone", "đel", "đume" | đang áp: biến thể của từ viết tắt → 1, biến thể của từ đầy đủ ("lồnn") → 2. Riêng "méo" → 0 [ĐÃ CHỐT] |
| Xúc phạm không chửi thề (trí tuệ, tâm thần, ngoại hình, nhân phẩm) | "con điên", "thứ thần kinh", "ngáo đá", "không có não", "xấu như con quỷ", "cave", "miệng thúi hơn hầm cầu", "trẻ trâu", "cái hạng này" | Đã chốt: nhận xét có thể đúng sự thật, chê nhẹ, bảo im → 0. Phần còn lại Claude **tạm** gán theo nguyên tắc: khinh miệt, gọi tên hạ thấp → 1; miệt thị trí tuệ/tâm thần/nhân phẩm, ví với con vật, body shaming nặng → 2. Chờ chủ dự án xác nhận (`challenge_round2_applied.csv`) |
| Viết không dấu, khó đọc nghĩa | "…con no đên cho may 1 trân ko cân quy lay đâu nha" | chủ dự án để nguyên nhãn gốc khi không chắc nghĩa |
| Trêu bạn bè được tag | "2 con điên", "… chó" khi tag bạn | đùa (0/1) hay xúc phạm (2)? |
| Chế giễu LGBT | "bê đê", "Lam giong bede", "bê đê chúa", "bóng lộ" | **T=2** [ĐÃ CHỐT] (nhắc trung tính như "Tim Cook là gay" → 0) |
| Chính trị chưa có mức | "ngụy", "việt cộng", "bò" (đứng một mình) | mức sàn? ("Chinese virus" → 0, "đông lào/trung qq" theo ngữ cảnh [ĐÃ CHỐT]) |
| Lên án tội phạm bằng "quỷ", "cầm thú", "Cẩu Tặc", "côn đồ, hèn hạ" | "2 con quỷ này", "Cầm thú cũng không ác độc bằng hai chị em mụ này" | đang giữ T như cũ (theo tiền lệ "mất nhân tính, đê hèn" → 0); chưa chốt |
| Kêu gọi tử hình | "tử hình mới vừa lòng", "Loại này tử hình luôn" | Còn tuỳ, xét thật kĩ ngữ cảnh vào, nghiêng về **T=1** [ĐÃ CHỐT] (thảo luận pháp lý thuần túy → 0; đe dọa bạo lực cực đoan/chửi bới → 2 hoặc 3) |
| Mỉa mai hay khen thật về Trần Dần | "3 que, diễn viên hài Trần Dần muôn năm hú hú🤩" | -1 hay 1? |
| Trù ẻo nhẹ bằng thành ngữ | "đời cha ăn mặn đời con khát nước" | 2 hay 3? |

Khi chủ dự án trả lời, **cập nhật mục này** (chuyển ca sang §4/§5/§7 với nhãn [ĐÃ CHỐT]).

---

## 9. Quy trình làm việc

### 9.1. Nguyên tắc chung

- **Đề xuất trước, sửa sau.** Agent duyệt nhãn **không ghi thẳng vào file dữ liệu**; chỉ xuất file đề xuất.
- **Backup trước mỗi lần sửa**: `cp merged/<tập>/<tập>.csv merged/_backup/<tập>_before_<tên_bước>.csv`.
- **Mọi thay đổi có log** trong `merged/_report/` (§9.4).
- Không commit/push khi chưa được yêu cầu.

### 9.2. Duyệt nhãn (review)

1. **Chuẩn bị chunk**: xuất tập cần duyệt ra TSV `dòng, Sentiment, Toxicity, đã_duyệt, Comment` (thay xuống dòng trong comment bằng `⏎`, tab bằng dấu cách). `dòng` = số thứ tự bản ghi tính cả header là dòng 1 (tức `index_trong_list_đọc_bằng_csv.reader + 1`). Đánh `đã_duyệt=1` cho các dòng đã được chủ dự án/đợt trước duyệt → bỏ qua.
2. **Đọc toàn bộ chunk** — không lấy mẫu, không dùng script từ khoá để quyết nhãn. Script chỉ dùng để đọc/ghi file.
3. **Ghi đề xuất** ra TSV, header:
   ```
   dòng	S_cũ	T_cũ	S_mới	T_mới	tin_cậy	lý_do	Comment
   ```
   - Chỉ ghi dòng **đề xuất đổi** hoặc **không chắc**.
   - `tin_cậy`: `cao` (sai rõ theo quy tắc) · `TB` (khá chắc) · `hỏi` (cần chủ dự án quyết; S_mới/T_mới là phương án nghiêng về).
   - `lý_do`: tiếng Việt, ≤ 15 từ, nêu căn cứ (trích quy tắc/casebook nếu có).
4. **Kiểm tra lại đề xuất** (người điều phối): soát ngẫu nhiên + toàn bộ dòng `cao`; đối chiếu quy ước dữ liệu khi đề xuất đổi Sentiment hàng loạt.
5. **Hỏi chủ dự án** các dòng `hỏi` — gom theo nhóm vấn đề, mỗi nhóm kèm 2–3 ví dụ, đề xuất một quy tắc chung để chủ dự án chốt một lần.
6. **Áp dụng** (§9.3), **xác minh** (§9.5), **cập nhật tài liệu này** nếu có quyết định mới.

### 9.3. Áp dụng thay đổi

- Định vị dòng theo `dòng` **và** assert nội dung comment khớp (tránh lệch dòng sau khi file thay đổi).
- Assert nhãn cũ khớp `S_cũ/T_cũ` trước khi ghi đè.
- Ghi lại đúng định dạng gốc (§11).

### 9.4. Log

Mỗi bước tạo một CSV trong `merged/_report/` hoặc `merged/_report/audit/`, tối thiểu các cột: `split, dòng, S_cũ, T_cũ, S_mới, T_mới, lý_do, Comment`.

### 9.5. Xác minh sau mỗi lần sửa

- Số dòng mỗi tập không đổi (trừ bước dedup/leak có chủ đích).
- Mọi nhãn hợp lệ, mỗi dòng đúng 3 cột.
- Không có trùng trong file, không có giao nhau giữa các tập (§10.3).
- File đọc lại bằng `csv.reader` round-trip ra đúng byte (§11).

---

## 10. Vệ sinh dữ liệu: trùng lặp & leak

### 10.1. Khoá so khớp **[ĐÃ CHỐT]**

Chỉ **làm sạch tối thiểu** để so sánh; nội dung ghi ra file **giữ nguyên bản gốc**:
1. Chuẩn hoá Unicode **NFC** (tiếng Việt hay lệch dựng sẵn / tổ hợp).
2. Bỏ URL (`http…`, `www.…`, `domain.com/…`).
3. Bỏ thẻ HTML lỗi (`<br>`, `<p>`…) và unescape entity (`&amp;`…).
4. Bỏ timestamp video (`12:34`, `1:02:03`).
5. Gộp khoảng trắng thừa, trim.

**Không** lowercase, **không** bỏ dấu câu/emoji, **không** so khớp gần đúng (fuzzy). Câu gần trùng → bỏ qua. Comment có khoá rỗng (chỉ gồm timestamp/URL) → không đem so khớp.

Cài đặt tham chiếu: hàm `key()` trong `merged/_report/dedup.py`, bọc thêm `unicodedata.normalize('NFC', …)` trước khi gọi.

### 10.2. Quy tắc xử lý **[ĐÃ CHỐT]**

| Tình huống | Xử lý |
|---|---|
| Trùng trong cùng file, **cùng nhãn** | Giữ 1 dòng (dòng xuất hiện đầu tiên), xoá phần còn lại |
| Trùng trong cùng file, **nhãn mâu thuẫn** | **Báo chủ dự án**, không tự xoá; sau khi gán lại thì giữ 1 dòng (vị trí đầu tiên) với nhãn mới |
| Comment có trong train **và** valid | Xoá ở **valid** |
| Comment có trong train/valid **và** challenge | Xoá ở **challenge** (challenge không được chứa dữ liệu đã thấy khi train) |

Thứ tự chạy: dedup trong file → loại leak challenge (so với train ∪ valid) → loại leak valid (so với train).

### 10.3. Kiểm tra nhanh (phải ra 0)

```python
import csv, unicodedata
# key() lấy từ merged/_report/dedup.py
K = lambda s: key(unicodedata.normalize('NFC', s))
ld = lambda f: list(csv.reader(open(f, encoding='utf-8-sig', newline='')))[1:]
T, V, C = (ld(f'merged/{n}/{n}.csv') for n in ['train', 'valid', 'challenge'])
s = lambda D: {K(r[0]) for r in D} - {''}
assert not (s(T) & s(V)) and not (s(T) & s(C)) and not (s(V) & s(C))
```

---

## 11. Định dạng file & cách ghi an toàn

Ba file có định dạng **khác nhau** — phải giữ nguyên:

| File | encoding khi đọc/ghi | lineterminator |
|---|---|---|
| `train/train.csv` | `utf-8-sig` (có BOM) | `\n` |
| `valid/valid.csv` | `utf-8` | `\r\n` |
| `challenge/challenge.csv` | `utf-8` | `\r\n` |

Lưu ý pandas: đọc train phải dùng `encoding="utf-8-sig"`, nếu không cột đầu sẽ thành `"﻿Comment"`.

Mẫu ghi an toàn (kiểm tra round-trip trước khi sửa):

```python
import csv, io
FMT = {'train': ('utf-8-sig', '\n'), 'valid': ('utf-8', '\r\n'), 'challenge': ('utf-8', '\r\n')}

def load(split):
    enc, lt = FMT[split]
    raw = open(f'merged/{split}/{split}.csv', 'rb').read()
    rows = list(csv.reader(io.StringIO(raw.decode(enc), newline='')))
    b = io.StringIO(); csv.writer(b, lineterminator=lt).writerows(rows)
    assert b.getvalue().encode(enc) == raw, 'round-trip mismatch — dừng lại'
    return rows            # rows[0] là header; rows[dòng-1] là bản ghi số "dòng"

def save(split, rows):
    enc, lt = FMT[split]
    b = io.StringIO(); csv.writer(b, lineterminator=lt).writerows(rows)
    open(f'merged/{split}/{split}.csv', 'wb').write(b.getvalue().encode(enc))
```

---

## 12. Những lỗi đã mắc — đừng lặp lại

1. **Gán nhãn theo nhóm mà không đọc lại từng câu** — 45 câu "3 que" bị gán chung, sai những câu tổng thể là khen hoặc có "lol". → Luôn đọc lại từng dòng sau khi áp nhãn theo nhóm/từ khoá.
2. **Áp mức sàn cho câu chỉ nhắc tới từ** (hỏi "DLV là gì", tự nhận, bênh vực) → bị đẩy lên T=3 vô lý. → Xem §5.3.
3. **Coi "thằng/tao/cưng" là toxic** → gán T=1 sai. → §4.3.
4. **Bỏ sót mỉa mai** → để S=0/1 cho câu châm chọc. → §3.3.
5. **So khớp không chuẩn hoá NFC** → bỏ sót leak giữa các câu giống hệt nhau nhưng khác mã hoá dấu. → §10.1.
6. **Regex có `pkill -f <chuỗi>`** khớp luôn shell đang chạy → tự kill. Tránh `pkill -f` với chuỗi xuất hiện trong chính lệnh.
7. **Thuật toán O(n²)** khi kiểm tra trùng (lồng list comprehension) → timeout. Dùng `Counter`/`dict`.

---

## 13. Phụ lục: file báo cáo & backup hiện có

**Báo cáo (`merged/_report/`)**
- `dedup.py` — script dedup/leak gốc (hàm `key()`).
- `removed.csv`, `conflicts_in_file.csv`, `cross_label_diff.csv`, `empty_key.csv` — đợt dedup đầu (số dòng theo file gốc trong `_backup/`).
- `removed_nfc_leak.csv` — 8 leak bổ sung sau khi chuẩn hoá NFC.
- `valid_relabel.csv`, `challenge_relabel.csv` — gán lại các nhóm mâu thuẫn.
- `train_vs_train_after/` — so sánh train cũ với train_after.
- `audit/` — `challenge_suggestions.csv`, `heuristic_flags_all.csv`, `neardup_label_diff.csv`, `identity_terms_review.csv`, `identity_applied.csv`, `identity_above_floor.csv`, `condemn_applied.csv`, `3que_review.csv`, `3que_applied.csv`, `context_review_fixes.csv`, `sarcasm_fixes.csv`.

**Backup (`merged/_backup/`)** — bản trước mỗi bước: `train.csv/valid.csv/challenge.csv` (gốc trước dedup), `*_before_relabel`, `*_before_audit`, `*_before_identity`, `*_before_condemn`, `*_before_3que`, `*_before_context_review`.

**Trạng thái (2026-11-08)**: đã duyệt toàn bộ challenge (6 phần). Các đề xuất nằm trong `merged/_report/audit/challenge_full_review.csv`; các cột S_mới/T_mới của dòng `hỏi` là do chủ dự án tự điền.
- Đã áp 83 dòng `hỏi` do chủ dự án chốt (log: `challenge_hoi_applied.csv`) và 452 dòng `cao`/`TB` (log: `challenge_full_review_applied.csv`).
- Dòng 1670 (`Cạo râu đi chị ba`) chưa áp vì Toxicity còn trống.
- 42 dòng công kích không chửi thề mới áp Sentiment, còn Toxicity chờ chủ dự án chốt (`challenge_ask_round2.csv`).
- Phần 3 và phần 4 (dòng 1082–2161) mới có 44 đề xuất, trong khi mỗi phần khác có 104–139. Hai phần này cần duyệt lại cho đồng đều.
- Train và valid **chưa** được duyệt lại theo các quy tắc mới ("thương xót = 1", mỉa mai, v.v.).
