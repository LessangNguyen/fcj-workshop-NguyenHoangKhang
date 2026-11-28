---
report_key: "workshopChapters"
report_language: "vi"
report_fields: {"shortTitle": "Vận hành, Chi phí & Kiểm thử", "customTitle": "5.5. Vận hành CloudWatch, Kiểm soát Chi phí & Kiểm thử", "summary": "Hoàn thiện trụ cột Operational Excellence, Cost Optimization & End-to-End Validation cho MediMind Clinical AI bằng CloudWatch Log Group (14 ngày), SNS Alert, 3 CloudWatch Metric Alarms (EC2/ALB/RDS), AWS Budgets ngưỡng $20/tháng cùng kiểm thử thực tế live (/health, /ready, CloudFront CSP, AWS WAF chặn SQLi 403 và sàng lọc cờ đỏ lâm sàng).", "content": "Đã hoàn tất cấu hình CloudWatch Log Group /medimind/clinical-api (14 ngày), SNS medimind-ops-alerts, 3 CloudWatch Metric Alarms (EC2/ALB/RDS), AWS Budgets $20/tháng và kiểm chứng thực tế live (/health, /ready, CloudFront SGN50-P1, AWS WAF SQLi 403 BLOCK và sàng lọc cờ đỏ lâm sàng Double Guardrails).", "notes": "", "steps": [{"title": "Khởi tạo CloudWatch Log Group (14 ngày) & SNS Email Alert (medimind-ops-alerts)", "summary": "Thiết lập kho lưu trữ nhật ký ứng dụng `/medimind/clinical-api` với chính sách tự xoay vòng `14 ngày` (tránh phình dung lượng log) và tạo SNS Topic `medimind-ops-alerts` đăng ký email quản trị để nhận cảnh báo sự cố theo thời gian thực (miễn phí trong AWS Free Tier).", "code": "export AWS_REGION=ap-southeast-1\naws logs create-log-group --log-group-name /medimind/clinical-api --region $AWS_REGION 2>/dev/null || true\naws logs put-retention-policy --log-group-name /medimind/clinical-api --retention-in-days 14 --region $AWS_REGION\nSNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)\naws sns subscribe --topic-arn \"$SNS_ARN\" --protocol email --notification-endpoint \"<YOUR_ALERT_EMAIL>\" --region $AWS_REGION\naws logs describe-log-groups --log-group-name-prefix /medimind --region $AWS_REGION --query 'logGroups[*].{LogGroup:logGroupName,RetentionDays:retentionInDays}' --output table", "expected": "LogGroup: /medimind/clinical-api | RetentionDays: 14\nSubscriptionArn: pending confirmation (xác nhận qua liên kết Confirm subscription trong email)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/16a-cli-cw-logs-sns.png", "caption": "Ảnh 16a (CLI) — Khởi tạo CloudWatch Log Group /medimind/clinical-api (14 ngày) & SNS Topic medimind-ops-alerts"}, {"url": "static/images/5-Workshop/5.5-Operations/16a-sns-email.png", "caption": "Ảnh 16a (Email) — Thư xác nhận AWS Notification - Subscription Confirmation gửi về hộp thư quản trị"}], "num": "Bước 5.5.1"}, {"title": "Thiết lập 3 CloudWatch Metric Alarms trọng yếu (EC2, ALB, RDS)", "summary": "Khởi tạo bộ 3 cảnh báo tự động theo dõi sức khỏe toàn chuỗi hạ tầng 3 tầng (`medimind-ec2-high-cpu > 80%`, `medimind-alb-unhealthy-hosts >= 1`, `medimind-rds-high-cpu > 80%`), nối trực tiếp vào SNS Topic `medimind-ops-alerts` (nằm trọn trong hạn mức `10` Alarms miễn phí của AWS Free Tier).", "code": "export AWS_REGION=ap-southeast-1\nSNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)\nEC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters \"Name=tag:Name,Values=medimind-app-server\" \"Name=instance-state-name,Values=running\" --query 'Reservations[0].Instances[0].InstanceId' --output text)\nALB_DIM=$(aws elbv2 describe-load-balancers --names medimind-alb --region $AWS_REGION --query 'LoadBalancers[0].LoadBalancerArn' --output text | sed 's|.*loadbalancer/||')\nTG_DIM=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text | sed 's|.*:targetgroup/|targetgroup/|')\n\naws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-ec2-high-cpu --namespace AWS/EC2 --metric-name CPUUtilization --statistic Average --period 300 --evaluation-periods 2 --threshold 80 --comparison-operator GreaterThanThreshold --dimensions Name=InstanceId,Value=\"$EC2_ID\" --alarm-actions \"$SNS_ARN\"\naws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-alb-unhealthy-hosts --namespace AWS/ApplicationELB --metric-name UnHealthyHostCount --statistic Maximum --period 60 --evaluation-periods 1 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --dimensions Name=LoadBalancer,Value=\"$ALB_DIM\" Name=TargetGroup,Value=\"$TG_DIM\" --alarm-actions \"$SNS_ARN\"\naws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-rds-high-cpu --namespace AWS/RDS --metric-name CPUUtilization --statistic Average --period 300 --evaluation-periods 2 --threshold 80 --comparison-operator GreaterThanThreshold --dimensions Name=DBInstanceIdentifier,Value=medimind-pg-db --alarm-actions \"$SNS_ARN\"\n\naws cloudwatch describe-alarms --alarm-name-prefix medimind- --region $AWS_REGION --query 'MetricAlarms[*].{AlarmName:AlarmName,Metric:MetricName,Threshold:Threshold,State:StateValue}' --output table", "expected": "medimind-alb-unhealthy-hosts (UnHealthyHostCount >= 1.0)\nmedimind-ec2-high-cpu (CPUUtilization > 80.0)\nmedimind-rds-high-cpu (CPUUtilization > 80.0)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/16b-cli-cw-alarms.png", "caption": "Ảnh 16b (CLI) — Khởi tạo 3 CloudWatch Alarms cho EC2, ALB và RDS trên CloudShell"}, {"url": "static/images/5-Workshop/5.5-Operations/16-cw-alarms.png", "caption": "Ảnh 16 (Console) — Danh sách 3 CloudWatch Alarms (Actions enabled) trên AWS Management Console"}], "num": "Bước 5.5.2"}, {"title": "Thiết lập AWS Budgets ($20/tháng) & Kiểm soát Chi phí Vận hành", "summary": "Khởi tạo ngân sách cảnh báo tự động `medimind-monthly-budget-20usd` (miễn phí `$0`) gửi email ngay khi chi phí thực tế vượt **80% (`$16`)** hoặc dự báo cuối tháng vượt **100% (`$20`)**. Toàn bộ bảng tính chi phí đối chiếu 100% (`~$16.86` demo ~40h · `$207.28` live 24/7 · `$228.76` Full Multi-AZ) được công khai tại [AWS Pricing Calculator](https://calculator.aws/#/estimate?id=ec26a26022ee93e0eb9785a0048f4dca71eb6fb3).", "code": "ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)\n\ncat > /tmp/budget.json << 'EOF'\n{\n  \"BudgetName\": \"medimind-monthly-budget-20usd\",\n  \"BudgetLimit\": { \"Amount\": \"20\", \"Unit\": \"USD\" },\n  \"TimeUnit\": \"MONTHLY\",\n  \"BudgetType\": \"COST\"\n}\nEOF\n\ncat > /tmp/budget-notif.json << 'EOF'\n[\n  {\n    \"Notification\": { \"NotificationType\": \"ACTUAL\", \"ComparisonOperator\": \"GREATER_THAN\", \"Threshold\": 80, \"ThresholdType\": \"PERCENTAGE\" },\n    \"Subscribers\": [{ \"SubscriptionType\": \"EMAIL\", \"Address\": \"<YOUR_ALERT_EMAIL>\" }]\n  },\n  {\n    \"Notification\": { \"NotificationType\": \"FORECASTED\", \"ComparisonOperator\": \"GREATER_THAN\", \"Threshold\": 100, \"ThresholdType\": \"PERCENTAGE\" },\n    \"Subscribers\": [{ \"SubscriptionType\": \"EMAIL\", \"Address\": \"<YOUR_ALERT_EMAIL>\" }]\n  }\n]\nEOF\n\naws budgets create-budget --account-id \"$ACCOUNT_ID\" --budget file:///tmp/budget.json --notifications-with-subscribers file:///tmp/budget-notif.json\naws budgets describe-budgets --account-id \"$ACCOUNT_ID\" --query 'Budgets[*].{Name:BudgetName,Limit:BudgetLimit.Amount,Unit:BudgetLimit.Unit,Type:BudgetType}' --output table", "expected": "Limit: 20.0 USD | Name: medimind-monthly-budget-20usd | Type: COST | Status: OK / Healthy", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/17a-cli-budgets.png", "caption": "Ảnh 17a (CLI) — Khởi tạo AWS Budget medimind-monthly-budget-20usd (ngưỡng $20/tháng)"}, {"url": "static/images/5-Workshop/5.5-Operations/17-budgets.png", "caption": "Ảnh 17 (Console) — AWS Billing & Cost Management Budgets trạng thái OK / Healthy ($20.00)"}], "num": "Bước 5.5.3"}, {"num": "Bước 5.5.4", "title": "Kiểm thử Hạ tầng Live (/health, /ready, CloudFront) & AWS WAF Chặn SQL Injection (403)", "summary": "Kiểm chứng trực tiếp hệ thống đang chạy live tại `https://d2typ9pphzk4as.cloudfront.net` theo hai kịch bản: **(1)** xác nhận sức khỏe toàn chuỗi CloudFront (`SGN50-P1`) → ALB → 2 EC2 → ChromaDB RAG (`circuit_breaker: CLOSED`) kèm HTTP Security Headers (`Content-Security-Policy`, `X-Frame-Options: SAMEORIGIN`); và **(2)** giả lập tấn công SQL Injection (`/?id=1' or '1'=1`) để xác minh khiên **AWS WAF (`medimind-edge-waf`)** chặn đứng bằng mã **`403 Forbidden`** ngay tại biên và ghi vết `BLOCK` dưới luật **`AWSManagedRulesSQLiRuleSet`**.", "code": "# 1. Kiểm thử /health, /ready & HTTP Security Headers + CloudFront Edge POP (PowerShell)\ncurl.exe -s https://d2typ9pphzk4as.cloudfront.net/health; echo \"\"\ncurl.exe -s https://d2typ9pphzk4as.cloudfront.net/ready; echo \"\"\ncurl.exe -sI https://d2typ9pphzk4as.cloudfront.net/static/css/style.css | Select-String -Pattern \"HTTP/|x-cache|x-amz-cf-pop|x-frame-options|content-security-policy\"\n\n# 2. Giả lập tấn công SQL Injection qua trình duyệt hoặc curl (kỳ vọng 403 Blocked tại CloudFront/WAF)\ncurl.exe -i \"https://d2typ9pphzk4as.cloudfront.net/?id=1%27%20OR%20%271%27=%271\"", "expected": "{\"status\":\"ok\"}\n{\"status\":\"ready\",\"rag\":\"ready\",\"gemini\":\"ready\",\"circuit_breaker\":\"CLOSED\"}\nHTTP/1.1 200 OK | x-frame-options: SAMEORIGIN | X-Amz-Cf-Pop: SGN50-P1\nSQLi Test -> HTTP/1.1 403 Forbidden (Request blocked) | AWS WAF Sampled requests: SQLiRuleSet -> BLOCK", "images": [{"url": "static/images/5-Workshop/5.5-Operations/22-live-health-cloudfront.jpg", "caption": "Ảnh 22 (Live CLI) — Xác nhận /health ok, /ready ready (RAG & Circuit Breaker CLOSED), CSP/X-Frame-Options & nút biên CloudFront SGN50-P1"}, {"url": "static/images/5-Workshop/5.5-Operations/23a-waf-sqli-403.png", "caption": "Ảnh 23a (Browser) — Kiểm thử tấn công SQL Injection (/?id=1' or '1'=1) bị CloudFront & AWS WAF chặn 403 ERROR ngay tại biên"}, {"url": "static/images/5-Workshop/5.5-Operations/23b-waf-sampled-block.jpg", "caption": "Ảnh 23b (Console) — AWS WAF medimind-edge-waf (Sampled requests) ghi nhận loạt sự kiện BLOCK bởi AWSManagedRulesSQLiRuleSet"}]}, {"num": "Bước 5.5.5", "title": "Kiểm thử Nghiệp vụ Lâm sàng Live: Sàng lọc Triệu chứng Cờ Đỏ (Double Guardrails)", "summary": "Kiểm chứng tầng an toàn lâm sàng trực tiếp trên giao diện live `https://d2typ9pphzk4as.cloudfront.net/#chat` với ca triệu chứng nguy kịch (*\"Tôi đau thắt ngực dữ dội, vã mồ hôi lạnh và khó thở lan ra cánh tay trái\"*). Hệ thống lập tức kích hoạt tầng **Double Guardrails**, hiển thị thẻ **`CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP`**, nhận diện đúng dấu hiệu **Đau thắt ngực** và khuyến cáo người bệnh đến cơ sở y tế thăm khám khẩn thay vì tự chẩn đoán qua tin nhắn.", "expected": "Thẻ cảnh báo: CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP\nTrạng thái phiên tư vấn: Nguy cơ cao (Cần thăm khám khẩn) — Không trấn an sai lệch", "images": [{"url": "static/images/5-Workshop/5.5-Operations/24-live-clinical-triage-redflag.jpg", "caption": "Ảnh 24 (Live App) — Trợ lý MediMind kích hoạt Cảnh báo Nguy cơ cao tức thì trước triệu chứng đau thắt ngực cấp cứu trên HTTPS CloudFront"}]}]}
report_index: 4
report_shared: {"code": "5.5"}
title: "Kiểm thử & Đánh giá"
date: "2026-11-28"
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

# 5.5. Vận hành CloudWatch, Kiểm soát Chi phí & Kiểm thử

Hoàn thiện trụ cột Operational Excellence & Cost Optimization cho MediMind Clinical AI bằng CloudWatch Log Group (14 ngày), SNS Alert, 3 CloudWatch Metric Alarms (EC2/ALB/RDS) và AWS Budgets ngưỡng $20/tháng — toàn bộ nằm trong AWS Free Tier ($0).

## Bước 5.5.1 — Khởi tạo CloudWatch Log Group (14 ngày) & SNS Email Alert (medimind-ops-alerts)

Thiết lập kho lưu trữ nhật ký ứng dụng `/medimind/clinical-api` với chính sách tự xoay vòng `14 ngày` (tránh phình dung lượng log) và tạo SNS Topic `medimind-ops-alerts` đăng ký email quản trị để nhận cảnh báo sự cố theo thời gian thực (miễn phí trong AWS Free Tier).

```bash
export AWS_REGION=ap-southeast-1
aws logs create-log-group --log-group-name /medimind/clinical-api --region $AWS_REGION 2>/dev/null || true
aws logs put-retention-policy --log-group-name /medimind/clinical-api --retention-in-days 14 --region $AWS_REGION
SNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)
aws sns subscribe --topic-arn "$SNS_ARN" --protocol email --notification-endpoint "<YOUR_ALERT_EMAIL>" --region $AWS_REGION
aws logs describe-log-groups --log-group-name-prefix /medimind --region $AWS_REGION --query 'logGroups[*].{LogGroup:logGroupName,RetentionDays:retentionInDays}' --output table
```

**Kết quả kỳ vọng:**
```text
LogGroup: /medimind/clinical-api | RetentionDays: 14
SubscriptionArn: pending confirmation (xác nhận qua liên kết Confirm subscription trong email)
```

![Ảnh 16a (CLI) — Khởi tạo CloudWatch Log Group /medimind/clinical-api (14 ngày) & SNS Topic medimind-ops-alerts](../../../static/images/5-Workshop/5.5-Operations/cli-steps/16a-cli-cw-logs-sns.png)
*Ảnh 16a (CLI) — Khởi tạo CloudWatch Log Group /medimind/clinical-api (14 ngày) & SNS Topic medimind-ops-alerts*

![Ảnh 16a (Email) — Thư xác nhận AWS Notification - Subscription Confirmation gửi về hộp thư quản trị](../../../static/images/5-Workshop/5.5-Operations/16a-sns-email.png)
*Ảnh 16a (Email) — Thư xác nhận AWS Notification - Subscription Confirmation gửi về hộp thư quản trị*

## Bước 5.5.2 — Thiết lập 3 CloudWatch Metric Alarms trọng yếu (EC2, ALB, RDS)

Khởi tạo bộ 3 cảnh báo tự động theo dõi sức khỏe toàn chuỗi hạ tầng 3 tầng (`medimind-ec2-high-cpu > 80%`, `medimind-alb-unhealthy-hosts >= 1`, `medimind-rds-high-cpu > 80%`), nối trực tiếp vào SNS Topic `medimind-ops-alerts` (nằm trọn trong hạn mức `10` Alarms miễn phí của AWS Free Tier).

```bash
export AWS_REGION=ap-southeast-1
SNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)
EC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters "Name=tag:Name,Values=medimind-app-server" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].InstanceId' --output text)
ALB_DIM=$(aws elbv2 describe-load-balancers --names medimind-alb --region $AWS_REGION --query 'LoadBalancers[0].LoadBalancerArn' --output text | sed 's|.*loadbalancer/||')
TG_DIM=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text | sed 's|.*:targetgroup/|targetgroup/|')

aws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-ec2-high-cpu --namespace AWS/EC2 --metric-name CPUUtilization --statistic Average --period 300 --evaluation-periods 2 --threshold 80 --comparison-operator GreaterThanThreshold --dimensions Name=InstanceId,Value="$EC2_ID" --alarm-actions "$SNS_ARN"
aws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-alb-unhealthy-hosts --namespace AWS/ApplicationELB --metric-name UnHealthyHostCount --statistic Maximum --period 60 --evaluation-periods 1 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --dimensions Name=LoadBalancer,Value="$ALB_DIM" Name=TargetGroup,Value="$TG_DIM" --alarm-actions "$SNS_ARN"
aws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-rds-high-cpu --namespace AWS/RDS --metric-name CPUUtilization --statistic Average --period 300 --evaluation-periods 2 --threshold 80 --comparison-operator GreaterThanThreshold --dimensions Name=DBInstanceIdentifier,Value=medimind-pg-db --alarm-actions "$SNS_ARN"

aws cloudwatch describe-alarms --alarm-name-prefix medimind- --region $AWS_REGION --query 'MetricAlarms[*].{AlarmName:AlarmName,Metric:MetricName,Threshold:Threshold,State:StateValue}' --output table
```

**Kết quả kỳ vọng:**
```text
medimind-alb-unhealthy-hosts (UnHealthyHostCount >= 1.0)
medimind-ec2-high-cpu (CPUUtilization > 80.0)
medimind-rds-high-cpu (CPUUtilization > 80.0)
```

![Ảnh 16b (CLI) — Khởi tạo 3 CloudWatch Alarms cho EC2, ALB và RDS trên CloudShell](../../../static/images/5-Workshop/5.5-Operations/cli-steps/16b-cli-cw-alarms.png)
*Ảnh 16b (CLI) — Khởi tạo 3 CloudWatch Alarms cho EC2, ALB và RDS trên CloudShell*

![Ảnh 16 (Console) — Danh sách 3 CloudWatch Alarms (Actions enabled) trên AWS Management Console](../../../static/images/5-Workshop/5.5-Operations/16-cw-alarms.png)
*Ảnh 16 (Console) — Danh sách 3 CloudWatch Alarms (Actions enabled) trên AWS Management Console*

## Bước 5.5.3 — Thiết lập AWS Budgets ($20/tháng) & Kiểm soát Chi phí Vận hành

Khởi tạo ngân sách cảnh báo tự động `medimind-monthly-budget-20usd` (miễn phí `$0`) gửi email ngay khi chi phí thực tế vượt **80% (`$16`)** hoặc dự báo cuối tháng vượt **100% (`$20`)**. Toàn bộ bảng tính chi phí đối chiếu 100% (`~$16.86` demo ~40h · `$207.28` live 24/7 · `$228.76` Full Multi-AZ) được công khai tại [AWS Pricing Calculator](https://calculator.aws/#/estimate?id=ec26a26022ee93e0eb9785a0048f4dca71eb6fb3).

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/budget.json << 'EOF'
{
  "BudgetName": "medimind-monthly-budget-20usd",
  "BudgetLimit": { "Amount": "20", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
EOF

cat > /tmp/budget-notif.json << 'EOF'
[
  {
    "Notification": { "NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80, "ThresholdType": "PERCENTAGE" },
    "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "<YOUR_ALERT_EMAIL>" }]
  },
  {
    "Notification": { "NotificationType": "FORECASTED", "ComparisonOperator": "GREATER_THAN", "Threshold": 100, "ThresholdType": "PERCENTAGE" },
    "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "<YOUR_ALERT_EMAIL>" }]
  }
]
EOF

aws budgets create-budget --account-id "$ACCOUNT_ID" --budget file:///tmp/budget.json --notifications-with-subscribers file:///tmp/budget-notif.json
aws budgets describe-budgets --account-id "$ACCOUNT_ID" --query 'Budgets[*].{Name:BudgetName,Limit:BudgetLimit.Amount,Unit:BudgetLimit.Unit,Type:BudgetType}' --output table
```

**Kết quả kỳ vọng:**
```text
Limit: 20.0 USD | Name: medimind-monthly-budget-20usd | Type: COST | Status: OK / Healthy
```

![Ảnh 17a (CLI) — Khởi tạo AWS Budget medimind-monthly-budget-20usd (ngưỡng $20/tháng)](../../../static/images/5-Workshop/5.5-Operations/cli-steps/17a-cli-budgets.png)
*Ảnh 17a (CLI) — Khởi tạo AWS Budget medimind-monthly-budget-20usd (ngưỡng $20/tháng)*

![Ảnh 17 (Console) — AWS Billing & Cost Management Budgets trạng thái OK / Healthy ($20.00)](../../../static/images/5-Workshop/5.5-Operations/17-budgets.png)
*Ảnh 17 (Console) — AWS Billing & Cost Management Budgets trạng thái OK / Healthy ($20.00)*

## Bước 5.5.4 — Kiểm thử Hạ tầng Live (/health, /ready, CloudFront) & AWS WAF Chặn SQL Injection (403)

Kiểm chứng trực tiếp hệ thống đang chạy live tại `https://d2typ9pphzk4as.cloudfront.net` theo hai kịch bản: **(1)** xác nhận sức khỏe toàn chuỗi CloudFront (`SGN50-P1`) → ALB → 2 EC2 → ChromaDB RAG (`circuit_breaker: CLOSED`) kèm HTTP Security Headers (`Content-Security-Policy`, `X-Frame-Options: SAMEORIGIN`); và **(2)** giả lập tấn công SQL Injection (`/?id=1' or '1'=1`) để xác minh khiên **AWS WAF (`medimind-edge-waf`)** chặn đứng bằng mã **`403 Forbidden`** ngay tại biên và ghi vết `BLOCK` dưới luật **`AWSManagedRulesSQLiRuleSet`**.

```powershell
# 1. Kiểm thử /health, /ready & HTTP Security Headers + CloudFront Edge POP (PowerShell)
curl.exe -s https://d2typ9pphzk4as.cloudfront.net/health; echo ""
curl.exe -s https://d2typ9pphzk4as.cloudfront.net/ready; echo ""
curl.exe -sI https://d2typ9pphzk4as.cloudfront.net/static/css/style.css | Select-String -Pattern "HTTP/|x-cache|x-amz-cf-pop|x-frame-options|content-security-policy"

# 2. Giả lập tấn công SQL Injection qua trình duyệt hoặc curl (kỳ vọng 403 Blocked tại CloudFront/WAF)
curl.exe -i "https://d2typ9pphzk4as.cloudfront.net/?id=1%27%20OR%20%271%27=%271"
```

**Kết quả kỳ vọng:**
```text
{"status":"ok"}
{"status":"ready","rag":"ready","gemini":"ready","circuit_breaker":"CLOSED"}
HTTP/1.1 200 OK | x-frame-options: SAMEORIGIN | X-Amz-Cf-Pop: SGN50-P1
SQLi Test -> HTTP/1.1 403 Forbidden (Request blocked) | AWS WAF Sampled requests: SQLiRuleSet -> BLOCK
```

![Ảnh 22 (Live CLI) — Xác nhận /health ok, /ready ready (RAG & Circuit Breaker CLOSED), CSP/X-Frame-Options & nút biên CloudFront SGN50-P1](../../../static/images/5-Workshop/5.5-Operations/22-live-health-cloudfront.jpg)
*Ảnh 22 (Live CLI) — Xác nhận /health ok, /ready ready (RAG & Circuit Breaker CLOSED), CSP/X-Frame-Options & nút biên CloudFront SGN50-P1*

![Ảnh 23a (Browser) — Kiểm thử tấn công SQL Injection (/?id=1' or '1'=1) bị CloudFront & AWS WAF chặn 403 ERROR ngay tại biên](../../../static/images/5-Workshop/5.5-Operations/23a-waf-sqli-403.png)
*Ảnh 23a (Browser) — Kiểm thử tấn công SQL Injection (/?id=1' or '1'=1) bị CloudFront & AWS WAF chặn 403 ERROR ngay tại biên*

![Ảnh 23b (Console) — AWS WAF medimind-edge-waf (Sampled requests) ghi nhận loạt sự kiện BLOCK bởi AWSManagedRulesSQLiRuleSet](../../../static/images/5-Workshop/5.5-Operations/23b-waf-sampled-block.jpg)
*Ảnh 23b (Console) — AWS WAF medimind-edge-waf (Sampled requests) ghi nhận loạt sự kiện BLOCK bởi AWSManagedRulesSQLiRuleSet*

## Bước 5.5.5 — Kiểm thử Nghiệp vụ Lâm sàng Live: Sàng lọc Triệu chứng Cờ Đỏ (Double Guardrails)

Kiểm chứng tầng an toàn lâm sàng trực tiếp trên giao diện live `https://d2typ9pphzk4as.cloudfront.net/#chat` với ca triệu chứng nguy kịch (*"Tôi đau thắt ngực dữ dội, vã mồ hôi lạnh và khó thở lan ra cánh tay trái"*). Hệ thống lập tức kích hoạt tầng **Double Guardrails**, hiển thị thẻ **`CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP`**, nhận diện đúng dấu hiệu **Đau thắt ngực** và khuyến cáo người bệnh đến cơ sở y tế thăm khám khẩn thay vì tự chẩn đoán qua tin nhắn.

**Kết quả kỳ vọng:**
```text
Thẻ cảnh báo: CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP
Trạng thái phiên tư vấn: Nguy cơ cao (Cần thăm khám khẩn) — Không trấn an sai lệch
```

![Ảnh 24 (Live App) — Trợ lý MediMind kích hoạt Cảnh báo Nguy cơ cao tức thì trước triệu chứng đau thắt ngực cấp cứu trên HTTPS CloudFront](../../../static/images/5-Workshop/5.5-Operations/24-live-clinical-triage-redflag.jpg)
*Ảnh 24 (Live App) — Trợ lý MediMind kích hoạt Cảnh báo Nguy cơ cao tức thì trước triệu chứng đau thắt ngực cấp cứu trên HTTPS CloudFront*
