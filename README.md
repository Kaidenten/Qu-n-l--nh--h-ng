# Hệ thống Nhà hàng – Next.js (Single Project)

**Stack:** Next.js 14 (App Router) · TypeScript · MySQL + Prisma · Tailwind CSS · shadcn/ui

## Cài đặt (5 bước)
```bash
npm install
# 1. Bật MySQL (XAMPP/Laragon/Workbench). Sửa DATABASE_URL trong .env nếu có mật khẩu
npm run db:setup          # tạo DB + bảng + dữ liệu mẫu (prisma db push + seed)
# 2. Cài các component shadcn (chạy 1 lần)
# (đã có sẵn: button, card, input, badge, skeleton) – các component còn lại thêm khi cần:
npx shadcn@latest add label select dialog sheet tabs table calendar popover sonner dropdown-menu separator
npm run dev               # http://localhost:3000
npm run db:studio         # xem dữ liệu bằng giao diện web
```
Tài khoản demo (mật khẩu `Password@123`): admin@restaurant.vn · manager@ · staff@ · shipper@restaurant.vn · customer@example.com

## Cấu trúc thư mục
```
restaurant-app/
├── prisma/
│   ├── schema.prisma          # 22 model/enum chính cho MySQL
│   └── seed.ts                # dữ liệu mẫu
├── public/images/             # ảnh món ăn, banner
└── src/
    ├── app/
    │   ├── page.tsx           # Trang chủ
    │   ├── (customer)/        # menu · reservation · checkout · orders/[code] (tracking)
    │   ├── admin/             # dashboard · tables · orders · dishes · reports
    │   ├── kitchen/           # Kitchen Display
    │   └── api/               # dishes · tables · reservations · orders · promotions
    │                          # shipping/quote · kitchen/stream (SSE)
    ├── components/ui/         # shadcn sinh ra ở đây
    ├── hooks/
    └── lib/                   # prisma.ts · utils.ts (cn, formatVND)
```

## Đã hoàn thành
- Trang chủ `/` · Thực đơn `/menu` (lọc danh mục, tìm kiếm, lọc chay/mặn)
- API: `GET /api/categories` · `GET /api/dishes?category=&q=&type=&featured=&limit=`
- Ảnh món: đặt file vào `public/images/dishes/<slug>.jpg` (slug xem trong seed). Chưa có ảnh thì hiện placeholder.

## Luồng đặt món (đã hoàn thành)
`Thực đơn → bấm món (chọn size/topping/mức cay/ghi chú) → Giỏ hàng (Sheet) → /checkout → /orders/<mã>`
- Giỏ hàng lưu `localStorage` (Zustand persist), đồng bộ giữa các tab.
- Checkout: địa chỉ + bản đồ Leaflet/OSM (tìm địa chỉ, định vị, bấm/kéo ghim) → `/api/shipping/quote` tính phí theo `ShippingRule` (khoảng cách OSRM, fallback đường chim bay × 1.3) → mã giảm giá → COD / QR.
- Server **tính lại toàn bộ giá** (món, size, topping, ship, giảm giá) khi tạo đơn; không tin số tiền từ client.
- Tracking `/orders/<mã>`: timeline 5 bước, polling 3 giây, QR VietQR (nội dung = mã đơn) + nút "Tôi đã chuyển khoản".

### Demo tracking khi chưa có trang Admin
```bash
npm run order:advance -- HV-241009-XXXXX   # mỗi lần chạy: đẩy đơn sang bước kế tiếp (gán shipper, đánh dấu đã thanh toán COD khi hoàn tất)
```
Mở trang tracking ở trình duyệt, chạy lệnh nhiều lần và xem timeline tự cập nhật.

### API mới
`GET /api/geocode` · `POST /api/shipping/quote` · `POST /api/promotions/validate` · `POST /api/orders` · `GET /api/orders/[code]` · `POST /api/orders/[code]/payment`

## Đặt bàn · Admin · Màn hình bếp (đã hoàn thành)

| Màn hình | Đường dẫn | Quyền |
|---|---|---|
| Đặt bàn (khách) | `/reservation` | công khai |
| Đăng nhập nhân viên | `/admin/login` | — |
| Tổng quan | `/admin` | ADMIN · MANAGER · STAFF |
| Sơ đồ bàn + đặt bàn | `/admin/tables` | ADMIN · MANAGER · STAFF |
| Đơn hàng (gán shipper) | `/admin/orders` | ADMIN · MANAGER · STAFF |
| Món ăn (bật/tắt hết món) | `/admin/dishes` | ADMIN · MANAGER · STAFF |
| Báo cáo doanh thu / món bán chạy | `/admin/reports` | ADMIN · MANAGER |
| Màn hình bếp (SSE) | `/kitchen` | ADMIN · MANAGER · STAFF |

Tài khoản demo (mật khẩu `Password@123`): `admin@restaurant.vn`, `manager@restaurant.vn`, `staff@restaurant.vn`.

### CRUD Món ăn · Danh mục · Bàn (Server Actions + shadcn/ui)
| Màn hình | Đường dẫn | Server Actions (`src/app/actions/`) | Quyền |
|---|---|---|---|
| Món ăn | `/admin/dishes` | `createDish` · `updateDish` · `deleteDish` · `setDishFlags` | CRUD: ADMIN/MANAGER · bật/tắt “còn hàng”: cả STAFF |
| Danh mục | `/admin/categories` | `createCategory` · `updateCategory` · `deleteCategory` | ADMIN/MANAGER |
| Bàn ăn | `/admin/tables` → tab “Quản lý bàn” | `createTable` · `updateTable` · `deleteTable` | ADMIN/MANAGER |

- Mọi action trả về `ActionResult` (`{ ok, message }` | `{ ok: false, error, fieldErrors }`), form hiện lỗi từng ô và `toast` (sonner) cho kết quả.
- Form dùng **react-hook-form + zod** (`src/lib/schemas.ts` – cùng một schema cho client và server) → Server Action luôn kiểm tra lại dữ liệu và **quyền** (không chỉ dựa vào middleware).
- Slug tự sinh từ tên (bỏ dấu, tự thêm `-2`, `-3` nếu trùng) và **giữ nguyên khi sửa** để không gãy đường dẫn/ảnh. Ảnh = file tải lên | đường dẫn nhập tay | mặc định `/images/dishes/<slug>.webp` (nếu file có sẵn) hoặc `.jpg`.
- Xoá món đã nằm trong đơn hàng cũ → tự động **lưu trữ (ẩn)** thay vì xoá, để không phá lịch sử/báo cáo. Danh mục còn món, bàn đang có khách/còn lượt đặt → bị chặn kèm lý do.
- Sau mỗi thay đổi gọi `revalidatePath` cho `/admin/dishes`, `/admin/categories`, `/admin/tables`, `/menu` (+ `/`, `/reservation`); sửa bàn còn phát SSE `table.updated` để sơ đồ bàn đang mở tự cập nhật.
- Lưu ý ảnh tải lên: file được ghi vào `public/images/dishes`, chạy tốt với `next dev` / VPS. Với `next start` file mới thêm sau khi build có thể chưa được phục vụ cho tới khi khởi động lại, còn Vercel/serverless không ghi được đĩa → khi triển khai thật hãy dùng object storage (S3/R2/Cloudinary) rồi lưu URL vào ô “Đường dẫn ảnh”.

### Size · Topping · Nhóm tùy chọn (Mức cay, Đường, Đá…) – `/admin/options` (Admin/Quản lý)
Mỗi món có 3 loại tùy chọn, **giá cộng thêm đặt riêng cho từng món**:

| Loại | Cách hoạt động | Giá |
|---|---|---|
| **Size** | khách chọn 1; luôn có đúng 1 size mặc định | giá bán = giá gốc món + giá cộng thêm của size (có thể âm) |
| **Topping** | chọn nhiều, lấy từ *Danh mục topping* chung | giá chung, hoặc **giá riêng cho món** (để trống = dùng giá chung) |
| **Nhóm tùy chọn** (Mức cay, Đường, Đá, Nước chấm…) | mỗi nhóm *chọn 1* hoặc *chọn nhiều*, có thể *bắt buộc* | mỗi lựa chọn có giá cộng thêm riêng (thường 0đ) |

- Trang có 2 tab: **Tùy chọn theo món** (chọn món ở cột trái → chỉnh Size / Topping / Nhóm) và **Danh mục topping** (CRUD topping chung). Từ bảng Món ăn có lối tắt “Size & tùy chọn”.
- Tiện lợi: “Tạo nhanh Nhỏ / Vừa / Lớn”, “Thêm nhanh” nhóm Mức cay / Đường / Đá, và “Sao chép từ món khác” (giá size được quy đổi theo tỉ lệ giá gốc).
- Server Actions: `src/app/actions/options.ts` (size, topping chung, topping theo món, nhóm, lựa chọn, preset, sao chép) – cùng cơ chế `ActionResult` + toast như phần CRUD trước.
- **Giá luôn do server tính** (`src/lib/order-lines.ts`): kiểm tra size/topping/lựa chọn thuộc đúng món và còn bán, nhóm *bắt buộc* phải có lựa chọn, nhóm *chọn 1* không quá 1 mục. Đơn hàng lưu snapshot (`OrderItem.options`) nên sửa/xóa tùy chọn không làm đổi đơn cũ.
- Khách thấy tùy chọn ở dialog chọn món; bếp, trang tracking và Admin hiển thị đầy đủ các lựa chọn (vd: *Mức cay: Cay nhiều*).

**Nâng cấp từ bản cũ** (đã có DB): `npx prisma db push` (thêm bảng `OptionGroup`, `OptionChoice`, cột `DishTopping.price`, `OrderItem.options`) → `npm run db:add-spice-groups` để tạo nhóm “Mức cay” cho các món cay đang có (trước đây mức cay được gắn cứng trong giao diện). Giỏ hàng cũ trong trình duyệt tự được chuyển đổi (mức cay cũ chuyển thành ghi chú).

### Gọi món tại bàn bằng mã QR (DINE_IN)
**Luồng:** Admin xuất QR cho từng bàn → khách quét → `/menu?table=<id bàn>` → chọn món → giỏ hàng → **Đặt món** → đơn `DINE_IN` gắn đúng `tableId` → Bếp nhận ngay.

| Bước | Chi tiết |
|---|---|
| **Xuất QR** (`/admin/tables`) | Nút **Xuất QR** ở bảng “Quản lý bàn” và trong chi tiết bàn trên sơ đồ. Hộp thoại: xem QR, tải PNG, sao chép / mở thử liên kết, **In thẻ bàn**. Nút **In tất cả mã QR** → `/admin/tables/qr` (thẻ A5, 2 thẻ/hàng). |
| **Địa chỉ trong QR** | Dùng id bàn (khó đoán hơn mã “A1”): `<website>/menu?table=<id>`. Ô “Địa chỉ website” sửa được; mặc định lấy `NEXT_PUBLIC_SITE_URL` hoặc địa chỉ đang mở Admin. **Điện thoại không mở được `localhost`** → khi thử nội bộ hãy nhập IP LAN (vd `http://192.168.1.10:3000`), khi chạy thật dùng tên miền. |
| **Khách quét QR** | Server kiểm tra bàn còn hoạt động → trình duyệt ghi nhớ bàn 4 giờ (localStorage). Banner “Đang đặt món tại Bàn A1” trên cùng + chip “Bàn A1” ở header, giữ nguyên khi chuyển trang. Có nút “Thoát chế độ tại bàn”. QR sai/bàn ngưng dùng → thông báo lỗi. |
| **Đặt món** | Trong giỏ hàng chỉ còn nút **Đặt món** (tên, ghi chú, thanh toán tại quầy/QR đều không bắt buộc) — không cần địa chỉ hay số điện thoại. Giá vẫn do server tính lại. |
| **Real-time** | Tạo đơn xong server phát `order.created` rồi `table.updated` qua `/api/kitchen/stream`: **Bếp** hiện đơn + phát âm báo ngay (nhớ bấm “Bật âm báo” 1 lần); **sơ đồ bàn** tự chuyển bàn sang **Đang ăn** (kèm số đơn đang phục vụ) mà không cần tải lại. |
| **Đặt trước + quét QR** | Nếu bàn có lượt đặt trước trong khoảng −45 → +30 phút so với giờ hiện tại, lượt đó tự chuyển sang “Đã nhận bàn” và gắn vào đơn. |
| **Thanh toán** | “Hoàn tất” với đơn tại bàn nghĩa là đã lên món (không tự đánh dấu đã thanh toán). Nhân viên bấm **Đã nhận tiền** ở `/admin/orders` (bộ lọc “Tại bàn”) sau bữa ăn. |

Giới hạn cần biết: mã QR là công khai nên ai chụp được QR đều có thể đặt món; đơn luôn hiện ở màn hình bếp/Admin để nhân viên xác nhận, và phiên bàn tự hết hạn sau 4 giờ. Triển khai thật nên thêm giới hạn tần suất đặt đơn.

### Thanh toán & Đóng bàn · Hóa đơn gộp (DINE_IN)
Ở `/admin/tables`, chọn bàn **Đang ăn** → nút **Thanh toán & Đóng bàn** (kèm số tiền còn nợ; ô bàn trên sơ đồ cũng hiện số tiền chưa thanh toán, vd `250k`).

| Bước | Chi tiết |
|---|---|
| **Gom hóa đơn** | `GET /api/admin/tables/:id/bill` gom mọi đơn DINE_IN của bàn *chưa thanh toán và chưa huỷ* thành 1 hóa đơn tạm tính: từng đơn → từng món kèm size / topping / tùy chọn / ghi chú, tạm tính, giảm giá, **tổng thanh toán**. Hóa đơn tự cập nhật khi khách gọi thêm trong lúc đang thanh toán. |
| **Thanh toán** | **Tiền mặt** (nhập tiền khách đưa → tính tiền thừa, có nút số tiền gợi ý) hoặc **VietQR** (mã QR đúng số tiền, nội dung `Thanh toan ban A1`; tài khoản lấy từ Setting `bank.*`). |
| **Xác nhận** | `POST /api/admin/tables/:id/close` – **một transaction**: tạo `Invoice` (mã `HD-…`) → các đơn → **PAID + COMPLETED** (gắn `invoiceId`, ghi log) → món trong bếp → DONE → lượt đặt bàn đang ngồi → COMPLETED → bàn → **AVAILABLE**. Lỗi giữa chừng thì hoàn tác toàn bộ. |
| **Real-time** | Sau khi commit phát `order.updated` (từng đơn), `reservation.updated`, rồi `table.updated` qua `/api/kitchen/stream` → sơ đồ bàn đổi sang **Trống** và Bếp/Đơn hàng cập nhật ngay, không cần F5. |
| **Hóa đơn** | Sau khi đóng bàn có nút **In hóa đơn** (`/admin/invoices/<mã>?print=1`). Danh sách + in lại tại **Hóa đơn** (`/admin/invoices`). |

Các lớp an toàn khi thu tiền:
- Server đối chiếu danh sách đơn + tổng tiền mà thu ngân vừa xem; nếu khách gọi thêm hay đơn bị huỷ ở giữa → từ chối (409) và tải lại hóa đơn, không thu thiếu/sai.
- Bàn còn đơn **chưa lên món** → bắt buộc tick xác nhận trước khi đóng.
- Bàn còn đơn chưa thanh toán thì nút “Trả bàn trống” thủ công bị khóa (tránh mất tiền); bàn không có đơn nào vẫn trả bàn trống được, không tạo hóa đơn rỗng.
- Hình thức thanh toán thực tế được ghi vào các đơn (kể cả khi khách đã chọn khác lúc đặt) nên báo cáo theo thanh toán chính xác.

**Nâng cấp từ bản cũ:** chạy `npx prisma db push` (thêm bảng `Invoice` và cột `Order.invoiceId`). `npm run db:seed-demo` tạo sẵn bàn A5 đang ăn với 2 đơn chưa thanh toán để thử ngay.

### Phân quyền (RBAC), Người dùng, Mã giảm giá
| Vai trò | Thấy / vào được |
|---|---|
| **ADMIN** (`admin@restaurant.vn`) | chỉ **Quản lý người dùng** `/admin/users` (tạo / sửa / đổi mật khẩu / khóa tài khoản) |
| **MANAGER** (`manager@restaurant.vn`) | toàn bộ nghiệp vụ: Tổng quan, Sơ đồ bàn, Đơn hàng, Lịch đặt bàn, Hóa đơn, Món ăn, Danh mục, Tùy chọn món, **Mã giảm giá** `/admin/promotions`, **Báo cáo**, Màn hình bếp |
| **STAFF** (`staff@restaurant.vn`) | chỉ **Sơ đồ bàn**, **Đơn hàng**, **Lịch đặt bàn** `/admin/reservations` |

- Cấu hình ở `src/lib/roles.ts` (sidebar + hằng số quyền). Quyền được kiểm tra **ở server** (trang, API, Server Action, SSE), không chỉ ẩn menu; vào sai trang → chuyển về trang đầu của vai trò kèm thông báo.
- Phiên được đối chiếu DB ở mỗi request: **khóa tài khoản / đổi chức vụ có hiệu lực ngay**. Không thể tự khóa hay tự đổi chức vụ của chính mình.
- Mã giảm giá: loại % (có giảm tối đa) / số tiền / miễn phí ship, số lượng, thời gian (giờ VN), bật-tắt, hiện trang chủ. Mã đã dùng trong đơn hàng không xóa được (hãy tắt).

### Hoàn thiện cuối
- **Đặt bàn thủ công** (Quản lý + Nhân viên): nút **Tạo đặt bàn mới** ở `/admin/reservations` và `/admin/tables` (có cả “Đặt bàn cho bàn này” trong chi tiết bàn). Form tự gợi ý bàn vừa đủ chỗ còn trống theo ngày giờ + số người, hiện bàn đã bị đặt / không đủ chỗ, cho chọn tay hoặc “Chưa xếp bàn”; nhận giờ 10:00–21:30, có thể ghi nhận luôn “đã xác nhận”.
- **In hóa đơn**: STAFF mở/in được từ hộp thoại “Thanh toán & Đóng bàn”. Trang hóa đơn chọn khổ **Bill 80mm / 58mm / A5 / A4** (nhớ lựa chọn); bill cuộn tự đo chiều cao để tờ in dài đúng bằng nội dung. Địa chỉ/hotline trên bill lấy từ Setting `restaurant.address`, `restaurant.phone`.
- **Tài khoản**: `/login` (khách & nhân viên, bằng email hoặc SĐT) và `/profile` để **tự đổi mật khẩu** (mật khẩu hiện tại + mới + nhập lại). Sai mật khẩu quá nhiều lần bị khóa tạm 15 phút (đăng nhập: 8 lần, đổi mật khẩu: 5 lần).

### Đặt món online bắt buộc có tài khoản
- **Đăng ký** `/register`: họ tên, **tên đăng nhập**, **email**, **số điện thoại**, mật khẩu (≥ 8 ký tự, có chữ và số). Mỗi tên đăng nhập / email / SĐT chỉ dùng cho **một** tài khoản; đăng ký xong tự đăng nhập. Đăng nhập `/login` bằng tên đăng nhập, email hoặc SĐT (demo: `customer` / `Password@123`).
- **Checkout** (giao hàng / mang đi): chưa đăng nhập → hiện màn hình đăng nhập/đăng ký (giỏ hàng được giữ). Tên + SĐT của đơn **lấy từ tài khoản**, không cho tự nhập; server từ chối đơn không có tài khoản (401) và bỏ qua mọi tên/SĐT do client gửi. Đơn gắn `customerId`, xem lại ở `/profile` → “Đơn hàng của tôi”.
- Gọi món tại bàn (QR) vẫn cho khách vãng lai. Nâng cấp từ bản cũ: `npx prisma db push` (thêm cột `User.username`) rồi `npm run db:seed` để thêm tên đăng nhập demo.

### Dữ liệu demo cho báo cáo & sơ đồ bàn
```bash
npm run db:seed-demo     # ~30 ngày đơn hàng (mã DM-…), đặt bàn mẫu, bàn đang ăn/đang dọn. Chạy lại được nhiều lần
```

### Real-time (SSE)
- `GET /api/kitchen/stream` đẩy `order.created`, `order.updated`, `reservation.*`, `table.updated`.
- Khách bấm "Đặt hàng" → `createOrder()` phát sự kiện → **Bếp nhận đơn + phát âm báo ngay**, Admin cũng tự cập nhật.
- Mọi thay đổi trạng thái đơn đi qua **một** máy trạng thái duy nhất (`src/lib/order-actions.ts`), ghi `OrderStatusLog` → trang tracking của khách tự nhảy bước.
- Bus sự kiện nằm trong bộ nhớ (`src/lib/events.ts`): đúng cho `next dev`/`next start` 1 tiến trình. Triển khai nhiều instance/serverless cần thay bằng Redis Pub/Sub.

### Đặt bàn: cách chọn bàn
Hệ thống tự gán **bàn nhỏ nhất còn đủ chỗ** và còn trống trong khung giờ (ngồi 90 phút, nhận đặt 10:00–20:30, trước ≥30 phút, tối đa 30 ngày). Đặt xong ở trạng thái "Chờ xác nhận", nhân viên xác nhận / đánh dấu khách đến ở `/admin/tables`.

### Chưa làm (gợi ý bước sau)
- Gọi món tại bàn (QR mỗi bàn → đơn `DINE_IN`; bếp đã hỗ trợ hiển thị).
- CRUD đầy đủ món ăn / bàn / mã giảm giá / người dùng trong Admin (hiện chỉ bật-tắt món).
- Giới hạn số lần đăng nhập sai, đổi mật khẩu; đối soát chuyển khoản tự động qua webhook ngân hàng.
- Ứng dụng riêng cho shipper.

## Ghi chú kỹ thuật
- **Real-time không cần WebSocket server:** Next.js không chạy Socket.IO tốt ở App Router, nên Kitchen Display và Order Tracking dùng **Server-Sent Events** (`/api/kitchen/stream`) hoặc polling 3s. Với demo là đủ mượt; vẫn đúng tinh thần "nhận đơn real-time".
- **Bản đồ & phí ship (demo):** dùng Leaflet + OpenStreetMap/Nominatim (miễn phí, không cần API key) và tính khoảng cách Haversine/OSRM ở `/api/shipping/quote`.
- **Tiền VND dùng `Int`**; `OrderItem` lưu snapshot tên/giá; `OrderStatusLog` là nguồn cho timeline tracking.
- Font Inter + Playfair Display tải qua `next/font` nên cần Internet lần build/dev đầu.
