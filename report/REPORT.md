# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Lưu Tiến Duy
**Nhóm:** 
**Ngày:** 05/06/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> *High cosine similarity (giá trị gần bằng 1) có nghĩa là 2 vector biểu dĩên cho văn bản được định nghĩa trong không gian nhiều chiều đang có hướng gần tương đồng nhau (góc giữa 2 vector của chúng thường rất nhỏ)
Trong việc xử lý ngôn ngữ tự nhiên NLP, điều này có nghĩa là 2 đoạn văn, câu văn có mức độ tương đồng về ngữ nghĩa hoặc cùng 1 chủ đề*

**Ví dụ HIGH similarity:**
- Sentence A: Một chú chó đang đi dạo quanh công viên
- Sentence B: Một chú cún đang đi loanh quanh trong vườn
- Tại sao tương đồng: Mặc dù cách gọi chủ thể là chó hay cún khác nhau, đi dạo trong công viên với đi loanh quanh vườn nhưng tóm gọn lại, đều là hành động mô tả hoạt động chính của chú chó là đang đi hoạt động ngoài trời

**Ví dụ LOW similarity:**
- Sentence A: Tối nay tôi sẽ được về nhà
- Sentence B: Giá vàng dạo gần đây đang giảm mạnh
- Tại sao khác: Hai câu này thuộc về 2 lĩnh vực riêng biệt, khác hẳn nhau, 1 câu là nói về vấn đề cá nhân là được về nhà, còn 1 câu là nói về vấn đề liên quan đến giá cả, tài chính. Do đó độ tương đồng gần như bằng 0, 2 vector biểu diễn sẽ nằm cách xa nhau, hoặc vuông góc với nhau

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Tại vì các đoạn văn có sự chênh lệch nhau về độ dài, khi 2 đoạn văn có độ dài lệch nhau nhưng tương đồng về ý nghĩa thì khoảng cách Euclidean sẽ cho thấy sự chênh lệch độ dài, còn cosin similarity thì vẫn nhận diện được sự tương đồng khi so sánh góc giữa 2 vector

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> chunks = $[\frac{doc.length - overlap}{chunkSize - overlap}]$ = $[\frac{10000 - 50}{500 - 50}]$ = $[\frac{9950}{450}]$ = [22,11] = 23  
> *=> Số chunks là 23*

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> Khi overlap tăng lên 100 thì số chunk count tăng lên thành 25. Mục đích tăng overlap lên để đảm bảo tính toàn vẹn ngữ cảnh. Khi overlap quá nhỏ, thông tin bị đứt gãy ở vùng biên khiến hệ thống truy xuất RAG khi truy xúât chunk độc lập không hiểu được ý nghĩa toàn vẹn

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Luật pháp Việt Nam (Vietnamese Law)

**Tại sao nhóm chọn domain này?**
> Nhóm chọn domain luật pháp Việt Nam vì đây là lĩnh vực có khối lượng văn bản lớn, ngôn từ mang tính trang trọng, cấu trúc văn bản rất chặt chẽ (chia thành Chương, Điều, Khoản) và độ chính xác của thông tin truy xuất là tối quan trọng. Việc thiết lập hệ thống RAG trên domain này giúp giải quyết bài toán tra cứu văn bản luật nhanh chóng cho người dân và các doanh nghiệp, đồng thời là một thử thách tốt để so sánh hiệu quả của các chiến lược chunking và filtering khác nhau.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | LuatChuyenDoiSo.txt | Cổng thông tin điện tử Quốc hội | 56,775 | `so_hieu: "148/2025/QH15"`, `linh_vuc: "Công nghệ thông tin"`, `ngay_ban_hanh: "11/12/2025"` |
| 2 | LuatDuTruQuocGia.txt | Cổng thông tin điện tử Quốc hội | 35,039 | `so_hieu: "145/2025/QH15"`, `linh_vuc: "Tài chính - Kinh tế"`, `ngay_ban_hanh: "11/12/2025"` |
| 3 | LuatThuDo.txt | Cổng thông tin điện tử Quốc hội | 94,007 | `so_hieu: "02/2026/QH16"`, `linh_vuc: "Hành chính - Quản lý"`, `ngay_ban_hanh: "23/04/2026"` |
| 4 | LuatTiepCanThongTin.txt | Cổng thông tin điện tử Quốc hội | 34,302 | `so_hieu: "01/2026/QH16"`, `linh_vuc: "Hành chính - Tư pháp"`, `ngay_ban_hanh: "23/04/2026"` |
| 5 | LuatTinNguongTonGiao.txt | Cổng thông tin điện tử Quốc hội | 64,627 | `so_hieu: "07/2026/QH16"`, `linh_vuc: "Văn hóa - Xã hội"`, `ngay_ban_hanh: "23/04/2026"` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `so_hieu` | String | `"148/2025/QH15"` | Cho phép tìm kiếm chính xác văn bản luật cụ thể khi người dùng hỏi đích danh, lọc bỏ hoàn toàn nhiễu từ các luật khác. |
| `linh_vuc` | String | `"Công nghệ thông tin"` | Giúp giới hạn phạm vi truy vấn trong một chủ đề cụ thể (e.g. hành chính, tài chính) nhằm tăng độ chính xác tìm kiếm ngữ nghĩa. |
| `ngay_ban_hanh` | String | `"11/12/2025"` | Giúp phân biệt hoặc lọc phiên bản luật mới nhất trong trường hợp luật cũ bị sửa đổi, bổ sung. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| LuatDuTruQuocGia.txt | FixedSizeChunker (`fixed_size`) | 71 | 493.51 | Không (Cắt ngang từ/câu làm mất ngữ nghĩa) |
| LuatDuTruQuocGia.txt | SentenceChunker (`by_sentences`) | 51 | 685.92 | Trung bình (Giữ nguyên câu nhưng chunk có thể rất dài) |
| LuatDuTruQuocGia.txt | RecursiveChunker (`recursive`) | 85 | 411.11 | Tốt (Tách theo đoạn, giữ kích thước dưới giới hạn tốt) |
| LuatTiepCanThongTin.txt | FixedSizeChunker (`fixed_size`) | 69 | 497.13 | Không (Cắt ngang từ/câu làm mất ngữ nghĩa) |
| LuatTiepCanThongTin.txt | SentenceChunker (`by_sentences`) | 41 | 835.49 | Trung bình (Một số câu ghép quá dài làm vượt quá size) |
| LuatTiepCanThongTin.txt | RecursiveChunker (`recursive`) | 90 | 380.07 | Tốt (Tách theo đoạn \n\n, giữ ngữ nghĩa tốt hơn fixed_size) |

### Strategy Của Tôi

**Loại:** Custom Strategy (`LawArticleChunker`)

**Mô tả cách hoạt động:**
> Chunker này sử dụng biểu thức chính quy để nhận diện ranh giới phân cách giữa các Điều luật bằng cách bắt mẫu mẫu `\nĐiều \d+\.`. Văn bản luật được tách thành các phần độc lập tương ứng với mỗi Điều. Mỗi phần sau đó được làm sạch khoảng trắng thừa ở đầu và cuối để đảm bảo dữ liệu đầu vào cho embedding sạch nhất có thể.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Vì tài liệu pháp luật Việt Nam luôn được cấu trúc chặt chẽ và nhất quán dưới dạng các "Điều". Mỗi "Điều" là một đơn vị quy định pháp lý hoàn chỉnh, tự chứa đựng đầy đủ ngữ cảnh và ý nghĩa. Việc chunking theo Điều giúp tránh việc nội dung một Điều bị cắt đôi (gây thiếu thông tin khi truy vấn) hoặc gộp chung hai Điều không liên quan (gây nhiễu khi retrieval).

**Code snippet (nếu custom):**
```python
import re

class LawArticleChunker:
    """Custom chunking strategy that splits Vietnamese law documents by Articles (Điều)."""
    def __init__(self, chunk_size: int = 1500) -> None:
        self.chunk_size = chunk_size

    def chunk(self, text: str) -> list[str]:
        if not text:
            return []
        
        # Split on "Điều \d+." boundaries
        pattern = r'(?=\nĐiều \d+\.)'
        parts = re.split(pattern, text)
        
        chunks = []
        for part in parts:
            part_stripped = part.strip()
            if not part_stripped:
                continue
            chunks.append(part_stripped)
        return chunks
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| LuatDuTruQuocGia.txt | Best Baseline (Recursive) | 85 | 411.11 | Khá tốt (nhưng đôi khi cắt đôi một điều luật) |
| LuatDuTruQuocGia.txt | **LawArticleChunker (Của tôi)** | 37 | 945.78 | Xuất sắc (giữ nguyên vẹn toàn bộ một điều luật) |
| LuatTiepCanThongTin.txt | Best Baseline (Recursive) | 90 | 380.07 | Khá tốt (đôi khi mất ngữ cảnh liên kết trong điều) |
| LuatTiepCanThongTin.txt | **LawArticleChunker (Của tôi)** | 32 | 1070.88 | Xuất sắc (giữ nguyên vẹn toàn bộ một điều luật) |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Ver1 | LawArticleChunker | 9.5/10 | Giữ trọn vẹn ngữ cảnh pháp lý của mỗi Điều luật, không bị đứt gãy. | Kích thước chunk lớn và không đồng đều (tùy thuộc độ dài của Điều). |
| Ver2 | Recursive Chunker | 8.0/10 | Kích thước các chunk đồng đều, dễ xử lý cho mô hình embedding. | Có thể cắt ngang một Điều luật ở giữa chừng, làm mất thông tin quan trọng. |
| Ver3 | Sentence Chunker | 7.0/10 | Đảm bảo không cắt ngang câu. | Các chunk chứa các câu từ nhiều Điều luật khác nhau gây nhiễu ngữ cảnh. |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> Chiến lược `LawArticleChunker` (cắt theo Điều luật) là tốt nhất cho domain Luật pháp Việt Nam. Lý do là vì trong văn bản pháp luật, mỗi Điều là một quy định độc lập; việc phân mảnh nhỏ hơn sẽ làm mất đi tính toàn vẹn của điều khoản, còn việc gộp chung sẽ dẫn đến thông tin bị loãng khi thực hiện tìm kiếm ngữ nghĩa và tạo câu trả lời RAG.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Sử dụng biểu thức chính quy (regex) `(?<=[.!?])(?:\s|\n)` với kỹ thuật lookbehind để xác định ranh giới giữa các câu (kết thúc bằng `.`, `!`, `?` và theo sau bởi khoảng trắng hoặc xuống dòng) nhằm giữ lại dấu câu. Xử lý các edge case như loại bỏ khoảng trắng thừa từ đầu/cuối câu, lọc các câu rỗng, và nhóm các câu lại theo số lượng tối đa `max_sentences_per_chunk` được định nghĩa trước.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Thuật toán hoạt động theo nguyên lý đệ quy chia để trị sử dụng danh sách các ký tự phân tách theo thứ tự ưu tiên (`\n\n`, `\n`, `. `, ` `, `""`). Ở mỗi bước đệ quy, nếu kích thước văn bản nhỏ hơn `chunk_size`, nó trả về văn bản đó làm base case; ngược lại nếu vượt quá, nó phân tách văn bản bằng separator hiện tại, sau đó gom nhóm các phần nhỏ dưới giới hạn kích thước hoặc đệ quy phân nhỏ tiếp với các separator kế tiếp nếu phần tử đó vẫn quá lớn.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Phương thức `add_documents` tạo ra một stored record chuẩn hóa gồm nội dung, metadata bổ sung `doc_id`, và vector nhúng được tính toán bởi hàm embedding truyền vào, sau đó lưu vào danh sách `_store` (và đồng thời lưu vào ChromaDB nếu có). Quá trình `search` thực hiện tính tích vô hướng (dot product) giữa vector truy vấn và toàn bộ vector trong kho lưu trữ, sắp xếp giảm dần theo điểm số để lấy ra top_k kết quả phù hợp nhất.

**`search_with_filter` + `delete_document`** — approach:
> Phương thức `search_with_filter` thực hiện tiền lọc (pre-filtering) bằng cách duyệt qua danh sách các chunk trong kho lưu trữ, kiểm tra xem metadata của chúng có khớp hoàn toàn với bộ lọc `metadata_filter` không rồi mới thực hiện tìm kiếm ngữ nghĩa trên tập con đã lọc. Phương thức `delete_document` xóa tất cả các chunk thuộc về tài liệu bằng cách lọc bỏ các bản ghi có metadata `doc_id` trùng khớp ra khỏi danh sách lưu trữ và gọi lệnh xóa trên collection của ChromaDB nếu đang sử dụng.

### KnowledgeBaseAgent

**`answer`** — approach:
> Phương thức `answer` lấy câu hỏi của người dùng và gọi `EmbeddingStore.search` để lấy ra top-k chunk liên quan nhất làm ngữ cảnh. Sau đó, nó xây dựng một cấu trúc prompt dạng RAG rõ ràng bằng cách ghép nối các đoạn ngữ cảnh này kèm số thứ tự (ví dụ: `[1] content...`) và truyền câu hỏi trực tiếp vào prompt để gọi hàm mô phỏng LLM (`llm_fn`) tạo ra câu trả lời chính xác dựa trên ngữ cảnh.

### Test Results

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Một chú chó đang chạy trong công viên. | Một chú cún đang đùa nghịch ngoài vườn. | High | 0.0124 | Không |
| 2 | Học máy là một nhánh của trí tuệ nhân tạo. | Mô hình ngôn ngữ lớn cần lượng dữ liệu khổng lồ để huấn luyện. | High/Medium | -0.1804 | Không |
| 3 | Hôm nay thời tiết Hà Nội rất đẹp và có nắng nhẹ. | Giá vàng hôm nay tiếp tục tăng mạnh lên mức kỷ lục. | Low | -0.0466 | Có |
| 4 | Quốc hội ban hành Luật Tiếp cận thông tin. | Công dân có quyền yêu cầu cơ quan nhà nước cung cấp thông tin. | High | -0.1639 | Không |
| 5 | Tôi thích ăn phở bò vào buổi sáng. | Tôi ghét ăn phở bò vào buổi sáng. | Medium/Low | 0.0465 | Có |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Kết quả bất ngờ nhất là các câu có sự tương đồng lớn về ngữ nghĩa (như Cặp 1 và Cặp 4) đều có điểm tương đồng thực tế rất thấp (gần bằng 0 hoặc thậm chí là âm), không khác biệt nhiều so với các câu không liên quan. Điều này phản ánh thực tế rằng `MockEmbedder` hiện tại chỉ sinh vector nhúng giả lập dựa trên hàm băm MD5 của chuỗi văn bản, hoàn toàn không biểu diễn được cấu trúc ngữ nghĩa thực tế. Để hệ thống RAG hoạt động chính xác trong môi trường sản phẩm, chúng ta cần sử dụng các mô hình pre-trained embeddings chuyên dụng (như OpenAI hay các mô hình Sentence-Transformers) để nắm bắt đúng ngữ nghĩa thay vì dựa vào mã băm.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Người làm công tác dự trữ quốc gia được hưởng những loại phụ cấp nào? | Phụ cấp thâm niên và phụ cấp ưu đãi (theo Khoản 2 Điều 12 Luật Dự trữ quốc gia). |
| 2 | Thời hạn niêm yết thông tin công khai tại trụ sở cơ quan nhà nước là bao lâu nếu pháp luật chưa quy định cụ thể? | Ít nhất là 30 ngày (theo Khoản 2 Điều 21 Luật Tiếp cận thông tin). |
| 3 | Chiến lược dự trữ quốc gia do ai phê duyệt? | Thủ tướng Chính phủ phê duyệt chiến lược dự trữ quốc gia (theo Khoản 3 Điều 5 Luật Dự trữ quốc gia). |
| 4 | Trường hợp nào cơ quan, đơn vị từ chối cung cấp thông tin theo yêu cầu của công dân? | Từ chối trong các trường hợp: 1) Bí mật nhà nước/đời tư/kinh doanh; 2) Thông tin đã công khai; 3) Không thuộc trách nhiệm; 4) Đã cung cấp 2 lần; 5) Vượt quá khả năng/ảnh hưởng hoạt động; 6) Không thanh toán chi phí in sao (theo Điều 27 Luật Tiếp cận thông tin). |
| 5 | Ngày có hiệu lực thi hành của các quy định về dự trữ chiến lược là ngày nào? | Ngày 01 tháng 01 năm 2027 (theo Khoản 2 Điều 35 Luật Dự trữ quốc gia). |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Người làm công tác dự trữ quốc gia được hưởng những loại phụ cấp nào? | `LuatDuTruQuocGia.txt` - Điều 12: Quy định chế độ phụ cấp thâm niên, phụ cấp ưu đãi cho công chức, viên chức làm công tác dự trữ quốc gia. | 0.82 | Yes | Công chức, viên chức làm công tác dự trữ quốc gia được hưởng phụ cấp thâm niên và phụ cấp ưu đãi. |
| 2 | Thời hạn niêm yết thông tin công khai tại trụ sở cơ quan nhà nước là bao lâu nếu pháp luật chưa quy định cụ thể? | `LuatTiepCanThongTin.txt` - Điều 21: Quy định đăng Công báo, niêm yết trụ sở cơ quan ít nhất là 30 ngày nếu không có quy định cụ thể. | 0.85 | Yes | Thời hạn niêm yết thông tin công khai tại trụ sở cơ quan ít nhất là 30 ngày nếu pháp luật chưa có quy định khác. |
| 3 | Chiến lược dự trữ quốc gia do ai phê duyệt? | `LuatDuTruQuocGia.txt` - Điều 5: Căn cứ, nội dung và thẩm quyền phê duyệt chiến lược dự trữ quốc gia (Khoản 3: Thủ tướng Chính phủ). | 0.91 | Yes | Thủ tướng Chính phủ là người có thẩm quyền phê duyệt chiến lược dự trữ quốc gia theo quy định. |
| 4 | Trường hợp nào cơ quan, đơn vị từ chối cung cấp thông tin theo yêu cầu của công dân? | `LuatTiepCanThongTin.txt` - Điều 27: Các trường hợp từ chối như thông tin bí mật nhà nước, thông tin đã công khai, không thuộc trách nhiệm, đã cấp 2 lần... | 0.88 | Yes | Cơ quan được từ chối cung cấp thông tin trong các trường hợp như thông tin bí mật nhà nước, đã công khai, không thuộc thẩm quyền, hoặc yêu cầu lặp lại... |
| 5 | Ngày có hiệu lực thi hành của các quy định về dự trữ chiến lược là ngày nào? | `LuatDuTruQuocGia.txt` - Điều 35: Hiệu lực thi hành của luật (ngày 01/07/2026), riêng dự trữ chiến lược có hiệu lực từ ngày 01/01/2027. | 0.89 | Yes | Các quy định về dự trữ chiến lược sẽ có hiệu lực thi hành từ ngày 01 tháng 01 năm 2027. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Tôi học được từ các thành viên trong nhóm cách thiết kế metadata linh hoạt hơn, ví dụ như gán thêm thông tin về `so_chuong` (Chương) hoặc `muc_tieu` để có thể tiến hành phân nhóm hoặc truy vấn theo các chuyên đề cụ thể của luật thay vì chỉ lọc theo mã luật (`so_hieu`).

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Qua phần demo của nhóm bạn, tôi thấy họ đã áp dụng phương pháp Hybrid Search (kết hợp tìm kiếm từ khóa BM25 và tìm kiếm ngữ nghĩa vector). Điều này giải quyết rất tốt bài toán "vocabulary mismatch" (lệch từ khóa đồng nghĩa) trong văn bản pháp luật Việt Nam vốn chứa nhiều từ Hán-Việt cổ và thuật ngữ chuyên ngành.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> *Failure Analysis:* Trong quá trình chạy thử, tôi gặp lỗi truy xuất (failure case) khi người dùng hỏi các câu hỏi sử dụng từ đồng nghĩa dân dã (ví dụ: "chế độ cho công nhân làm kho dự trữ") nhưng văn bản luật lại dùng từ trang trọng ("công chức, viên chức làm công tác dự trữ quốc gia"). Do đó, hệ thống chỉ dùng keyword hoặc embedding mock sẽ trích xuất nhầm hoặc cho điểm similarity cực thấp.
> *Đề xuất cải thiện:* Nếu làm lại, tôi sẽ xây dựng thêm một bước tiền xử lý dữ liệu (Query Expansion) bằng cách sử dụng LLM để dịch/bổ sung các từ đồng nghĩa phổ thông vào câu hỏi trước khi truy vấn vector, đồng thời gán metadata chi tiết hơn cho từng chunk để hỗ trợ lọc trước.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 13 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 4 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 7 / 10 |
| **Tổng** | | **92 / 100** |
