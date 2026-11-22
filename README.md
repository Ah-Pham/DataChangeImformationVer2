# DataExchange Frontend

Frontend thuần HTML, CSS, JavaScript cho website lưu trữ văn bản, file và thư mục qua REST API C# ASP.NET Core.

## 1. Chạy thử

1. Giải nén thư mục.
2. Mở `app.js`.
3. Sửa `API_BASE` thành địa chỉ Backend ASP.NET Core của bạn, ví dụ `https://localhost:7001`.
4. Chạy Backend.
5. Mở `index.html` bằng trình duyệt (khuyên dùng extension Live Server trong VS Code).

> Đăng nhập và các thao tác dữ liệu chỉ hoạt động khi Backend đã được xây dựng đúng API contract bên dưới. Giao diện không tự lưu dữ liệu vào SQL Server.

## 2. API contract Frontend đang sử dụng

### Đăng nhập

`POST /api/auth/login`

Request:
```json
{
  "username": "demo",
  "password": "123456"
}
```

Response thành công:
```json
{
  "token": "JWT_ACCESS_TOKEN"
}
```

Frontend lưu token trong `localStorage`, sau đó gửi header:
`Authorization: Bearer JWT_ACCESS_TOKEN`

### Lấy danh sách dữ liệu

`GET /api/data`

Response:
```json
[
  {
    "id": 1,
    "title": "Ghi chú",
    "content": "Nội dung văn bản cần lưu",
    "createdAt": "2026-11-12T10:00:00",
    "updatedAt": null
  }
]
```

API cũng có thể trả `{ "data": [...] }`.

### Thêm dữ liệu

`POST /api/data`

Request:
```json
{
  "title": "Ghi chú",
  "content": "Nội dung văn bản cần lưu"
}
```

### Cập nhật dữ liệu

`PUT /api/data/{id}`

Request giống API thêm dữ liệu.

### Xóa dữ liệu

`DELETE /api/data/{id}`

Trả `204 No Content` hoặc một response 2xx khác.

## 3. Gợi ý DTO C# tương ứng

```csharp
public class LoginRequest
{
    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
}

public class LoginResponse
{
    public string Token { get; set; } = "";
}

public class DataRequest
{
    public string Title { get; set; } = "";
    public string Content { get; set; } = "";
}

public class DataResponse
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Content { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}
```

ASP.NET Core mặc định thường serialize tên thuộc tính JSON theo camelCase (`CreatedAt` thành `createdAt`), khớp với Frontend.

## 4. CORS trong ASP.NET Core

Nếu Frontend và Backend chạy khác origin/port, hãy bật CORS trong Backend. Ví dụ trong `Program.cs`:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("FrontendPolicy", policy =>
    {
        policy
            .WithOrigins("http://127.0.0.1:5500", "http://localhost:5500")
            .AllowAnyHeader()
            .AllowAnyMethod();
    });
});

// Sau khi build app:
app.UseCors("FrontendPolicy");
app.UseAuthentication();
app.UseAuthorization();
```

Thứ tự middleware cần phù hợp với cấu hình ứng dụng của bạn. Không dùng `AllowAnyOrigin` kèm credentials.

## 5. Lưu ý bảo mật

- Chỉ triển khai thật qua HTTPS.
- Backend phải xác thực JWT và kiểm tra quyền sở hữu dữ liệu ở mọi API; không chỉ dựa vào việc Frontend ẩn/hiện giao diện.
- Không lưu mật khẩu người dùng ở Frontend.
- `localStorage` chỉ là cách lưu token đơn giản cho bản mẫu. Với ứng dụng production, cân nhắc cookie `HttpOnly`, `Secure`, `SameSite` và cơ chế chống CSRF phù hợp.
- Nội dung text được hiển thị dưới dạng text đã escape HTML để tránh chèn mã HTML/JavaScript.


## 6. API contract cho file và thư mục

Frontend đã có giao diện quản lý file/thư mục. Bạn cần tự xây dựng các API dưới đây trong Backend. Tất cả API (trừ đăng nhập) phải nhận header `Authorization: Bearer <token>`.

### Quy ước dữ liệu

Thư mục:
```json
{
  "id": 10,
  "name": "Tai lieu",
  "parentId": null
}
```

File:
```json
{
  "id": 25,
  "name": "report.pdf",
  "size": 245760,
  "contentType": "application/pdf",
  "folderId": 10,
  "createdAt": "2026-11-22T10:00:00"
}
```

Các danh sách có thể trả về mảng trực tiếp hoặc `{ "data": [...] }`.

### Danh sách thư mục

`GET /api/folders`

Trả về danh sách thư mục của tài khoản hiện tại. Nên trả cả `parentId` để Frontend dựng đường dẫn thư mục.

### Tạo thư mục

`POST /api/folders`

Request:
```json
{
  "name": "Tai lieu",
  "parentId": null
}
```

### Đổi tên thư mục

`PUT /api/folders/{id}`

Request:
```json
{ "name": "Tai lieu moi" }
```

### Xóa thư mục

`DELETE /api/folders/{id}`

Backend cần quyết định rõ chính sách khi thư mục còn file hoặc thư mục con (ví dụ: từ chối xóa nếu chưa trống). Không được xóa nhầm dữ liệu của tài khoản khác.

### Danh sách file trong thư mục

`GET /api/files`

Ở thư mục gốc, Frontend gọi không kèm query. Trong thư mục, Frontend gọi `GET /api/files?folderId={id}`.

### Upload file

`POST /api/files/upload`

Frontend gửi `multipart/form-data` gồm:
- `file`: file cần tải lên.
- `folderId`: ID thư mục hiện tại (không gửi nếu đang ở thư mục gốc).

Không đặt header `Content-Type` thủ công cho request này; trình duyệt tự tạo multipart boundary.

### Đổi tên file

`PUT /api/files/{id}`

Request:
```json
{ "name": "report-new.pdf" }
```

### Xóa file

`DELETE /api/files/{id}`

### Tải file xuống

`GET /api/files/{id}/download`

Backend trả nội dung file (binary) và nên đặt `Content-Disposition` phù hợp để tải xuống.

### Lưu ý khi bạn viết Backend

- Xác thực JWT và kiểm tra quyền sở hữu file/thư mục ở **từng endpoint**.
- Không tin `folderId`, `fileId` hoặc tên file do Frontend gửi lên; phải kiểm tra quyền và chuẩn hóa tên file ở Backend.
- Giới hạn dung lượng file, loại file nếu cần; không cho phép đường dẫn do người dùng tự nhập để tránh path traversal.
- Lưu file vật lý ở thư mục do Backend quản lý; Database chỉ lưu metadata và đường dẫn nội bộ.
- Với xóa thư mục, hãy chọn chính sách rõ ràng cho file/thư mục con.
- Cấu hình CORS cho đúng origin của trang Frontend.

> Lưu ý: các endpoint file/thư mục ở trên là hợp đồng API đề xuất để Frontend gọi. Chúng chưa tự xuất hiện trong Backend của bạn; bạn cần triển khai chúng tương ứng.
