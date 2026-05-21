

## EVIDENCE MH-OBS

```markdown
# MH-OBS — Monitoring and Observability

## 1. Luồng hoạt động (Observability Pipeline Workflow)


Upload ảnh sản phẩm vào S3 bucket
          ↓
S3 Event Trigger (ObjectCreated) kích hoạt Lambda
          ↓
Lambda xử lý metadata của ảnh (kích thước, tên file, định dạng...)
          ↓
Lambda ghi thông tin metadata vào DynamoDB table
          ↓
Lambda đẩy dữ liệu Custom Metric (số lượng upload, dung lượng file) lên CloudWatch
          ↓
CloudWatch Dashboard hiển thị biểu đồ trực quan
          ↓
CloudWatch Alarm cảnh báo khi có sự kiện upload vượt ngưỡng
          ↓
CloudWatch Logs Insights giúp truy vết và xử lý sự cố (troubleshooting)
```

### Lý do chọn luồng này cho dự án
Vì ứng dụng của chúng tôi là một ứng dụng thương mại điện tử, việc đăng tải ảnh sản phẩm (media upload) là một tính năng cốt lõi. Thiết lập observability trên luồng này giúp nhóm nắm được:
- Số lượng ảnh sản phẩm được tải lên theo thời gian thực.
- Dung lượng lưu trữ phát sinh.
- Các lỗi phát sinh trong quá trình xử lý ảnh thông qua Logs, Dashboard và Alarm.

---

## MH-OBS 1 — Tạo DynamoDB table

Nhóm đã tạo một bảng DynamoDB để lưu trữ metadata của các file ảnh được upload lên S3.

### Các bước thực hiện:
1. Truy cập **DynamoDB Console** -> Chọn **Tables** ở menu trái -> Click **Create table**.
2. Điền thông tin cấu hình:
   * **Table name:** `minie-metadata`
   * **Partition key:** `mediaId` (Kiểu dữ liệu: `String`).
     > **Giải thích:** partition key đóng vai trò là khóa chính duy nhất để định danh cho từng bản ghi metadata của file ảnh trong DynamoDB, cho phép truy xuất nhanh chóng theo ID mà không cần quét toàn bộ bảng.
   * **Table settings:** Chọn **Default settings** (Sử dụng dung lượng Provisioned capacity mặc định).
3. **Cấu hình Tags (MH-COST-V):**
   * Cuộn xuống mục **Tags** ở gần cuối trang -> Click **Add new tag** và nhập 4 tag bắt buộc để phục vụ việc phân loại và giám sát chi phí:
     * **Key:** `Owner` | **Value:** `quochiep1610@gmail.com`
     * **Key:** `Environment` | **Value:** `production`
     * **Key:** `CostCenter` | **Value:** `G4`
     * **Key:** `Application` | **Value:** `mini-e`
4. Bấm **Create table** và đợi bảng chuyển sang trạng thái `Active`.

![DynamoDB Table Config](./Evidence/dynamodb_table_config.png)
![Dynamo DB Tag config](./Evidence/dynamodb_tag.png)

*Ý nghĩa hình ảnh:* Hình ảnh trên chứng minh bảng DynamoDB `minie-metadata` đã được tạo thành công với cấu hình Partition key `mediaId` và được gắn đầy đủ bộ tag Cost Allocation theo quy định của Workshop.

---

## MH-OBS 2 — Tạo IAM Role cho Lambda OBS

Để Lambda có quyền đọc ghi log vào CloudWatch và lưu metadata vào DynamoDB, nhóm đã tạo một IAM Role với policy giới hạn quyền truy cập.

### Các bước thực hiện:
1. Truy cập **IAM Console** -> Chọn **Roles** ở menu trái -> Click **Create role**.
2. Chọn Trusted entity type: **AWS service** -> Chọn Use case: **Lambda** -> Bấm **Next**.
3. Ở phần **Add permissions**, tìm kiếm và tick chọn policy: `AWSLambdaBasicExecutionRole` -> Bấm **Next**.
4. Thiết lập tên Role: `lambdaRole-obs-w6` -> Bấm **Create role**.
5. Đính kèm **Inline Policy** (quyền ghi DynamoDB và gửi Custom Metric):
   * Click vào Role `lambdaRole-obs-w6` vừa tạo.
   * Tại tab **Permissions**, click **Add permissions** -> Chọn **Create inline policy**.
   * Chuyển sang chế độ nhập **JSON** và dán đoạn chính sách sau:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "dynamodb:PutItem",
                    "dynamodb:GetItem"
                ],
                "Resource": "arn:aws:dynamodb:us-east-1:683065762519:table/minie-metadata"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "cloudwatch:PutMetricData"
                ],
                "Resource": "*"
            }
        ]
    }
    ```
   * Nhấn **Next**. Đặt tên policy: `LambdaOBSPolicy` -> Click **Create policy**.

![IAM Role Policy](./Evidence/iam_role_policy.png)

*Ý nghĩa hình ảnh:* IAM Role `lambdaRole-obs-w6` đã được cấu hình các quyền ghi log vào CloudWatch, ghi dữ liệu vào bảng DynamoDB `minie-metadata` và xuất bản Metric lên CloudWatch.

---

## MH-OBS 3 — Tạo Lambda OBS & Cấu hình Environment Variables

Nhóm tạo một Lambda function xử lý sự kiện S3 và ghi nhận dữ liệu vào DynamoDB & CloudWatch.

### Các bước thực hiện:
1. Truy cập **Lambda Console** -> Click **Create function** -> Chọn **Author from scratch**.
2. Thiết lập thông số:
   * **Function name:** `lambda-media-obs`
   * **Runtime:** Chọn `Node.js 22.x` hoặc `Node.js 20.x`.
   * **Execution role:** Chọn **Use an existing role** -> chọn Role `lambdaRole-obs-w6` đã tạo ở Bước 2.
3. Click **Create function**.
4. Cấu hình biến môi trường (**Environment Variables**):
   * Chuyển sang tab **Configuration** -> Chọn mục **Environment variables** ở menu bên trái -> Click **Edit** -> Chọn **Add environment variable**.
   * Thêm các biến sau để code có thể đọc động:
     * **`TABLE_NAME`** = `minie-metadata`
     * **`REGION`** = `us-east-1`
     > **Giải thích:** Giúp tách biệt cấu hình và code. Khi thay đổi bảng DynamoDB hoặc Region, ta chỉ cần đổi biến môi trường trên Console mà không cần deploy lại code.
5. Triển khai mã nguồn:
   * Chuyển sang tab **Code**, thay thế nội dung file `index.mjs` bằng code xử lý S3 Event, ghi DynamoDB và gửi Custom Metric lên CloudWatch.
   * Nhấn **Deploy** để kích hoạt code mới.

![Lambda Environment Variables](./Evidence/lambda_env_vars.png)

*Ý nghĩa hình ảnh:* Cấu hình các biến môi trường cho Lambda function giúp tách biệt thông tin bảng DynamoDB (`minie-metadata`) ra khỏi mã nguồn để dễ dàng quản lý và cấu hình.

---

## MH-OBS 4 — Cấu hình S3 Trigger cho Lambda

Cấu hình trigger để mỗi khi có file được upload lên thư mục `products/` của S3 bucket, Lambda sẽ tự động được kích hoạt.

### Các bước thực hiện:
1. Tại giao diện Lambda function `lambda-media-obs` -> Click **Add trigger** ở sơ đồ Function overview.
2. Cấu hình thông số trigger:
   * **Source:** Chọn **S3**.
   * **Bucket:** Chọn bucket `media-s3-minie` (tên bucket lưu media của dự án).
   * **Event types:** Chọn **All object create events** (Kích hoạt khi tạo mới file).
   * **Prefix:** Điền `products/` (chỉ kích hoạt khi upload ảnh vào thư mục sản phẩm sản phẩm).
3. Nhấn **Add** để lưu trigger.

![S3 Trigger on Lambda](./Evidence/s3_trigger_config.png)

*Ý nghĩa hình ảnh:* Sơ đồ trigger cho thấy Lambda `lambda-media-obs` đã được liên kết với sự kiện ObjectCreated từ S3 bucket `media-s3-minie` với bộ lọc folder `products/`.

---

## MH-OBS 5 — Upload file kiểm thử lên S3

Nhóm tiến hành tải một file ảnh lên thư mục `products/` của S3 bucket để kiểm tra xem hệ thống có tự động kích hoạt luồng xử lý không.

### Các bước thực hiện:
1. Truy cập **S3 Console** -> click chọn bucket `media-s3-minie` -> truy cập vào thư mục `products/`.
2. Click **Upload** -> Chọn một file ảnh từ máy tính -> Click **Upload** ở góc dưới.

![Upload File S3](./Evidence/s3_upload_test.png)

*Ý nghĩa hình ảnh:* File ảnh kiểm thử đã được upload thành công vào thư mục `products/` của S3 bucket `media-s3-minie` để bắt đầu trigger luồng xử lý.

---

## MH-OBS 6 — Kiểm tra Logs trong CloudWatch Log Groups

Sau khi file được upload, Lambda được trigger và in ra các log chi tiết về quá trình xử lý.

### Các bước thực hiện:
1. Truy cập **CloudWatch Console** -> Chọn **Log groups** ở menu trái.
2. Tìm và click chọn Log Group `/aws/lambda/lambda-minie-media-obs`.
3. Mở log stream mới nhất để kiểm tra các dòng log in ra từ Lambda.

![CloudWatch Logs Stream](./Evidence/cloudwatch_logs.png)



---

## MH-OBS 7 — Kiểm tra dữ liệu metadata trong DynamoDB

Kiểm tra DynamoDB để xác nhận metadata của file ảnh vừa tải lên đã được Lambda ghi thành công vào bảng.

### Các bước thực hiện:
1. Truy cập **DynamoDB Console** -> Chọn **Tables** -> click vào bảng `minie-metadata`.
2. Click chọn **Explore table items** ở góc trên cùng bên phải để xem dữ liệu.

![DynamoDB Item Metadata](./Evidence/dynamodb_item.png)


---

## MH-OBS 8 — Kiểm tra Custom Metric trong CloudWatch

Lambda đã đẩy thành công hai Custom Metrics lên CloudWatch để theo dõi hiệu suất nghiệp vụ:
*   **Namespace:** `MineE/Media`
*   **Metrics:** `MediaUploadCount` (Số lượng upload) và `MediaUploadBytes` (Dung lượng upload)

### Các bước thực hiện:
1. Truy cập **CloudWatch Console** -> Chọn **Metrics** -> **All metrics**.
2. Tìm và click chọn Namespace **MineE/Media** -> Xem các metric tương ứng để kiểm tra dữ liệu được biểu diễn dưới dạng đồ thị.

![CloudWatch Custom Metric](./Evidence/cloudwatch_custom_metric.png)


---

## MH-OBS 9 — Tạo CloudWatch Dashboard

Nhóm thiết kế một Dashboard tập trung để theo dõi sức khỏe và hiệu suất của luồng xử lý ảnh.

### Các bước thực hiện:
1. Vào **CloudWatch Console** -> Click **Dashboards** ở menu trái -> Click **Create dashboard**.
2. Đặt tên: `dashboard-obs` -> Click **Create**.
3. Thêm các Widget (đồ thị hiển thị) để đáp ứng đúng yêu cầu tối thiểu **3 Widgets (1 Custom + 2 Standard)** của đề bài:
   * **Widget 1 (Custom Metric):** Chọn loại Line -> Chọn metric `MediaUploadCount` từ namespace `MineE/Media` -> Đổi **Statistic** thành `Sum`, **Period** thành `1 minute`.
   * **Widget 2 (Standard Metric 1):** Chọn loại Line -> Chọn metric `Duration` (Thời gian chạy) của Lambda function `lambda-media-obs`.
   * **Widget 3 (Standard Metric 2):** Chọn loại Line -> Chọn metric `Errors` (Lỗi phát sinh) của Lambda function `lambda-media-obs`.
4. Nhấn **Save dashboard** để lưu lại.

![CloudWatch Dashboard](./Evidence/cloudwatch_dashboard.png)


---

## MH-OBS 10 — Cấu hình CloudWatch Alarm và Kiểm thử

### Các bước thực hiện:
1. Vào **CloudWatch Console** -> Click **Alarms** -> **All alarms** -> Click **Create alarm** -> Click **Select metric**.
2. Chọn metric `MediaUploadCount` từ namespace `MineE/Media` -> Click **Select metric**.
3. Cấu hình điều kiện cảnh báo:
   * **Statistic:** `Sum` (Cộng dồn số lượng).
   * **Period:** `5 minutes`.
   * **Condition:** Static -> Chọn `Greater/Equal (>=)` -> Điền giá trị `1` (Báo động ngay khi có ít nhất 1 file upload).
4. Thiết lập Action: Cấu hình gửi thông báo tới SNS Topic `alarm-media-upload` để gửi thông tin về email.
5. Đặt tên Alarm: `alarm-media-upload` -> Click **Create alarm**.
6. **Kiểm thử Alarm:** Tiến hành upload thêm một file ảnh nữa lên S3. Đợi 1-2 phút để CloudWatch cập nhật dữ liệu và kiểm tra xem Alarm có chuyển sang màu đỏ (trạng thái **In alarm**) hay không.

### Trạng thái Alarm chuyển sang "In Alarm" khi có file upload:

![CloudWatch In Alarm](./Evidence/cloudwatch_alarm_state.png)


---

## MH-OBS 11 — Sử dụng Logs Insights Query để truy vết

Sử dụng Logs Insights để thực hiện truy vấn nhanh các log quan trọng của Lambda, giúp kiểm toán và xử lý sự cố nhanh chóng.

### Các bước thực hiện:
1. Vào **CloudWatch Console** -> Click **Logs Insights** ở menu trái.
2. Tại thanh chọn Log Group, chọn `/aws/lambda/lambda-minie-media-obs`.
3. Nhập câu truy vấn nâng cao để lọc các bước xử lý chính:
4. Bấm **Run query** để xem kết quả.

![CloudWatch Logs Insights Query](./Evidence/logs_insights_query.png)


---

# Kết luận MH-OBS

Trong MH-OBS, nhóm đã triển khai observability cho luồng xử lý media của Mine-e bằng các dịch vụ AWS gồm S3, Lambda, DynamoDB và CloudWatch.

Luồng observability được kiểm chứng như sau:

```text
Upload ảnh vào S3 bucket media-s3-minie/products/
        ↓
S3 ObjectCreated trigger Lambda
        ↓
Lambda xử lý metadata của object
        ↓
Lambda ghi metadata vào DynamoDB
        ↓
Lambda publish custom metrics lên CloudWatch
        ↓
CloudWatch Dashboard hiển thị metric
        ↓
CloudWatch Alarm cảnh báo khi có upload
        ↓
Logs Insights query giúp truy vết pipeline
```

**Các evidence đã chứng minh:**
1. S3 upload event đã trigger Lambda thành công.
2. Lambda log ghi nhận object được xử lý.
3. Metadata của file upload được lưu vào DynamoDB table `minie-metadata`.
4. Lambda publish custom metrics `MediaUploadCount` và `MediaUploadBytes` vào namespace `MineE/Media`.
5. CloudWatch Dashboard `dashboard-obs` hiển thị custom metrics.
6. CloudWatch Alarm `alarm-media-upload` chuyển sang trạng thái In alarm khi `MediaUploadCount` >= 1.
7. Logs Insights query truy vấn được các log quan trọng:
   - S3 Event received
   - DynamoDB PutItem completed
   - CloudWatch custom metrics published successfully

**Ý nghĩa của MH-OBS là hệ thống không chỉ chạy được, mà còn có khả năng quan sát được:**
- Biết khi nào có file media được upload.
- Biết Lambda đã xử lý object nào.
- Biết metadata đã được ghi vào DynamoDB hay chưa.
- Có metric để theo dõi số lượng upload và dung lượng upload.
- Có dashboard để quan sát tập trung.
- Có alarm để cảnh báo khi có event upload.
- Có Logs Insights để điều tra khi pipeline lỗi.

**Kết luận: MH-OBS đã hoàn thành, đã chứng minh được một observability pipeline hoàn chỉnh cho media workflow của Mine-e, bao gồm logs, custom metrics, dashboard, alarm và query phục vụ troubleshooting.**
