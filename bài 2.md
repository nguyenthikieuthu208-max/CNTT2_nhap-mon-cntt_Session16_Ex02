


# Bài 2: Sự cố phiên làm việc trên hệ thống vận tải

## Nguyên nhân sự cố
Hệ thống đang dùng **Session Cookie** (không có `Max-Age`/`Expires`). Loại cookie này chỉ nằm trong bộ nhớ trình duyệt và **bị xóa khi đóng trình duyệt**, nên shipper phải đăng nhập lại và giỏ hàng bị mất.

**Giải pháp:** dùng **Persistent Cookie** (có thời hạn sống) để ghi nhớ phiên, kết hợp cờ `HttpOnly` và `Secure` để bảo mật.

---

## Phần 1. Phân tích

### Session Cookie và Persistent Cookie
| Tiêu chí | Session Cookie | Persistent Cookie |
|---|---|---|
| Thời hạn | Hết khi đóng trình duyệt | Tồn tại đến khi hết `Max-Age`/`Expires` |
| Lưu ở đâu | Bộ nhớ RAM của trình duyệt | Lưu xuống ổ đĩa |
| Phù hợp | Phiên ngắn, máy dùng chung | Ghi nhớ đăng nhập, giữ giỏ hàng |

### Đánh đổi giữa Cache (MB/GB) và Cookies (~4KB)
| Tiêu chí | Cookies | Cache |
|---|---|---|
| Dung lượng | Rất nhỏ (~4KB mỗi cookie) | Lớn (MB đến GB) |
| Nội dung lưu | Dữ liệu định danh nhỏ: token phiên, tùy chọn | Tài nguyên tĩnh: CSS, JS, hình ảnh |
| Gửi lên server | **Có**, tự động đính kèm trong **mọi request** | **Không** gửi, đọc trực tiếp từ máy |
| Ảnh hưởng hiệu năng | Cookie quá lớn làm tăng băng thông, chậm request | Giúp tải trang nhanh, giảm tải server |
| Rủi ro | Bị đánh cắp nếu thiếu cờ bảo mật | Hiển thị **dữ liệu cũ** (stale) nếu không làm mới |

**Kết luận:** Cookie chỉ nên lưu **một token định danh nhỏ**. Dữ liệu lớn (giao diện, hình ảnh) để cho Cache. Giỏ hàng nên lưu **trên server**, gắn với token, vì cookie quá nhỏ và không an toàn để chứa dữ liệu lớn.

---

## Phần 2. Triển khai

### Cấu trúc mẩu tin Cookie (HTTP Response Header)
```http
Set-Cookie: session_token=a8f3c9e1b7d24f6a9c0e5b31d8f7a2c4; Max-Age=604800; Path=/; Secure; HttpOnly; SameSite=Strict
```

### Giải thích từng thành phần
| Thành phần | Ý nghĩa |
|---|---|
| `session_token=...` | **Token** ngẫu nhiên, khó đoán, server dùng để nhận diện shipper và tìm lại giỏ hàng. Không chứa mật khẩu hay thông tin nhạy cảm. |
| `Max-Age=604800` | Sống **7 ngày** (604800 giây). Đây là điểm biến Session Cookie thành **Persistent Cookie**. |
| `Path=/` | Cookie có hiệu lực trên toàn bộ website. |
| `Secure` | Chỉ gửi qua **HTTPS**, chống nghe lén trên đường truyền. |
| `HttpOnly` | **JavaScript không đọc được** (`document.cookie`), chống XSS đánh cắp token. |
| `SameSite=Strict` | Không gửi cookie khi request đến từ trang web khác, giảm nguy cơ CSRF. |

### Đáp ứng ràng buộc
Nhờ cờ `HttpOnly`, mã JavaScript phía Client **không thể
