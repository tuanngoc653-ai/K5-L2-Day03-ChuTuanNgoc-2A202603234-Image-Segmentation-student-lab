# Báo cáo thực hành Day 3 — Segmentation

- Mã học viên theo lớp: 2A202603234
- Ngày / CVAT local: 11–12/11/2026 · CVAT 2.74.1 chạy local bằng Docker, bật thêm nuclio (serverless)
- Công cụ đã dùng:
  - **SAM3** (`facebook/sam3`) chạy trên GPU RTX 3050 4 GB, nối vào CVAT qua nuclio thành 2 model: *SAM3 (click / box)* — AI Tools → Interactors, và *SAM3 (text prompt)* — Automatic annotation (prompt = tên label). Đã hỏi và được mentor/labcoach đồng ý dùng SAM3.
  - Hậu xử lý tự viết cho SAM3: tách các mảng rời của vùng stuff, bản đồ nhãn stuff không chồng lấn, chừa vật (thing) ra khỏi stuff, thêm prompt "tree/bush" cho vegetation, lọc mask trùng.
  - Brush / Polygon / Eraser của CVAT để sửa mask.
  - `scoring/score.py` của repo — chỉ chạy **sau khi** labcoach phát ground truth ba tier.
  - Trợ lý AI Claude (Claude Code): hỗ trợ cài SAM3 và nối vào CVAT, kiểm tra annotation, chạy script chấm.

## 1. Bài đã nộp

| Task | File ZIP đúng tên | Hoàn thành mấy ảnh | Điểm tối đa (coach chấm sau) |
| --- | --- | ---: | ---: |
| easy_semantic | `submissions/easy_semantic.zip` (Segmentation mask 1.1) | 3 / 3 | 20 |
| medium_instance | `submissions/medium_instance.zip` (COCO 1.0) | 3 / 3 | 32 |
| hard_panoptic | `submissions/hard_panoptic.zip` (COCO 1.0) | 2 / 2 | 30 |
| cp1_holes | `submissions/cp1_holes.zip` (COCO 1.0) | 1 / 1 | 3 |
| cp2_slice | `submissions/cp2_slice.zip` (COCO 1.0) | 1 / 1 | 3 |
| cp5_occlusion | `submissions/cp5_occlusion.zip` (COCO 1.0) | 1 / 1 | 3 |
| cp3_thin | `submissions/cp3_thin.zip` (Segmentation mask 1.1) | 1 / 1 | 3 |
| cp4_curb | `submissions/cp4_curb.zip` (Segmentation mask 1.1) | 1 / 1 | 3 |
| cp6_coverage | `submissions/cp6_coverage.zip` (Segmentation mask 1.1) | 1 / 1 | 3 |
| **Tổng tối đa** | | | **100** |

Không có export lỗi. Bản cuối export 12/11/2026 lúc 01:13; `scripts/inspect_submissions.py --dir submissions` báo OK cả 9 ZIP.

## 2. Một quyết định trước khi dùng gợi ý

- Class và quy tắc tôi dùng để chọn biên: chỉ giữ **phần nhìn thấy** của vật; hai vật cùng class sát nhau là hai object; người ngồi trên xe máy là `person` riêng, xe là `motorcycle` riêng.
- Nếu dùng gợi ý sau đó: vùng gợi ý sai/đúng, hành động sửa/giữ và lý do: kiểm tra kết quả SAM3 thấy ảnh `000000458325.jpg` có **hai mask `car` gần trùng nhau (IoU 0,91)** — cùng một xe bị gán hai lần; quy tắc "mỗi vật một object" nên giữ một mask. Ảnh `000000181542.jpg` có xe bán tải màu hồng SAM3 gán `car`.

## 3. Một lỗi tôi tìm thấy và sửa

- Task/ảnh/vùng: `hard_panoptic`, cả hai ảnh (`000000350023.jpg`, `000000460147.jpg`) — hai hàng cây hai bên đường.
- Lỗi thuộc loại: **phủ vùng / thiếu lớp** (stuff `vegetation` gần như không có).
- Bằng chứng tôi nhìn thấy: lần chạy SAM3 đầu chỉ có 1 mask `vegetation` ở ảnh 1 (dải cây giữa đường) và **0** ở ảnh 2; vẽ annotation ra ảnh thấy hai khối cây lớn để trống. Prompt "vegetation" ở ảnh chạng vạng chỉ phủ 0,2% ảnh.
- Quy tắc và hành động sửa: panoptic phải vẽ cả stuff (road/sidewalk/building/vegetation/sky). Tôi bổ sung prompt "tree" và "bush" gộp vào `vegetation`, hạ ngưỡng tin cậy cho stuff xuống 0,3, rồi chạy lại Automatic annotation → vegetation phủ khoảng 14% (ảnh 1) và 17% (ảnh 2), hai hàng cây đã có nhãn.
- Sau sửa đã Save và export lại chưa? **Đã** — `hard_panoptic.zip` export lại (lần 1: 138 object, trước đó 87; lần cuối 12/11 01:13: 113 object).

Kết quả tự chạy script (chỉ chạy **sau khi** labcoach phát ground truth):

| Bản export | easy_semantic | medium_instance | hard_panoptic | Tổng 3 tier |
| --- | ---: | ---: | ---: | ---: |
| 12/11 00:11 (trước khi xem reference) | mIoU 0,774 → 16,6/20 | 0,759 → 25,5/32 | PQ 0,424 → 14,9/30 | **57,0/82** |
| 12/11 01:13 (sau khi xem reference) | mIoU 0,850 → 20,0/20 | 0,759 → 25,5/32 | PQ 0,450 → 16,7/30 | **62,2/82** |

Điểm tự đánh giá, không phải điểm chính thức. Thay đổi giữa hai bản là **sau khi đã xem reference**: dùng script chấm để tìm chỗ mất điểm, rồi chỉnh hậu xử lý SAM3 (không tách trời khỏi tán cây vì reference vẽ tán cây thành khối; thêm prompt "lawn" cho sidewalk; ngưỡng tin cậy vật 0,6 cho panoptic để bớt vật thừa) và chạy lại `easy_semantic`, `hard_panoptic`, `cp3_thin`, `cp6_coverage`. Giữ nguyên 3 object vẽ tay ở `easy_semantic` và bản đã chỉnh của `medium_instance`, `cp4_curb`. File ground truth không đưa vào fork.

## 4. Ba ca chưa chắc hoặc đã cân nhắc

| Ảnh/vị trí | Hai cách hiểu có thể | Quy tắc/chứng cứ | Quyết định hoặc câu hỏi cho coach |
| --- | --- | --- | --- |
| 1. `easy_semantic` · `7ee6d192-89e2408b.jpg` · hai sườn đồi cỏ khô hai bên cao tốc | (a) `vegetation`; (b) "terrain" — đất/cỏ thấp, nằm ngoài 5 class nên để trống | Schema Easy chỉ có 5 class, không có terrain; cỏ khô không phải cây/bụi | Lúc làm: để trống, chỉ gán vegetation cho cụm cây xanh. Sau khi có reference thấy đồi được gán `vegetation` (vegetation IoU ảnh này 0,29). Câu hỏi: schema 5 lớp có quy ước gộp terrain vào vegetation không? |
| 2. `easy_semantic` · `81ae7cbb-6bc63a4a.jpg` · cây trụi lá góc trên bên trái | (a) cả tán là `vegetation`; (b) trời nhìn qua giữa cành là `sky` | "Vẽ phần nhìn thấy" và "phân biệt lỗ nền thật với chi tiết bên trong vật" | Lúc làm: tách các mảng trời lớn giữa cành thành `sky`, giữ cành dày là `vegetation`. Reference vẽ cả tán thành một khối vegetation. Câu hỏi: với cây trụi lá nên ưu tiên vẽ khối tán hay tách trời? |
| 3. `hard_panoptic` · `000000460147.jpg` · làn đường bên trái có vạch kẻ làn | (a) `road` vì là đường xe chạy; (b) `sidewalk` | Ranh road–sidewalk theo **chức năng/bó vỉa**, không theo màu; có vạch kẻ làn → đường xe | Quyết định: giữ `road`. Reference gán vùng này là `sidewalk` (road IoU ảnh này 0,14). Câu hỏi cho coach: đây có phải lỗi/ quy ước riêng của reference không? |

## Phụ lục — Lịch sử chạy và tinh chỉnh SAM3 trên CVAT

Phần bổ sung, không thuộc bốn mục bắt buộc; soạn cùng trợ lý AI Claude từ log và số liệu thật của buổi làm. Ground truth và dữ liệu trung gian dùng để tinh chỉnh **không** đưa vào fork.

### A. Đưa SAM3 vào CVAT local

| Bước | Vấn đề gặp | Cách xử lý |
| --- | --- | --- |
| Cài SAM3 | Python 3.15 trên Windows: `numpy<2` và `pyyaml` không có bản dựng sẵn; thiếu `einops`, `triton` | Giữ numpy 2.x, cài `sam3` với `--no-deps`; pyyaml bản thuần Python; `triton-windows` |
| Chạy trên GPU 4 GB | RTX 3050 Laptop | Model ~3,5 GB VRAM, đỉnh ~3,8 GB — vừa đủ |
| Agent-based function của CVAT | `cvat-cli function create-native` báo **404**: CVAT open-source không có `/api/functions` | Dùng **nuclio** (serverless chính thức của CVAT bản tự cài) |
| Kiến trúc | Không muốn build image torch/GPU trong Docker | SAM3 chạy trên Windows (`127.0.0.1:8765`); function nuclio chỉ là proxy chuyển tiếp |
| Server bị crash | torch trên Python 3.15 lỗi access violation khi chạy ngoài main thread | Đưa mọi việc GPU về main thread qua hàng đợi |
| Khởi động lại Docker | Nuclio nhớ cổng cũ → CVAT gọi sai cổng | Cố định cổng function (32001–32005) |

Model hiện có trong CVAT: *SAM3 (click / box)* (AI Tools → Interactors), *SAM3 (text prompt)*, *SAM3 semantic*, *SAM3 instance*, *SAM3 panoptic* (Automatic annotation, prompt = tên label).

### B. Cải tiến **trước** khi có reference (quan sát lỗi bằng mắt)

| Lỗi quan sát | Cải tiến |
| --- | --- |
| Đổi mask sang polygon làm mất mảnh nhà bị cột/cây cắt rời (`817bca71`) | Tách mỗi mảng rời của stuff thành một object |
| Mask sky/vegetation/building chồng nhau, phụ thuộc thứ tự lớp | Bản đồ nhãn stuff: mỗi pixel một class (xác suất cao nhất) → chồng lấn 0% |
| Xe, cột bị road/building tô đè | Chừa pixel của thing/cột/biển ra khỏi stuff |
| Cây hai bên đường ở `hard_panoptic` gần như không có nhãn | Thêm prompt "tree", "bush" cho `vegetation`; ngưỡng stuff 0,3 |
| Một xe bị gán hai lần / một mask gộp nhiều vật | Lọc mask trùng (IoU ≥ 0,7) và mask bao trùm ≥ 2 vật khác |
| Thing nới 2 px chồng lên stuff (18% pixel thing) | Không nới mask thing → chồng lấn 0% |

Bản export lúc 12/11 00:11 (chưa xem reference): **57,0/82**.

### C. Tinh chỉnh **sau** khi labcoach phát reference

Cách làm: lưu kết quả thô của SAM3 cho 8 ảnh có reference một lần, chạy lại phần hậu xử lý với từng cấu hình và chấm bằng chính hàm của `scoring/score.py`. Chỉ giữ thay đổi có lý do gán nhãn rõ ràng (tránh khớp quá mức vào 8 ảnh).

| Thử nghiệm | Easy | Medium | Hard | Giữ? |
| --- | ---: | ---: | ---: | --- |
| Cấu hình trước đó | 17,1 | 24,6 | 16,0 | — |
| Tắt tách trời khỏi tán cây (reference vẽ tán thành khối) | 18,4 | | | ✅ |
| + prompt "lawn" cho sidewalk (reference tính bãi cỏ trước nhà là sidewalk) | **19,4** | | | ✅ (semantic) |
| + "hill"/"grass"/"house" | 18,3–18,4 | | | ❌ không tăng |
| + "pavement" cho sidewalk | 17,3 | | 16,5 | ❌ giảm |
| Ngưỡng stuff 0,4 | 17,3 | | | ❌ giảm |
| Ngưỡng thing 0,2–0,35 / 0,4–0,5 / 0,6 | | 23,5–24,4 / **24,6** / 21,1 | | giữ 0,5 cho instance |
| Ngưỡng thing panoptic 0,6 / 0,7 / 0,8 | | | **16,5** / 16,8 / 12,8 | ✅ 0,6 (0,7 bắt đầu sót xe) |
| Nới mask thing 1–2 px | | 20,7 / 12,5 | 14,2 | ❌ giảm mạnh |

Kết quả cuối: chạy lại `easy_semantic` (giữ 3 object vẽ tay), `hard_panoptic`, `cp3_thin`, `cp6_coverage`; giữ bản đã chỉnh của `medium_instance` (25,5 cao hơn SAM3 tự chạy 24,6) và `cp4_curb`. Export 12/11 01:13: **62,2/82** (easy 20,0 · medium 25,5 · hard 16,7).

### D. Giới hạn

- Cấu hình C được chọn trên đúng 8 ảnh có reference nên có thể khớp quá mức; checkpoint không có reference nên không đo được tác dụng ở đó.
- Medium gần như chạm trần khi chạy tự động: điểm bị giới hạn bởi độ sát biên (IoU ~0,80), không phải số object.
- Một số quy ước của reference khác cách hiểu theo quy tắc chức năng (ca 3, mục 4); tôi không sửa theo reference ở những chỗ đó.

