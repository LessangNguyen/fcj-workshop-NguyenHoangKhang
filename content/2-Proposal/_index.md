---
title: "Proposal"
date: 2026-09-14
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# MediMind Clinical AI  
## Unified AWS Clinical Assistant, Prescription Cross-Verification & Smart Healthcare Platform  

### 1. Executive Summary  
MediMind Clinical AI is designed to support initial symptom triage, prescription/lab-result cross-verification, and personalized clinical nutrition for patients and doctors in Vietnam. The platform combines a **Hybrid RAG architecture (BM25 + ChromaDB Vector + Cross-Encoder Reranker)** indexed over 364 standardized medical documents (*Vietnam Ministry of Health, kcb.vn, MedlinePlus NIH*), **Clinical Triage & Double Guardrails**, and an **8-Step Multimodal OCR Verification Pipeline** running EasyOCR in parallel with Vision LLMs (`nvidia/nemotron-3.5-super-vl:free` via OpenRouter and Groq API). Containerized with Docker and deployed on **Amazon Web Services (AWS)** in the `ap-southeast-1` (Singapore) region, the system leverages a 3-tier isolated VPC architecture compliant with the **AWS Well-Architected Framework** and **HIPAA** healthcare data security standards.  

### 2. Problem Statement  
*Current Problem*  
Frontline public hospitals frequently operate at 150% – 200% capacity, resulting in 2 – 4 hour patient wait times and a lack of structured pre-consultation medical histories for doctors. Meanwhile, self-diagnosing through unverified AI tools leads to missed emergency "golden hours," antibiotic resistance, or dosage errors from blurry handwritten prescriptions. Standard chatbots also struggle with negation traps (triggering false alarms when users type *"no shortness of breath"*) or hallucinating medical citations and allergenic meal recommendations.  

*Solution*  
The platform delivers a closed-loop healthcare workflow across five core modules: (1) Multi-turn symptom intake and 3-tier risk stratification (**Green – Yellow – Red**), handling unaccented Vietnamese and negation phrases to trigger immediate **Emergency 115** alerts; (2) An 8-step OCR pipeline running EasyOCR and Vision LLMs in parallel, cross-checking fields, flagging discrepancies (`needs_review`), and generating visual medication schedules (*Morning – Noon – Afternoon – Evening; Before/After meals*) upon user confirmation; (3) Grounded medical Q&A backed by 364 verified clinical documents; (4) Clinical nutrition recommendations using deterministic *Fail-Closed Allergy Filtering* across **2,462 Vietnamese dishes and drinks (ViFoodRec)**; and (5) A 30-second Pre-Consultation Summary linked to appointment booking across **2,144 healthcare facilities** (1,505 in HCMC and 639 in Hanoi). The workload runs on **Amazon EC2 + EBS** and **Amazon RDS PostgreSQL**, protected at the edge by **AWS WAF**, **CloudFront**, and **ALB**.  

*Benefits and Return on Investment (ROI)*  
The solution reduces initial medical history intake to 30 seconds, minimizes home medication errors, and prevents adverse drug/food allergy events via *Human-in-the-Loop* confirmation. By storing the ChromaDB vector index directly on **Amazon EBS** instead of costly managed vector databases and utilizing `:free` Vision/Chat models on OpenRouter, monthly Dev/PoC infrastructure costs remain at **$16.26/month** ($195.12/12 months) and achieve a unit cost of **$0.00418 per consultation session** at a production scale of 10,000 monthly users—saving over 95% of initial clinical AI deployment costs.  

### 3. Solution Architecture  
The platform adopts an **AWS 3-Tier VPC (`10.0.0.0/16`)** architecture in `ap-southeast-1` to completely isolate compute and database layers from direct public Internet access. Incoming user traffic routes through **Amazon Route 53**, **AWS WAF**, and **Amazon CloudFront** to an **Application Load Balancer (ALB)** in the `Public Subnet`, forwarding internal requests on port `8000` to a Docker container on **Amazon EC2** inside the `Private App Subnet`. The **ChromaDB vector index (`./medical_vectordb`)** and uploaded files (`./uploads`) persist on an attached **Amazon EBS (`gp3`)** volume with automated backups to **Amazon S3**, while five transactional tables reside in **Amazon RDS for PostgreSQL** (`SQL :5432`) inside the `Private Database Subnet`. All outbound requests to **OpenRouter / Gemini AI**, **Google OAuth 2.0**, and **Gmail SMTP / SpeedSMS** traverse a **NAT Gateway**.  

<img width="2560" height="1674" alt="architecture" src="https://github.com/user-attachments/assets/cefbdeed-7135-4709-812c-856d00c76c3b" />


*AWS Services Used*  
- *Amazon Route 53 & Amazon CloudFront*: High-availability DNS routing, static asset edge caching (1 TB free/month), and backend origin masking.  
- *AWS WAF & AWS Certificate Manager (ACM)*: Layer-7 firewall defending against OWASP Top 10 and rate abuse, paired with free SSL/TLS 1.3 certificates.  
- *Application Load Balancer (ALB) & NAT Gateway*: Handles TLS termination in the `Public Subnet` and provides secure outbound internet access for the `Private App Subnet`.  
- *Amazon EC2 & Amazon EBS (`gp3`)*: Hosts the containerized application (`PORT=8000`) with high-speed block storage for ChromaDB (364 documents) and `/uploads`.  
- *Amazon RDS for PostgreSQL*: Fully managed relational database isolated in the `Private Database Subnet` (`SQL :5432`) storing user, document, intake, and appointment records.  
- *Amazon S3 & Amazon ECR*: Stores encrypted backups/archives of medical uploads and RAG snapshots, plus private Docker container images.  
- *AWS Secrets Manager, CloudWatch & AWS IAM*: Centralizes runtime API secrets, collects PII-redacted application logs, and enforces least-privilege access via an `EC2 IAM Role`.  

*Component Design*  
- *Edge & Security Layer*: Route 53, WAF, CloudFront, and ALB filter malicious traffic and enforce end-to-end HTTPS encryption.  
- *Application & AI Layer (`Private App Subnet`)*: Python backend (`app.py` / `core/main_fastapi.py`) orchestrating Double Guardrails, Hybrid RAG (BM25 + ChromaDB + BGE Reranker), 8-step OCR cross-checking, and Circuit Breaker fallbacks.  
- *Data & Storage Layer (`Private Database Subnet` & Storage)*: RDS PostgreSQL manages five core tables (`users`, `medical_documents`, `clinical_consultations`, `intake_sessions`, `medical_appointments`); EBS provides low-latency vector storage; S3 retains long-term backups.  
- *External Integrations*: Google OAuth 2.0 login, 6-digit OTP verification via Gmail SMTP / SpeedSMS, multi-key AI inference via OpenRouter (`:free`), and interactive facility mapping across 2,144 clinics via Google Maps Platform API.  

### 4. Technical Implementation  
*Implementation Phases*  
The project was executed from clinical core engineering to AWS cloud deployment across four phases:  
1. *Research & Architecture Design*: Standardized medical datasets, designed the AWS 3-Subnet VPC architecture, 5-table PostgreSQL schema, and REST API specifications (Weeks 1–2).  
2. *Clinical Core & RAG Development*: Built Double Guardrails with negation handling, indexed 364 medical documents in Hybrid RAG, and engineered the 8-step OCR cross-verification pipeline (Weeks 3–6).  
3. *Feature Expansion & Web SPA*: Integrated 2,462 ViFoodRec Vietnamese nutrition recipes, 2,144 healthcare facilities, appointment booking, OAuth/OTP authentication, and the Web SPA (Weeks 7–8).  
4. *Containerization, AWS Deployment & Verification*: Packaged Docker images to Amazon ECR, provisioned EC2 + EBS + RDS + ALB + WAF + Secrets Manager, and validated 78 automated test files against quality gates (Weeks 9–12).  

*Technical Requirements*  
- *Backend & AI Pipeline*: Python (`Flask` / `FastAPI` gateway, `SQLAlchemy`), `ChromaDB`, `rank-bm25`, `BAAI/bge-reranker-v2-m3`, `EasyOCR`, and multi-key OpenRouter integration (`OPENROUTER_CHAT_API_KEY`, `OPENROUTER_OCR_API_KEY`, `OPENROUTER_NUTRITION_API_KEY`) protected by Circuit Breaker and Retry policies.  
- *File & Data Hardening*: Magic-byte binary validation, anti-Decompression/Zip Bomb limits (images $\le 25\text{ MP}$, PDFs $\le 30\text{ pages}$, DOCX compression ratio $< 50\text{x}$), CSRF protection, and automated PII redaction in logs.  
- *AWS Infrastructure*: VPC `10.0.0.0/16` in `ap-southeast-1`, strict Security Group chaining across ALB (`443`), EC2 (`8000`), and RDS (`5432`), and runtime secret injection via AWS Secrets Manager and EC2 IAM Role.  

### 5. Roadmap & Milestones  
- *Pre-Internship (Month 0)*: Assessed frontline hospital triage bottlenecks and curated 364 clinical guidelines (*Vietnam Ministry of Health, MedlinePlus NIH*), 2,462 Vietnamese recipes (*ViFoodRec*), and 2,144 medical facilities.  
- *Internship (Months 1–3 / 12 Weeks)*:  
  - Month 1 (Weeks 1–4): Designed the AWS 3-Subnet VPC architecture, core Clinical Triage engine, Double Guardrails, and relational database schema.  
  - Month 2 (Weeks 5–8): Completed Hybrid RAG, 8-step OCR pipeline, Clinical Nutrition Studio, facility booking, and Web SPA interface.  
  - Month 3 (Weeks 9–12): Pushed Docker images to Amazon ECR, deployed on EC2 + EBS + RDS + ALB + WAF, executed 78 automated test files (`pytest`), and prepared the Workshop Live Demo.  
- *Post-Deployment*: Expand interoperability with HL7 FHIR health record standards and pilot deployment in satellite outpatient clinics.  

### 6. Budget Estimation  
Costs were estimated via **AWS Pricing Calculator** in the **Singapore (`ap-southeast-1`)** region across two scenarios: **Dev/PoC Workshop** (leveraging AWS Free Tier and on-demand ALB/NAT demo hours) and **Production** (10,000 monthly active users running 24/7).  

*Infrastructure Costs (Dev / PoC Workshop Environment)*  
- Amazon EC2 (`t3.micro` / `t3.small`, port 8000): $0.00 – $3.17/month (covered by 750 hours/month `t3.micro` Free Tier).  
- Amazon EBS (`gp3` 20 GB for ChromaDB & `/uploads`): $0.00/month (within 30 GB Free Tier).  
- Amazon RDS for PostgreSQL (`db.t4g.micro`, port 5432): $0.00/month (covered by 750 hours + 20 GB Free Tier).  
- Application Load Balancer & NAT Gateway (~40 demo/testing hours): $3.37/month.  
- Amazon S3 (`Backup / Archive` ~2 GB) & Amazon ECR (~450 MB): $0.00/month (within Free Tier limits).  
- AWS Secrets Manager (3 secret groups): $1.20/month.  
- Amazon CloudFront (15 GB), ACM (TLS 1.3), CloudWatch & AWS IAM: $0.00/month (Always-Free / Free Tier).  
- Amazon Route 53 (1 Hosted Zone + 50,000 queries): $0.52/month.  
- AWS WAF (1 Web ACL + 3 Managed Rules): $8.00/month.  
- External AI & Auth Services (OpenRouter `:free`, Gmail SMTP, Google OAuth): $0.00/month.  

*Total*: **$16.26/month**, **$195.12/12 months**  
- *Production Scaling Estimate (10,000 users / 25,000 consultations / 8,000 OCR scans per month, 24/7)*: **$104.60/month** (**$0.00418 per consultation session**).  

### 7. Risk Assessment  
*Risk Matrix*  
- Missed emergency red flags or negation traps (*"no shortness of breath"*): High impact, medium probability.  
- OCR misinterpretation on blurry or glaring handwritten prescriptions: High impact, medium probability.  
- Malicious file uploads (Decompression/Zip Bombs) or unauthorized record access (IDOR): High impact, low probability.  
- External AI provider outages (OpenRouter/Gemini) or budget overruns: Medium impact, low probability.  

*Mitigation Strategies*  
- Clinical & Negation Safety: Double Guardrails (`core/triage.py`) validated at 100% Emergency Recall and 100% Negation Accuracy across 100 benchmark cases, strictly blocking unauthorized antibiotic prescriptions.  
- Prescription OCR Verification: Parallel EasyOCR + Vision LLM execution with automated `needs_review=True` flagging on discrepancies and mandatory *Human-in-the-Loop* user confirmation before generating medication schedules.  
- Security & Cost Control: Strict Private Subnet isolation for EC2 and RDS, magic-byte & size validation (`core/storage_service.py`), PII log redaction, and **AWS Budgets** alerts set at a $20 threshold.  

*Contingency Plan*  
- Trigger the **Circuit Breaker** (`core/circuit_breaker.py`) after 5 consecutive external API failures to fail over seamlessly to the local **Fallback Clinical Engine** and deterministic nutrition filtering without UI downtime.  
- Rapidly restore ChromaDB indices and relational records from encrypted **Amazon S3** backups and automated **Amazon RDS** snapshots.  

### 8. Expected Results  
*Technical Improvements*  
Deliver a production-ready multimodal clinical AI platform passing **78 automated test files** and four quantitative benchmarks (308 test samples), achieving **100% Emergency Recall**, **100% Negation Accuracy**, **0.0% unsafe advice rate**, **Holdout Recall@5 $\ge$ 85%**, and an average triage latency under **2.5 seconds**.  

*Long-Term Value*  
Establish a secure, cost-efficient AWS reference architecture (**~$0.00418 per session** at 10,000-user scale) that can be reused and scaled across outpatient clinics and primary healthcare facilities to reduce triage congestion and improve medication safety.
