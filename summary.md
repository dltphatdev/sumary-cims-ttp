# Bảng Kế Hoạch Dự Án CIMS-TTP

## Môi trường Production cims.ttptelecom.vn

## Môi trường Development cimsdev.ttptelecom.vn

## Các điều kiện cần để thực hiện dự án trên subdomain được cung cấp bởi cty

1/ Các thông tin cần thiết VPS Server:

- Địa chỉ Ipv4 của server
- Hệ điều hành của server (Ubuntu)
- Quyền để truy cập vào server (username/password/SSH key)

2/ Cấu hình DNS cho subdomain: Record loại CNAME

- Trỏ subdomain cims.ttptelecom.vn và cimsdev.ttptelecom.vn về địa chỉ IP hoặc hosting server

3/ Chuẩn bị lưu lượng ổ cứng để lưu trữ các file upload ảnh của user sau này: (optional)

- 1000 user account sẽ ước tính tầm 1.5GB

4/ Mở setting mở các port cần thiết để có thể kết nối

- Port: 22 để client có thể connect tới server
- Port 443: để có thể config được Nginx và dùng được chứng chỉ SSL

## Công nghệ sử dụng trong dự án

1/ Backend API: ExpressJS + Typescript

2/ Frontend: ReactJS + Typescript

3/ Database: Prismas + SQL Lite

## Thiết Schema Database

### Bảng User

```prisma
enum UserVerifyStatus {
  Unverified,
  Verified,
  Banned
}

enum UserType {
  SuperAdmin
  Admin
  Sale
}

model User {
  id                    Int            @id @default(autoincrement())
  email                 String         @unique
  password              String
  fullname              String?
  verify                UserVerifyStatus @default(Unverified)
  avatar                String?
  address               String?
  phone                 String?
  code                  String?
  refreshTokens         RefreshToken[]
  date_of_birth         DateTime?
  created_at            DateTime       @default(now())
  updated_at            DateTime       @updatedAt
  type                  UserType       @default(Sale)
  roles                 Role[]           @relation("UserRoles")
  createdCustomers   Customer[] @relation("CreatedCustomers")
  consultedCustomers Customer[] @relation("ConsultedCustomers")
  effective Effective[]  // Một user tạo nhiều hiệu quả
}
```

- Với type trong bảng User dùng để xác định xem account này đang là account: SuperAdmin hay là Admin hay là account Sale
- Với account là Supprer Admin
- Với account là Supper Admin và account Admin cho phép cấp quyền cho các user còn lại
- Với account là sale chỉ được phép xem danh sách thành viên
- Tất cả tài khoản ban đầu ngoại trừ Supper Admin thì đều là tài khoản Sale

### Bảng RefreshToken

```prisma
model RefreshToken {
  id      Int      @id @default(autoincrement())
  token   String   @unique
  user_id Int
  iat     DateTime
  exp     DateTime
  user User @relation(fields: [user_id], references: [id])
  @@index([token])
  @@index([exp])
}
```

### Bảng khách hàng

```prisma
enum CustomerType {
  Business
  Personal
}

enum CustomerStatus {
  Active
  UnActive
}

model Customer {
  id           Int       @id @default(autoincrement())
  fullName     String
  cccd_or_tax    String
  phone        String
  address      String
  type         CustomerType @default(Personal)
  status       CustomerStatus @default(UnActive)
  note        String?
  creator_id  Int
  consultant_id Int
  created_at            DateTime       @default(now())
  updated_at            DateTime       @updatedAt
  creator      User      @relation("CreatedCustomers", fields: [creator_id], references: [id])
  consultant   User      @relation("ConsultedCustomers", fields: [consultant_id], references: [id])
  effective Effective[]  // Một user tạo nhiều hiệu quả
}
```

### Bảng quyền

```prisma
model Role {
  id    Int     @id @default(autoincrement())
  name  String  @unique // tên của quyền
  action String @unique // hành động của
  users User[]  @relation("UserRoles")
}
```

- Theo bảng thiết kế sẽ có 4 quyền chính:

1. Quyền chỉnh sửa: cho phép account chỉnh sửa khách hàng của mình
2. Quyền phân bổ: chỉ có admin và supper admin mới có thể phân bổ
3. Quyền thu hồi: chỉ có admin và supper admin mới có quyền thu hồi
4. Quyền xác minh: chỉ có admin và supper admin mới có quyền xác minh

- Lưu ý:

* Sale chỉ nhìn thấy được các khách hàng mà admin phân bổ cho mình và các khách hàng mà mình tạo ra
* Khi bị thu hồi khách hàng thì dữ liệu của khách hàng đó trên account bị thu hồi sẽ không còn
* Khi sale tạo ra khách hàng mới thì sale mặc định sẽ được gán vào mục kinh doanh phân bổ
* Khi admin tạo ra khách mới thì mặc định mục kinh doanh phân bổ là trống

### Bảng hiệu quả

```prisma
enum EffectiveStatus {
  New // Mới
  Approved // Đã duyệt
  Canceled // Đã hủy
}
model Effective {
  id    Int     @id @default(autoincrement())
  name String
  customer_id      Int
  creator_id          Int
  status          EffectiveStatus // Trang thai hiệu quả công việc
  note            String?            // ghi chu
  created_at            DateTime       @default(now())
  updated_at            DateTime       @updatedAt
  customer        Customer   @relation(fields: [customer_id], references: [id])
  creator            User       @relation(fields: [creator_id], references: [id])
  revenue         Revenue[]
}
```

### Bảng doanh thu (trước VAT)

```prisma
enum RevenueType {
  OneTime // 1 lần
  OneMonth //  1 tháng
  OneYear // 1 năm
}

model Revenue {
  id             Int          @id @default(autoincrement())
  effective_id       Int
  category           String       // Hạng mục
  type           RevenueType  @default(OneTime) // Loại (1 lần / hàng tháng / hàng năm)
  description    String       // Diễn giải
  unit           String       // Đơn vị tính
  price      Decimal      @db.Decimal(20, 2) // Đơn giá
  quantity       Int
  created_at            DateTime       @default(now())
  updated_at            DateTime       @updatedAt
  effective         Effective       @relation(fields: [effective_id], references: [id])
}
```

### Bảng chi phí

```prisma
model ExpenseRate {
  id               Int      @id @default(autoincrement())
  effective_id      Int      @unique // moi quan he voi ban hieu qua
  operation_rate    Float   // Chi phí vận hành %
  cskh_rate         Float    // Chi phí CSKH %
  commission_rate    Float    // Chi phí hoa hồng %
  diplomatic_expense_rate           Float    // Chi phí ngoại giao %
  customer_rate     Float    // Chi phí khách hàng %
  reserve_rate    Float    // Chi phí dự phòng %
  created_at            DateTime       @default(now())
  updated_at            DateTime       @updatedAt
  effective        Effective @relation(fields: [effective_id], references: [id])
}
```

### Các Task sẽ thực hiện và thời gian dự kiến của từng Task:

1. Thiết kế Database với SQL Lite: (đã hoàn thành)

- Thời gian bắt đầu nhận: 3/6/2025
- Thời gian dự kiến hoàn thành: cuối ngày 3/6/2025 (1 ngày)

2. Backend:

- User API (Create, Update, Read):

* Thời gian bắt đầu nhận: 4/6/2025
* Thời gian dự kiến hoàn thành: cuối ngày 6/6/2025 (tầm 3 ngày)

3. Frontend:

- User UI và xử lý API trả về:

* Thời gian bắt đầu nhận: 7/6/2025
* Thời gian dự kiến hoàn thành: cuối ngày 10/6/2025 (tầm 3 ngày)
