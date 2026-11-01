---
title: "Worklog Week 5"
date: 2026-10-12
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---
{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}


### Week 5 Objectives:

- Learn and practice Infrastructure as Code (IaC) provisioning using **AWS CloudFormation**.
- Set up centralized identity management and Single Sign-On (**AWS IAM Identity Center / SSO**) across organizational accounts.
- Enforce security boundaries and restrict maximum IAM permissions using **Permission Boundaries**.
- Restrict and control role assumption workflows based on fine-grained conditions.
- Monitor, assess, and audit security compliance standards using **AWS Security Hub**.

### Tasks Implemented This Week:
| Day | Work Details | Start Date | Completion Date | Reference Link |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Research concepts and practice infrastructure provisioning as code using **AWS CloudFormation** (writing templates, creating stacks, and managing resource lifecycles). | 12/10/2026   | 12/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| Tue | - Set up centralized access management and Single Sign-On (**AWS IAM Identity Center / SSO**) for the organization. | 13/10/2026   | 13/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| Wed | - Learn and practice limiting user and role privileges using **IAM Permission Boundaries**. | 14/10/2026   | 14/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| Thu | - Configure and restrict role assumptions (`AssumeRole`) based on specific contextual security conditions (`aws:PrincipalTag`, source parameters, etc.). | 15/10/2026   | 15/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| Fri | - Explore and evaluate security compliance standards and configuration posture findings using **AWS Security Hub**. | 16/10/2026   | 16/10/2026      | <https://cloudjourney.awsstudygroup.com/> |


### Week 5 Key Accomplishments:

- Mastered template authoring and automated end-to-end provisioning, updating, and teardown of cloud infrastructure using **AWS CloudFormation**.
- Successfully configured **AWS IAM Identity Center (SSO)** for centralized identity management and streamlined secure access across multi-account environments.
- Implemented **IAM Permission Boundaries** to establish maximum permission guardrails, effectively preventing privilege escalation risks when delegating permissions.
- Configured fine-grained policy **Conditions** to restrict role-assumption workflows, enhancing security governance over cross-role operations.
- Deployed and navigated **AWS Security Hub** to centralize security alerts and audit infrastructure compliance against industry security standards.
- Cleaned up all lab resources safely to maintain resource hygiene and avoid unexpected cloud costs.
