## 1. Biểu đồ tuần tự

### 1.1 Biểu đồ tuần tự chức năng đăng ký

```mermaid
sequenceDiagram
    actor Guest as Khách truy cập
    participant Controller
    participant Service
    participant Repository
    participant Redis
    participant Notification

    Guest->>Controller: Truy cập trang đăng ký
    Controller-->>Guest: Hiển thị form đăng ký

    Guest->>Controller: Gửi form (Họ tên, Email, Mật khẩu, Xác nhận)
    Controller->>Service: validateInput(data)

    alt Dữ liệu không hợp lệ (A1)
        Service-->>Controller: Trả lỗi validation
        Controller-->>Guest: Hiển thị lỗi + giữ dữ liệu (trừ mật khẩu)
    else Dữ liệu hợp lệ
        Service->>Repository: existsByEmail(email)

        alt Email đã tồn tại (A2)
            Repository-->>Service: true
            Service-->>Controller: Lỗi: email đã tồn tại
            Controller-->>Guest: "Email này đã được đăng ký..."
        else Email chưa tồn tại
            Service->>Service: Mã hóa mật khẩu
            Service->>Repository: Lưu tài khoản (trạng thái: chưa xác thực, vai trò: Ứng viên)
            Service->>Redis: Tạo & lưu token (TTL 30 phút)

            Service->>Notification: sendEmail(email, "Xác thực tài khoản", activationToken)

            alt Gửi email thất bại (A3)
                Notification-->>Service: Lỗi gửi email
                opt Thực hiện rollback
                    Service->>Repository: Xóa tài khoản tạm
                    Service->>Redis: Xóa token
                end
                Service-->>Controller: Lỗi hệ thống gửi thông báo
                Controller-->>Guest: "Đã có lỗi... Vui lòng thử lại sau."
            else Gửi email thành công
                Notification-->>Service: Thành công
                Service-->>Controller: Đăng ký thành công
                Controller-->>Guest: Chuyển hướng → "Vui lòng kiểm tra email..."
            end
        end
    end
```


### 1.2. Biểu đồ tuần tự chức năng đăng tin tuyển dụng

```mermaid
sequenceDiagram
    actor Recruiter as Nhà Tuyển dụng
    participant Controller
    participant Service
    participant Repository
    participant Notification

    Recruiter->>Controller: Chọn "Đăng tin mới"
    Controller->>Service: checkPrerequisites(recruiterId)

    alt Không đủ điều kiện (A1)
        Service-->>Controller: Lỗi: chưa liên kết công ty / hết lượt đăng
        Controller-->>Recruiter: Hiển thị thông báo lỗi
    else Đủ điều kiện
        Controller-->>Recruiter: Hiển thị form đăng tin

        Recruiter->>Controller: Gửi form (hoặc chọn "Lưu nháp")
        alt Lưu nháp (A4)
            Controller->>Service: validateDraft(title)
            Service->>Repository: Lưu tin (trạng thái: Draft)
            Service-->>Controller: Thành công
            Controller-->>Recruiter: "Đã lưu nháp thành công."
        else Gửi đi (gửi duyệt)
            Controller->>Service: validateJobPost(data)
            alt Dữ liệu không hợp lệ (A2)
                Service-->>Controller: Trả lỗi validation
                Controller-->>Recruiter: Hiển thị lỗi + giữ dữ liệu
            else Dữ liệu hợp lệ
                Service->>Repository: Bắt đầu giao dịch
                Service->>Repository: Lưu tin (trạng thái: Pending)

                Service->>Notification: sendWebSocket(adminChannel, "Có tin tuyển dụng mới cần duyệt")

                alt Lỗi hệ thống (A3)
                    Notification-->>Service: Lỗi gửi thông báo hoặc lưu CSDL
                    opt Rollback giao dịch
                        Service->>Repository: Hủy lưu tin
                    end
                    Service-->>Controller: Lỗi hệ thống
                    Controller-->>Recruiter: "Đã có lỗi xảy ra. Vui lòng thử lại sau."
                else Thành công
                    Service-->>Controller: Đăng tin thành công
                    Controller-->>Recruiter: Chuyển hướng → "Tin đang chờ phê duyệt."
                end
            end
        end
    end
```

### 1.3. Biểu đồ tuần tự chức năng ứng tuyển

```mermaid
sequenceDiagram
    actor Candidate as Ứng viên
    participant Controller
    participant Service
    participant Repository
    participant Notification

    Candidate->>Controller: Nhấn "Ứng tuyển ngay" trên tin tuyển dụng
    Controller->>Service: checkEligibility(candidateId, jobId)

    alt Chưa có CV (A1)
        Service-->>Controller: Lỗi: chưa có CV
        Controller-->>Candidate: "Vui lòng tải lên CV trước khi ứng tuyển."
    else Đã ứng tuyển rồi (A2)
        Service-->>Controller: Lỗi: đã ứng tuyển
        Controller-->>Candidate: "Bạn đã ứng tuyển vào vị trí này."
    else Đủ điều kiện
        Controller-->>Candidate: Hiển thị modal: chọn CV + (tùy chọn) thư xin việc

        Candidate->>Controller: Gửi lựa chọn (CV, cover letter)
        Controller->>Service: validateApplication(cvSelected)

        alt CV chưa chọn (dữ liệu không hợp lệ)
            Service-->>Controller: Lỗi: vui lòng chọn CV
            Controller-->>Candidate: Hiển thị lỗi trong modal
        else Hợp lệ
            Service->>Repository: Bắt đầu giao dịch
            Service->>Repository: Tạo đơn ứng tuyển (trạng thái: Submitted)

            Service->>Notification: sendWebSocket(recruiterId, "Có ứng viên mới ứng tuyển")
            Service->>Notification: sendEmail(recruiterEmail, "Thông báo ứng tuyển mới")

            alt Lỗi hệ thống (A3)
                Notification-->>Service: Lỗi gửi thông báo hoặc lưu CSDL
                opt Rollback giao dịch
                    Service->>Repository: Hủy tạo đơn ứng tuyển
                end
                Service-->>Controller: Lỗi hệ thống
                Controller-->>Candidate: "Đã có lỗi xảy ra. Vui lòng thử lại sau."
            else Thành công
                Service-->>Controller: Ứng tuyển thành công
                Controller-->>Candidate: Hiển thị thông báo: "Ứng tuyển thành công!"
            end
        end
    end
```

### 1.4. Biểu đồ tuần tự chức năng quản lý quy trình ứng tuyển

```mermaid
sequenceDiagram
    actor Recruiter as Nhà Tuyển dụng
    participant Controller
    participant Service
    participant Repository
    participant Notification

    Recruiter->>Controller: Truy cập "Quản lý ứng viên", chọn tin tuyển dụng
    Controller->>Service: getApplicationsByJob(jobId, recruiterId)

    alt Không có quyền hoặc tin không tồn tại
        Service-->>Controller: Lỗi: không tìm thấy hoặc không thuộc quyền
        Controller-->>Recruiter: Hiển thị lỗi
    else Có dữ liệu
        Service->>Repository: Truy vấn danh sách ứng viên cho tin
        Repository-->>Service: Trả danh sách (tên, ngày nộp, trạng thái)
        Service-->>Controller: Danh sách ứng viên
        Controller-->>Recruiter: Hiển thị danh sách

        Recruiter->>Controller: Nhấp vào một ứng viên
        Controller->>Service: getApplicationDetail(applicationId, recruiterId)

        alt Đơn không tồn tại hoặc không thuộc NTD (A1)
            Service-->>Controller: Lỗi truy cập
            Controller-->>Recruiter: "Không tìm thấy đơn ứng tuyển."
        else Hợp lệ
            Service->>Repository: Lấy chi tiết: hồ sơ, CV, thư xin việc
            Repository-->>Service: Trả dữ liệu chi tiết
            Service-->>Controller: Dữ liệu chi tiết
            Controller-->>Recruiter: Hiển thị CV, thư xin việc, trạng thái

            Recruiter->>Controller: Chọn trạng thái mới + nhấn "Cập nhật"
            Controller->>Service: updateApplicationStatus(applicationId, newStatus)

            alt Trạng thái không hợp lệ (A2)
                Service-->>Controller: Lỗi: trạng thái không được phép
                Controller-->>Recruiter: "Trạng thái này không hợp lệ."
            else Hợp lệ
                Service->>Repository: Bắt đầu giao dịch
                Service->>Repository: Cập nhật trạng thái đơn ứng tuyển

                Service->>Notification: sendWebSocket(candidateId, "Trạng thái hồ sơ của bạn đã được cập nhật")
                Service->>Notification: sendEmail(candidateEmail, "Cập nhật trạng thái ứng tuyển")

                alt Lỗi hệ thống (A3)
                    Notification-->>Service: Lỗi gửi thông báo hoặc lưu CSDL
                    opt Rollback giao dịch
                        Service->>Repository: Hoàn tác cập nhật trạng thái
                    end
                    Service-->>Controller: Lỗi hệ thống
                    Controller-->>Recruiter: "Đã có lỗi xảy ra. Vui lòng thử lại sau."
                else Thành công
                    Service-->>Controller: Cập nhật thành công
                    Controller-->>Recruiter: Cập nhật giao diện → hiển thị trạng thái mới
                end
            end
        end
    end
```

## 2. Biểu đồ hoạt động

### 2.1. Biểu đồ hoạt động chức năng đăng ký
![alt text](image-1.png)

### 2.2. Biểu đồ hoạt động chức năng đăng tin
![alt text](image.png)

### 2.3. Biểu đồ hoạt động chức năng quản lý tin tuyển dụng
![alt text](image-2.png)

### 2.4. Biểu đồ hoạt động chức năng quản lý danh sách ứng viên
![alt text](image-3.png)