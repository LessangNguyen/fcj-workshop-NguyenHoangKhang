---
report_key: "workshopChapters"
report_language: "en"
report_fields: {"shortTitle": "Operations, Cost & Validation", "customTitle": "5.5. CloudWatch Operations, Cost Governance & Validation", "summary": "Complete the Operational Excellence, Cost Optimization & End-to-End Validation pillars for MediMind Clinical AI with CloudWatch Log Group (14-day retention), SNS Alerting, 3 CloudWatch Metric Alarms (EC2/ALB/RDS), a $20/month AWS Budget, and live HTTPS verification (/health, /ready, CloudFront CSP, AWS WAF SQLi 403 blocking, and clinical Red-Flag triage).", "content": "Completed CloudWatch Log Group /medimind/clinical-api (14-day retention), SNS medimind-ops-alerts, 3 CloudWatch Metric Alarms (EC2/ALB/RDS), $20/month AWS Budget, and live HTTPS end-to-end validation (/health, /ready, CloudFront SGN50-P1, AWS WAF SQLi 403 BLOCK, and clinical Red-Flag triage).", "notes": "", "steps": [{"title": "Create CloudWatch Log Group (14-Day Retention) & SNS Alert Topic", "summary": "Create `/medimind/clinical-api` with a `14-day` retention policy to cap log storage costs and provision SNS Topic `medimind-ops-alerts` subscribed to an operations email for real-time incident notifications (AWS Free Tier).", "code": "export AWS_REGION=ap-southeast-1\naws logs create-log-group --log-group-name /medimind/clinical-api --region $AWS_REGION 2>/dev/null || true\naws logs put-retention-policy --log-group-name /medimind/clinical-api --retention-in-days 14 --region $AWS_REGION\nSNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)\naws sns subscribe --topic-arn \"$SNS_ARN\" --protocol email --notification-endpoint \"<YOUR_ALERT_EMAIL>\" --region $AWS_REGION\naws logs describe-log-groups --log-group-name-prefix /medimind --region $AWS_REGION --query 'logGroups[*].{LogGroup:logGroupName,RetentionDays:retentionInDays}' --output table", "expected": "LogGroup: /medimind/clinical-api | RetentionDays: 14\nSubscriptionArn: pending confirmation (confirm via email link)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/16a-cli-cw-logs-sns.png", "caption": "Figure 16a (CLI) — Creating CloudWatch Log Group /medimind/clinical-api (14 days) & SNS Topic"}, {"url": "static/images/5-Workshop/5.5-Operations/16a-sns-email.png", "caption": "Figure 16a (Email) — AWS Notification Subscription Confirmation email received"}], "num": "Step 5.5.1"}, {"title": "Configure 3 Core CloudWatch Metric Alarms (EC2, ALB, RDS)", "summary": "Deploy 3 automated alarms covering all 3 tiers (`medimind-ec2-high-cpu > 80%`, `medimind-alb-unhealthy-hosts >= 1`, `medimind-rds-high-cpu > 80%`) routed to `medimind-ops-alerts` (within the 10-alarm AWS Free Tier).", "code": "export AWS_REGION=ap-southeast-1\nSNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)\nEC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters \"Name=tag:Name,Values=medimind-app-server\" \"Name=instance-state-name,Values=running\" --query 'Reservations[0].Instances[0].InstanceId' --output text)\nALB_DIM=$(aws elbv2 describe-load-balancers --names medimind-alb --region $AWS_REGION --query 'LoadBalancers[0].LoadBalancerArn' --output text | sed 's|.*loadbalancer/||')\nTG_DIM=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text | sed 's|.*:targetgroup/|targetgroup/|')\n\naws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-ec2-high-cpu --namespace AWS/EC2 --metric-name CPUUtilization --statistic Average --period 300 --evaluation-periods 2 --threshold 80 --comparison-operator GreaterThanThreshold --dimensions Name=InstanceId,Value=\"$EC2_ID\" --alarm-actions \"$SNS_ARN\"\naws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-alb-unhealthy-hosts --namespace AWS/ApplicationELB --metric-name UnHealthyHostCount --statistic Maximum --period 60 --evaluation-periods 1 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --dimensions Name=LoadBalancer,Value=\"$ALB_DIM\" Name=TargetGroup,Value=\"$TG_DIM\" --alarm-actions \"$SNS_ARN\"\naws cloudwatch put-metric-alarm --region $AWS_REGION --alarm-name medimind-rds-high-cpu --namespace AWS/RDS --metric-name CPUUtilization --statistic Average --period 300 --evaluation-periods 2 --threshold 80 --comparison-operator GreaterThanThreshold --dimensions Name=DBInstanceIdentifier,Value=medimind-pg-db --alarm-actions \"$SNS_ARN\"\n\naws cloudwatch describe-alarms --alarm-name-prefix medimind- --region $AWS_REGION --query 'MetricAlarms[*].{AlarmName:AlarmName,Metric:MetricName,Threshold:Threshold,State:StateValue}' --output table", "expected": "medimind-alb-unhealthy-hosts (UnHealthyHostCount >= 1.0)\nmedimind-ec2-high-cpu (CPUUtilization > 80.0)\nmedimind-rds-high-cpu (CPUUtilization > 80.0)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/16b-cli-cw-alarms.png", "caption": "Figure 16b (CLI) — Creating 3 CloudWatch Metric Alarms for EC2, ALB, and RDS"}, {"url": "static/images/5-Workshop/5.5-Operations/16-cw-alarms.png", "caption": "Figure 16 (Console) — All 3 CloudWatch Alarms with Actions enabled"}], "num": "Step 5.5.2"}, {"title": "Configure AWS Budgets ($20/month) & Cost Governance", "summary": "Create `medimind-monthly-budget-20usd` ($0 free) alerting via email whenever actual spend crosses **80% ($16)** or forecasted spend crosses **100% ($20)**. Full cost breakdown (`~$16.86` ~40h demo · `$207.28` live 24/7 · `$228.76` Full Multi-AZ) is published on [AWS Pricing Calculator](https://calculator.aws/#/estimate?id=ec26a26022ee93e0eb9785a0048f4dca71eb6fb3).", "code": "ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)\n\ncat > /tmp/budget.json << 'EOF'\n{\n  \"BudgetName\": \"medimind-monthly-budget-20usd\",\n  \"BudgetLimit\": { \"Amount\": \"20\", \"Unit\": \"USD\" },\n  \"TimeUnit\": \"MONTHLY\",\n  \"BudgetType\": \"COST\"\n}\nEOF\n\ncat > /tmp/budget-notif.json << 'EOF'\n[\n  {\n    \"Notification\": { \"NotificationType\": \"ACTUAL\", \"ComparisonOperator\": \"GREATER_THAN\", \"Threshold\": 80, \"ThresholdType\": \"PERCENTAGE\" },\n    \"Subscribers\": [{ \"SubscriptionType\": \"EMAIL\", \"Address\": \"<YOUR_ALERT_EMAIL>\" }]\n  },\n  {\n    \"Notification\": { \"NotificationType\": \"FORECASTED\", \"ComparisonOperator\": \"GREATER_THAN\", \"Threshold\": 100, \"ThresholdType\": \"PERCENTAGE\" },\n    \"Subscribers\": [{ \"SubscriptionType\": \"EMAIL\", \"Address\": \"<YOUR_ALERT_EMAIL>\" }]\n  }\n]\nEOF\n\naws budgets create-budget --account-id \"$ACCOUNT_ID\" --budget file:///tmp/budget.json --notifications-with-subscribers file:///tmp/budget-notif.json\naws budgets describe-budgets --account-id \"$ACCOUNT_ID\" --query 'Budgets[*].{Name:BudgetName,Limit:BudgetLimit.Amount,Unit:BudgetLimit.Unit,Type:BudgetType}' --output table", "expected": "Limit: 20.0 USD | Name: medimind-monthly-budget-20usd | Type: COST | Status: OK / Healthy", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/17a-cli-budgets.png", "caption": "Figure 17a (CLI) — Creating AWS Budget medimind-monthly-budget-20usd ($20/month)"}, {"url": "static/images/5-Workshop/5.5-Operations/17-budgets.png", "caption": "Figure 17 (Console) — AWS Billing Budgets overview showing OK / Healthy ($20.00)"}], "num": "Step 5.5.3"}, {"num": "Step 5.5.4", "title": "Live Infrastructure Validation (/health, /ready, CloudFront) & AWS WAF SQLi Blocking (403)", "summary": "Verify the live deployment at `https://d2typ9pphzk4as.cloudfront.net` across two live tests: **(1)** confirm full-stack readiness from CloudFront Edge (`SGN50-P1`) → ALB → 2 EC2 nodes → ChromaDB RAG (`circuit_breaker: CLOSED`) with strict HTTP Security Headers (`Content-Security-Policy`, `X-Frame-Options: SAMEORIGIN`); and **(2)** trigger a live SQL Injection probe (`/?id=1' or '1'=1`) to prove **AWS WAF (`medimind-edge-waf`)** blocks it at the edge with **`403 Forbidden`** and logs `BLOCK` events under **`AWSManagedRulesSQLiRuleSet`**.", "code": "# 1. Verify /health, /ready, HTTP Security Headers & CloudFront Edge POP (PowerShell)\ncurl.exe -s https://d2typ9pphzk4as.cloudfront.net/health; echo \"\"\ncurl.exe -s https://d2typ9pphzk4as.cloudfront.net/ready; echo \"\"\ncurl.exe -sI https://d2typ9pphzk4as.cloudfront.net/static/css/style.css | Select-String -Pattern \"HTTP/|x-cache|x-amz-cf-pop|x-frame-options|content-security-policy\"\n\n# 2. Simulate SQL Injection via browser or curl (expected 403 Blocked at CloudFront/WAF)\ncurl.exe -i \"https://d2typ9pphzk4as.cloudfront.net/?id=1%27%20OR%20%271%27=%271\"", "expected": "{\"status\":\"ok\"}\n{\"status\":\"ready\",\"rag\":\"ready\",\"gemini\":\"ready\",\"circuit_breaker\":\"CLOSED\"}\nHTTP/1.1 200 OK | x-frame-options: SAMEORIGIN | X-Amz-Cf-Pop: SGN50-P1\nSQLi Test -> HTTP/1.1 403 Forbidden (Request blocked) | AWS WAF Sampled requests: SQLiRuleSet -> BLOCK", "images": [{"url": "static/images/5-Workshop/5.5-Operations/22-live-health-cloudfront.jpg", "caption": "Figure 22 (Live CLI) — Verified /health ok, /ready ready (RAG & Circuit Breaker CLOSED), CSP/X-Frame-Options & CloudFront POP SGN50-P1"}, {"url": "static/images/5-Workshop/5.5-Operations/23a-waf-sqli-403.png", "caption": "Figure 23a (Browser) — Live SQL Injection test (/?id=1' or '1'=1) blocked at CloudFront edge with 403 ERROR"}, {"url": "static/images/5-Workshop/5.5-Operations/23b-waf-sampled-block.jpg", "caption": "Figure 23b (Console) — AWS WAF medimind-edge-waf Sampled requests logging BLOCK actions under AWSManagedRulesSQLiRuleSet"}]}, {"num": "Step 5.5.5", "title": "Live Clinical Safety Validation: Emergency Red-Flag Triage (Double Guardrails)", "summary": "Validate clinical safety guardrails live at `https://d2typ9pphzk4as.cloudfront.net/#chat` using a high-risk chest pain scenario (*\"Tôi đau thắt ngực dữ dội, vã mồ hôi lạnh và khó thở lan ra cánh tay trái\"*). MediMind immediately triggers **Double Guardrails**, raises the **`CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP`** urgent alert banner, identifies acute chest pain, and advises immediate in-person clinical evaluation without false reassurance.", "expected": "Alert banner: CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP\nConsultation status: High Risk (Urgent clinical evaluation required) — Zero false reassurance", "images": [{"url": "static/images/5-Workshop/5.5-Operations/24-live-clinical-triage-redflag.jpg", "caption": "Figure 24 (Live App) — MediMind live HTTPS app triggering immediate High-Risk Emergency Triage warning for acute chest pain"}]}]}
report_index: 4
report_shared: {"code": "5.5"}
title: "Testing & Validation"
date: "2026-11-28"
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

# 5.5. CloudWatch Operations, Cost Governance & Validation

Complete the Operational Excellence & Cost Optimization pillars for MediMind Clinical AI with CloudWatch Log Group (14-day retention), SNS Alerting, 3 CloudWatch Metric Alarms (EC2/ALB/RDS), and a $20/month AWS Budget — all within AWS Free Tier ($0).

## Step 5.5.1 — Create CloudWatch Log Group (14-Day Retention) & SNS Alert Topic

Create `/medimind/clinical-api` with a `14-day` retention policy to cap log storage costs and provision SNS Topic `medimind-ops-alerts` subscribed to an operations email for real-time incident notifications (AWS Free Tier).

```bash
export AWS_REGION=ap-southeast-1
aws logs create-log-group --log-group-name /medimind/clinical-api --region $AWS_REGION 2>/dev/null || true
aws logs put-retention-policy --log-group-name /medimind/clinical-api --retention-in-days 14 --region $AWS_REGION
SNS_ARN=$(aws sns create-topic --name medimind-ops-alerts --region $AWS_REGION --query 'TopicArn' --output text)
aws sns subscribe --topic-arn "$SNS_ARN" --protocol email --notification-endpoint "<YOUR_ALERT_EMAIL>" --region $AWS_REGION
aws logs describe-log-groups --log-group-name-prefix /medimind --region $AWS_REGION --query 'logGroups[*].{LogGroup:logGroupName,RetentionDays:retentionInDays}' --output table
```

**Expected Output:**
```text
LogGroup: /medimind/clinical-api | RetentionDays: 14
SubscriptionArn: pending confirmation (confirm via email link)
```

![Figure 16a (CLI) — Creating CloudWatch Log Group /medimind/clinical-api (14 days) & SNS Topic](../../../static/images/5-Workshop/5.5-Operations/cli-steps/16a-cli-cw-logs-sns.png)
*Figure 16a (CLI) — Creating CloudWatch Log Group /medimind/clinical-api (14 days) & SNS Topic*

![Figure 16a (Email) — AWS Notification Subscription Confirmation email received](../../../static/images/5-Workshop/5.5-Operations/16a-sns-email.png)
*Figure 16a (Email) — AWS Notification Subscription Confirmation email received*

## Step 5.5.2 — Configure 3 Core CloudWatch Metric Alarms (EC2, ALB, RDS)

Deploy 3 automated alarms covering all 3 tiers (`medimind-ec2-high-cpu > 80%`, `medimind-alb-unhealthy-hosts >= 1`, `medimind-rds-high-cpu > 80%`) routed to `medimind-ops-alerts` (within the 10-alarm AWS Free Tier).

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

**Expected Output:**
```text
medimind-alb-unhealthy-hosts (UnHealthyHostCount >= 1.0)
medimind-ec2-high-cpu (CPUUtilization > 80.0)
medimind-rds-high-cpu (CPUUtilization > 80.0)
```

![Figure 16b (CLI) — Creating 3 CloudWatch Metric Alarms for EC2, ALB, and RDS](../../../static/images/5-Workshop/5.5-Operations/cli-steps/16b-cli-cw-alarms.png)
*Figure 16b (CLI) — Creating 3 CloudWatch Metric Alarms for EC2, ALB, and RDS*

![Figure 16 (Console) — All 3 CloudWatch Alarms with Actions enabled](../../../static/images/5-Workshop/5.5-Operations/16-cw-alarms.png)
*Figure 16 (Console) — All 3 CloudWatch Alarms with Actions enabled*

## Step 5.5.3 — Configure AWS Budgets ($20/month) & Cost Governance

Create `medimind-monthly-budget-20usd` ($0 free) alerting via email whenever actual spend crosses **80% ($16)** or forecasted spend crosses **100% ($20)**. Full cost breakdown (`~$16.86` ~40h demo · `$207.28` live 24/7 · `$228.76` Full Multi-AZ) is published on [AWS Pricing Calculator](https://calculator.aws/#/estimate?id=ec26a26022ee93e0eb9785a0048f4dca71eb6fb3).

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

**Expected Output:**
```text
Limit: 20.0 USD | Name: medimind-monthly-budget-20usd | Type: COST | Status: OK / Healthy
```

![Figure 17a (CLI) — Creating AWS Budget medimind-monthly-budget-20usd ($20/month)](../../../static/images/5-Workshop/5.5-Operations/cli-steps/17a-cli-budgets.png)
*Figure 17a (CLI) — Creating AWS Budget medimind-monthly-budget-20usd ($20/month)*

![Figure 17 (Console) — AWS Billing Budgets overview showing OK / Healthy ($20.00)](../../../static/images/5-Workshop/5.5-Operations/17-budgets.png)
*Figure 17 (Console) — AWS Billing Budgets overview showing OK / Healthy ($20.00)*

## Step 5.5.4 — Live Infrastructure Validation (/health, /ready, CloudFront) & AWS WAF SQLi Blocking (403)

Verify the live deployment at `https://d2typ9pphzk4as.cloudfront.net` across two live tests: **(1)** confirm full-stack readiness from CloudFront Edge (`SGN50-P1`) → ALB → 2 EC2 nodes → ChromaDB RAG (`circuit_breaker: CLOSED`) with strict HTTP Security Headers (`Content-Security-Policy`, `X-Frame-Options: SAMEORIGIN`); and **(2)** trigger a live SQL Injection probe (`/?id=1' or '1'=1`) to prove **AWS WAF (`medimind-edge-waf`)** blocks it at the edge with **`403 Forbidden`** and logs `BLOCK` events under **`AWSManagedRulesSQLiRuleSet`**.

```powershell
# 1. Verify /health, /ready, HTTP Security Headers & CloudFront Edge POP (PowerShell)
curl.exe -s https://d2typ9pphzk4as.cloudfront.net/health; echo ""
curl.exe -s https://d2typ9pphzk4as.cloudfront.net/ready; echo ""
curl.exe -sI https://d2typ9pphzk4as.cloudfront.net/static/css/style.css | Select-String -Pattern "HTTP/|x-cache|x-amz-cf-pop|x-frame-options|content-security-policy"

# 2. Simulate SQL Injection via browser or curl (expected 403 Blocked at CloudFront/WAF)
curl.exe -i "https://d2typ9pphzk4as.cloudfront.net/?id=1%27%20OR%20%271%27=%271"
```

**Expected Output:**
```text
{"status":"ok"}
{"status":"ready","rag":"ready","gemini":"ready","circuit_breaker":"CLOSED"}
HTTP/1.1 200 OK | x-frame-options: SAMEORIGIN | X-Amz-Cf-Pop: SGN50-P1
SQLi Test -> HTTP/1.1 403 Forbidden (Request blocked) | AWS WAF Sampled requests: SQLiRuleSet -> BLOCK
```

![Figure 22 (Live CLI) — Verified /health ok, /ready ready (RAG & Circuit Breaker CLOSED), CSP/X-Frame-Options & CloudFront POP SGN50-P1](../../../static/images/5-Workshop/5.5-Operations/22-live-health-cloudfront.jpg)
*Figure 22 (Live CLI) — Verified /health ok, /ready ready (RAG & Circuit Breaker CLOSED), CSP/X-Frame-Options & CloudFront POP SGN50-P1*

![Figure 23a (Browser) — Live SQL Injection test (/?id=1' or '1'=1) blocked at CloudFront edge with 403 ERROR](../../../static/images/5-Workshop/5.5-Operations/23a-waf-sqli-403.png)
*Figure 23a (Browser) — Live SQL Injection test (/?id=1' or '1'=1) blocked at CloudFront edge with 403 ERROR*

![Figure 23b (Console) — AWS WAF medimind-edge-waf Sampled requests logging BLOCK actions under AWSManagedRulesSQLiRuleSet](../../../static/images/5-Workshop/5.5-Operations/23b-waf-sampled-block.jpg)
*Figure 23b (Console) — AWS WAF medimind-edge-waf Sampled requests logging BLOCK actions under AWSManagedRulesSQLiRuleSet*

## Step 5.5.5 — Live Clinical Safety Validation: Emergency Red-Flag Triage (Double Guardrails)

Validate clinical safety guardrails live at `https://d2typ9pphzk4as.cloudfront.net/#chat` using a high-risk chest pain scenario (*"Tôi đau thắt ngực dữ dội, vã mồ hôi lạnh và khó thở lan ra cánh tay trái"*). MediMind immediately triggers **Double Guardrails**, raises the **`CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP`** urgent alert banner, identifies acute chest pain, and advises immediate in-person clinical evaluation without false reassurance.

**Expected Output:**
```text
Alert banner: CẢNH BÁO NGUY CƠ CAO: CẦN ĐƯỢC ĐÁNH GIÁ Y TẾ KHẨN CẤP
Consultation status: High Risk (Urgent clinical evaluation required) — Zero false reassurance
```

![Figure 24 (Live App) — MediMind live HTTPS app triggering immediate High-Risk Emergency Triage warning for acute chest pain](../../../static/images/5-Workshop/5.5-Operations/24-live-clinical-triage-redflag.jpg)
*Figure 24 (Live App) — MediMind live HTTPS app triggering immediate High-Risk Emergency Triage warning for acute chest pain*
