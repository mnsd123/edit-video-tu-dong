# vf studio — giao diện edit cho skill /edit-video-tu-dong (/smilee-video)

Ngày: 2026-11-10 · Người duyệt: chủ repo · Trạng thái: chờ duyệt

## Vấn đề
Hiện mọi chỉnh sửa sau render đều đi qua chat: người dùng mô tả bằng lời → Claude sửa `jobs/<lô>/edl.json` → render
3–5 phút/bản → xem → lặp. Sửa một chữ, dời một thẻ, cắt bớt 0,3 s cũng mất một vòng. Muốn có "phiên bản khác"
(đổi preset, đổi hook, bản ngắn) phải nhờ Claude viết lại biến thể.

## Mục tiêu
Một web app chạy trên máy (`vf studio lo-NN` → trình duyệt mở `http://127.0.0.1:8765`) đọc/ghi thẳng `edl.json`
của lô, xem trước trực tiếp, và gọi lại đúng pipeline `vf render` hiện có. Không thay pipeline; studio chỉ là lớp sửa EDL.

## Chức năng (đủ mọi phương án — người dùng chọn 2026-11-10)
1. **Danh sách biến thể** của lô: id, preset, hook_type, thời lượng, trạng thái render (OK / CẦN XEM / chưa render /
   đã sửa sau render), điểm `review.csv`.
2. **Xem trước trực tiếp**: `@remotion/player` phát các đoạn nguồn (`work/<nguồn>/normalized.mp4`) nối theo timeline
   + lớp chữ/đồ hoạ thật của `remotion/src/Overlay.tsx`, props tính bằng chính code Python của render (vị trí né mặt,
   vùng an toàn, cam kết tự phát hiện). Bật/tắt lưới vùng an toàn. Zoom hiệu ứng là xấp xỉ — bản render là chuẩn.
3. **Sửa chữ** (`overlays`): text (cú pháp `*nhấn*`, `[nhấn 2]`), sub, style (lọc theo `overlay_style` của preset),
   animation, position, emphasis, start/end, avoid, free. Thêm/xoá/nhân đôi.
4. **Sửa đồ hoạ** (`graphics`): chỉ các kind thuộc gói hồ sơ đã bật; form sinh theo từng kind (trường liên quan);
   `callout` đặt cx/cy bằng cách bấm lên khung xem trước; `stills` chọn khung bằng giờ nguồn + crop.
5. **Sửa hiệu ứng** (`effects`): zoom_punch, zoom_to (focus bấm trên khung), shake, sfx.
6. **Timeline kéo thả**: kéo đổi thứ tự đoạn, kéo mép đầu/cuối (bắt dính đầu/cuối chữ phiên âm và điểm chuyển cảnh),
   xoá đoạn, đưa một đoạn ≤ 3 s lên làm hook, đặt `protect`/`chapter`/`video_end`. Chữ/đồ hoạ/hiệu ứng **đi theo đoạn**
   chúng đang nằm khi đoạn bị dời hoặc cắt.
7. **Trình duyệt nguồn**: chọn nguồn, xem phiên âm từng chữ, khoảng `keep.json` và điểm chuyển cảnh; bôi một khoảng chữ
   → thêm thành đoạn mới vào vị trí chọn.
8. **Kiểm tra (lint)** ngay khi sửa: lỗi schema (hook > 3 s, start ≥ end…), preset/gói đồ hoạ sai hồ sơ, cụm cấm của
   brief, lint thứ tự chương / cắt giữa từ / dính tiếng / chữ quá nhanh — hiện tiếng Việt, bấm vào nhảy tới giờ lỗi.
9. **Lưu**: ghi `edl.json` (giữ thụt lề 1, `ensure_ascii=False`), chụp bản trước vào `jobs/<lô>/history/`, phát hiện
   file đã bị sửa từ ngoài (Claude/skill) và không ghi đè. Hoàn tác/làm lại trong phiên; khôi phục bản lịch sử.
10. **Xuất phiên bản khác** từ một biến thể đã có: nhân bản; đổi preset; đổi hook (lấy hook của biến thể khác);
    bản cắt ngắn theo số giây; đổi CTA; bản sạch không chữ; tạo nhiều bản một lần (vd 1 biến thể × 3 preset). Biến thể
    mới có `derived_from` + `edit_note`, id `<gốc>-v2`, `-v3`…
11. **Render từ giao diện**: chọn biến thể → chạy `vf render <lô> --only …` (bản không đổi tự GIỮ), xem log trực tiếp,
    chỉ một lượt render/lô một lúc; xong thì xem mp4 + `_sheet.png` + ghi chú QA.
12. **Chấm & ghi chú**: nhập `score` + `notes` từng biến thể → ghi `review.csv` (nối `; YYYY-MM-DD: …`, không ghi đè)
    để skill `video-learnings` học tiếp.

## Kiến trúc
- Backend: gói `vf/studio/` (FastAPI + uvicorn, extra `[studio]`), chỉ nghe `127.0.0.1`. Logic thuần (sửa timeline,
  tạo phiên bản, lưu/lịch sử) viết bằng Python có test; frontend không tự tính lại luật.
- Frontend: `studio/web/` (Vite + React 19 + TypeScript + `@remotion/player`), import thẳng component từ
  `remotion/src/`. Build ra `vf/studio/static/` (không commit), setup script tự build.
- Composition mới `remotion/src/Preview.tsx` = các đoạn video nguồn + `Overlay`.

## Ràng buộc
- Không đổi kết quả render của biến thể không sửa (dấu vân tay cache giữ nguyên).
- Không bao giờ ghi vào `source gốc`, `video tham khảo`; chỉ ghi `jobs/<lô>/` và `out/<lô>/review.csv`.
- Chạy được Mac (Apple Silicon) và Windows; tên file tiếng Việt (NFC/NFD, khoảng trắng).
- Giao diện tiếng Việt.

## Ngoài phạm vi
Chỉnh màu footage; nhiều người sửa cùng lúc; chạy trên cloud; sửa `brief.yaml`/hồ sơ trong giao diện (đọc thôi).
