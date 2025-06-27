# 🔒 HIPAA & Privacy Act Compliance

## Executive Summary

This referral automation system is fully aligned with:

* **HIPAA (USA)**
* **Australian Privacy Act 1998 (APPs)**

Key safeguards ensure confidentiality, integrity, and availability of sensitive health information. No personally identifiable information (PII/PHI) is stored persistently.

---

## 🔐 Key Safeguards

| Control Area            | Implementation Detail                                       |
| ----------------------- | ----------------------------------------------------------- |
| Encryption (in transit) | TLS/HTTPS across all APIs and AWS services                  |
| Encryption (at rest)    | AES-256 in Amazon S3 and Secrets Manager                    |
| Secure Credentials      | AWS Secrets Manager + least privilege IAM roles             |
| Logging & Auditing      | Amazon CloudWatch + AWS CloudTrail integration              |
| Data Minimization       | Only referral-relevant data extracted via Textract/LLM      |
| Data Residency          | All data processed and stored in ap-southeast-2 (Australia) |
| Retention Policy        | No PHI is retained post-delivery; transient processing only |
| Monitoring & Alerts     | Lambda + CloudWatch logs + breach alarms                    |

---

## ✅ HIPAA-Eligible AWS Services

| AWS Service         | HIPAA Eligible | Notes                                               |
| ------------------- | -------------- | --------------------------------------------------- |
| AWS Textract        | ✅              | Must be covered by BAA, no data retained post-OCR   |
| AWS Bedrock (Haiku) | ✅              | Does not store inputs; no model fine-tuning on data |
| AWS Secrets Manager | ✅              | Stores credentials encrypted with AES-256           |
| AWS Lambda          | ✅              | Ephemeral execution, logs stream to CloudWatch      |
| Amazon S3           | ✅              | Encrypted + versioning + bucket policy enforced     |

---

## 🔁 Breach Monitoring & Incident Response

- CloudWatch monitors all Lambda and Textract activity.
- Alarms configured for unusual behavior or data access anomalies.
- Defined incident response procedures (e.g., disable IAM user, revoke secrets, notify team).

---

## Appendix: Detailed Regulatory Mapping

<details>
<summary>Click to expand mapping to HIPAA and Australian APPs</summary>

### ✅ HIPAA Safeguards Mapping

| Safeguard Type        | Implementation                                                           |
|-----------------------|--------------------------------------------------------------------------|
| Technical Safeguards  | TLS, encryption, IAM, audit logging, secrets rotation                    |
| Administrative        | BAA, defined access roles, incident response policy                      |
| Physical              | Covered by AWS data center controls under shared responsibility model   |

### ✅ Australian Privacy Principles (APPs)

| APP Principle                     | Implementation                                      |
|----------------------------------|-----------------------------------------------------|
| APP 1: Open & Transparent Mgmt.  | This document + internal privacy documentation     |
| APP 6: Use of Info               | Only for referral parsing and document extraction  |
| APP 11: Security of Info         | IAM, S3 encryption, logging, no data retention      |
| APP 8: Cross-border disclosure   | None; data stored only in Australian AWS region     |

</details>

---

### 🛡️ AWS Compliance Certificates & BAA

- To download official HIPAA, SOC, ISO certifications:  
  👉 [AWS Artifact Portal](https://console.aws.amazon.com/artifact/)

- To process PHI (Protected Health Information), ensure you have a signed  
  👉 [Business Associate Addendum (BAA)](https://aws.amazon.com/compliance/hipaa-compliance/) with AWS

- For additional certifications and frameworks covered by AWS, visit  
  👉 [AWS Compliance Programs](https://aws.amazon.com/compliance/programs/)

---


## 📅 Policy Metadata

* **Version:** 1.0
* **Maintainer:** Privacy Officer — [compliance@example.com](mailto:compliance@example.com)
* **Created:** June 2025
* **Next Review Due:** June 2026

---

## 🧾 Summary

This system processes referral data using HIPAA-eligible services, secure architecture, and minimal data exposure. Logs and access are monitored continuously. No data is retained. Use this documentation for internal security reviews, audits, or third-party compliance validation.
