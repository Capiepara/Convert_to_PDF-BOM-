# BOM Excel sang PDF

Tool chạy hoàn toàn trên trình duyệt (không có server, không upload file).

## Đưa lên GitHub Pages để có link

1. Vào github.com, bấm **New repository**, đặt tên (ví dụ `bom-pdf-tool`), chọn **Public**, bấm **Create repository**.
2. Bấm **uploading an existing file**, kéo file `index.html` vào, bấm **Commit changes**.
3. Vào **Settings > Pages**. Ở mục **Build and deployment**, chọn **Deploy from a branch**, branch **main**, thư mục **/(root)**, bấm **Save**.
4. Chờ khoảng 1 phút. Link sẽ là `https://<tên-github>.github.io/bom-pdf-tool/`

Lưu ý: repo Public thì ai có link đều mở được tool, nhưng file Excel không bao giờ rời khỏi máy người dùng.

## Cách dùng

1. Chọn **Season** (tự điền theo file đầu tiên nếu để trống) và **Stage** (LR2, FLC, SMS, CFM). Date để trống nếu file không có ngày.
2. Kéo thả hoặc chọn nhiều file Excel một lúc. Gender và Model name tự đọc từ file, sửa được ngay trên từng dòng.
3. Bấm **Chuyển sang PDF**, rồi tải từng file hoặc **Tải tất cả (.zip)**.

Tên file: `SEASON STAGE BOM – GENDER MODEL NAME - D.M.YYYY.pdf` (toàn bộ chữ in hoa).
