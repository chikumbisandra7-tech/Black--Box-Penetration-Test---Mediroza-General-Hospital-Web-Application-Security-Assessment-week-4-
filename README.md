
# Penetration Testing Report
## Mediroza General Hospital — Web Application Security Assessment
**Week 4 Capstone | Black-Box Penetration Test | Networkwalks**

---

| Field | Details |
|---|---|
| **Prepared By** | Sandra Chkumbi |
| **Organisation** | Networkwalks |
| **Batch** | B083 |
| **Project** | Week 4 Capstone — Penetration Testing Project |
| **Target** | https://medirozahospital.com |
| **Assessment Type** | Black-Box Web Application Penetration Testing |
| **Duration** | 5 Days |
| **Date** | 1–5 October 2026 |
| **Classification** | Confidential — Authorised Personnel Only |
| **Authorisation** | Written authorisation granted by Networkwalks and Mediroza General Hospital |

> ⚠️ **Legal Disclaimer:** This assessment was conducted in a controlled educational environment with explicit written authorisation. All findings are for educational and security improvement purposes only. These techniques must never be applied to any system without explicit written permission from the owner.

---

## Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Scope and Methodology](#2-scope-and-methodology)
3. [Findings and Proof of Exploitation](#3-findings-and-proof-of-exploitation)
4. [Risk Rating Summary](#4-risk-rating-summary)
5. [Recommendations and Remediation](#5-recommendations-and-remediation)
6. [Conclusion](#6-conclusion)

---

## 1. Executive Summary

Between 1 and 5 October 2026, Sandra Chkumbi (Cohort B083) conducted a black-box penetration test against the Mediroza General Hospital web application at https://medirozahospital.com, as part of the Networkwalks Week 4 Capstone Project. The engagement simulated a real-world external attacker with no prior knowledge of the target's internal systems.

The assessment revealed multiple critical vulnerabilities that collectively resulted in complete compromise of the web application. An attacker was able to:

- Enumerate hidden directories via a publicly accessible `robots.txt` file
- Access an exposed database backup containing confidential staff salary, national ID, and shareholder data
- Exploit SQL injection in the patient portal login page
- Download three confidential encrypted patient PDF lab reports without authorisation
- Crack the encryption on all three patient PDF files
- Extract sensitive metadata from the recovered files revealing internal infrastructure details

**The overall risk posture of Mediroza General Hospital's web application is assessed as CRITICAL.**

### Risk Summary

| 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low | Total Findings |
|---|---|---|---|---|
| 4 | 1 | 1 | 1 | **7** |

---

## 2. Scope and Methodology

### 2.1 Scope

- **Target domain:** https://medirozahospital.com
- All subpages, directories, and web application functionality accessible from the root domain
- No social engineering, denial of service, or testing outside the agreed scope was performed

### 2.2 Methodology

**Phase 1 — Reconnaissance:**
Passive and active reconnaissance using `curl` to retrieve `robots.txt`, `wget` to download exposed files, and browser-based directory traversal to enumerate accessible paths.

**Phase 2 — Exploitation:**
SQL injection tested and confirmed on the Patient Portal. Directory listing vulnerabilities exploited to access `/patient/` and `/old/` directories. Exposed database backup downloaded and analysed. Three confidential patient PDFs retrieved.

**Phase 3 — Post-Exploitation:**
All three patient PDFs cracked using Networkwalks Hash Calculator and `qpdf`. `ExifTool` used to extract metadata revealing internal infrastructure and staff identity.

### 2.3 Tools Used

| Tool | Purpose |
|---|---|
| `curl` | Retrieve robots.txt and HTTP headers |
| `wget` | Download the exposed database backup |
| Browser (Firefox) | Manual recon, directory traversal, patient portal access |
| `sqlmap` | Automated SQL injection testing |
| Networkwalks Hash Calculator | Extract PDF hashes and crack via dictionary attack |
| `qpdf` | Decrypt patient_report_3.pdf |
| `ExifTool` | Extract metadata from decrypted PDFs |

---

## 3. Findings and Proof of Exploitation

---

### Finding 1 — robots.txt Discloses Sensitive Directory Paths
**Severity: 🟢 Low | CVSS: 2.7**

The `robots.txt` file at `https://medirozahospital.com/robots.txt` is publicly accessible and discloses three sensitive directory paths: `/patient/`, `/staff/`, and `/old/`. While not a security control, these entries directly pointed to the most sensitive areas of the application and served as the starting point for the entire attack chain.

**Command used:**
```bash
curl https://medirozahospital.com/robots.txt
```

**Output:**
```
# robots.txt
User-agent: *
Disallow: /patient/
Disallow: /staff/
Disallow: /old/

Sitemap: https://medirozahospital.com/sitemap.xml
```

**Screenshot:**

![robots.txt](screenshots/01-robots-txt.png)
*Figure 1.1: robots.txt revealing /patient/, /staff/, and /old/ directory paths*

**Impact:** An attacker reads `robots.txt` in the first minutes of reconnaissance and immediately knows which directories to target.

---

### Finding 2 — Directory Listing Enabled — /old/ and /patient/ Exposed
**Severity: 🔴 Critical | CVSS: 9.1**

Directory listing is enabled on the web server, allowing any unauthenticated visitor to browse the full contents of `/old/` and `/patient/`. This exposed a 2019 database backup in `/old/` and the full patient portal source structure in `/patient/` including `download.php`, `error_log`, `login.php`, and the `reports/` folder.

**URLs browsed:**
```
https://medirozahospital.com/old/
https://medirozahospital.com/patient/
```

**Screenshots:**

![/old/ directory](screenshots/02-old-directory.png)
*Figure 2.1: Directory listing of /old/ revealing mediroza_db_backup_2019.sql*

![wget database](screenshots/03-wget-database.png)
*Figure 2.2: wget downloading the exposed database backup (6.2 KB, HTTP 200 OK)*

**Impact:** Directly led to discovery of the database backup containing all staff data and the patient PDF download mechanism.

---

### Finding 3 — Exposed Database Backup — Staff Salaries and Shareholder Data
**Severity: 🔴 Critical | CVSS: 9.8**

The file `mediroza_db_backup_2019.sql` was publicly accessible at `https://medirozahospital.com/old/mediroza_db_backup_2019.sql` with no authentication. This is a complete MySQL dump of the `mediroza_hr` database containing highly sensitive records for 30 staff members and 10 shareholders.

A comment embedded in patient PDF metadata (Finding 7) confirms the cause: *"DB backup moved to /old before site migration, do not delete"* — written by j.malik (IT Systems Administrator), who placed the file in a public directory during a site migration and failed to restrict access.

**Command:**
```bash
wget https://medirozahospital.com/old/mediroza_db_backup_2019.sql
```

#### Staff Salaries Exposed (30 employees):

| Employee | Role | Monthly Salary (ZAR) |
|---|---|---|
| Dr. Johan van der Merwe | Medical Director | R 160,000 |
| Sarah Botha | CFO | R 152,000 |
| Dr. Vikram Chetty | Anaesthetist | R 135,000 |
| Dr. Rajesh Naidoo | Chief Pathologist | R 138,000 |
| Dr. Suresh Moodley | Consultant Radiologist | R 130,000 |
| Dr. Fatima Patel | Pediatrician | R 118,000 |
| Dr. Ahmed Kara | Consultant Physician | R 128,000 |
| Michael Roberts | HR Director | R 96,000 |
| Kagiso Sithole | Pharmacist | R 61,000 |
| Nisha Singh | Physiotherapist | R 48,000 |
| ... | *23 more employees* | *R 19,000 – R 74,000* |

#### Shareholder Details Exposed (10 shareholders):

| Shareholder | Shares Held | Share % | Class |
|---|---|---|---|
| Dr. Rajesh Naidoo | 180,000 | 18.0% | Ordinary |
| Cedar Health Holdings (Pty) Ltd | 150,000 | 15.0% | Ordinary |
| Dr. Johan van der Merwe | 120,000 | 12.0% | Ordinary |
| Reddy Family Trust | 110,000 | 11.0% | Ordinary |
| Thabo Molefe | 100,000 | 10.0% | Ordinary |
| Sarah Botha | 90,000 | 9.0% | Ordinary |
| Dr. Ahmed Kara | 80,000 | 8.0% | Preferential |
| Naledi Zulu | 70,000 | 7.0% | Ordinary |
| Michael Roberts | 60,000 | 6.0% | Ordinary |
| Dr. Vikram Chetty | 40,000 | 4.0% | Preferential |

**Impact:** Full financial exposure of all staff and shareholders. National ID numbers of all 30 employees also exposed. POPIA notification obligations triggered.

---

### Finding 4 — SQL Injection — Patient Portal Login Page
**Severity: 🔴 Critical | CVSS: 9.4**

The Patient Portal login page at `https://medirozahospital.com/patient/login.php` is vulnerable to SQL injection. When a crafted payload is submitted, the application passes input directly to a `mysqli_query()` call without sanitisation, returning a raw MySQL error:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the manual
that corresponds to your MySQL server version for the right syntax to use near " #' at line 1
```

This confirms direct string concatenation is used to build SQL queries from user input.

**Payload tested:**
```
Username: admin'#
Password: anything
```

**Screenshot:**

![SQL Injection](screenshots/04-sql-injection.png)
*Figure 4.1: Raw MySQL error returned by the Patient Portal confirming SQL injection*

**Impact:** Authentication bypass, potential extraction of the entire patient database including medical records and login credentials.

---

### Finding 5 — Unauthenticated Access to Confidential Patient Lab Reports
**Severity: 🔴 Critical | CVSS: 9.8**

Following exploitation, three confidential patient pathology lab reports were accessed and downloaded from the Patient Portal without valid credentials:

| Report | Lab Reference | Date |
|---|---|---|
| Pathology Report — S. Dlamini | LR-2024-1187 | 2024-11-04 |
| Pathology Report — P. Reddy | LR-2024-1192 | 2024-11-05 |
| Pathology Report — E. Thompson | LR-2024-1205 | 2024-11-06 |

**Screenshots:**

![Patient Portal](screenshots/05-patient-portal-reports.png)
*Figure 5.1: Patient portal displaying all three confidential lab reports*

![Report Opened](screenshots/06-patient-report-opened.png)
*Figure 5.2: Pathology Report for Sipho Dlamini (MG-P-10231) opened — Full Blood Count results visible*

**Impact:** Serious breach of patient confidentiality. Violates South African POPIA. Data could be used for blackmail, identity theft, or insurance fraud.

---

### Finding 6 — Weak PDF Encryption Passwords on Patient Reports
**Severity: 🟠 High | CVSS: 7.5**

All three patient PDFs were password-protected using RC4 128-bit encryption (outdated standard). All three passwords were cracked within seconds using a 100-entry dictionary wordlist:

| File | Method | Password Recovered |
|---|---|---|
| patient_report_1.pdf | Networkwalks Hash Calculator | `password` |
| patient_report_2.pdf | Networkwalks Hash Calculator | `123456` |
| patient_report_3.pdf | qpdf | `!@#$%^&` |

**Commands used:**
```bash
# Extract hash and crack online via Networkwalks Hash Calculator
# networkwalks.com/hash-calculator/

# Decrypt PDF3 with qpdf
qpdf --password='!@#$%^&' --decrypt patient_report_3.pdf report3_open.pdf
```

**Screenshots:**

![PDF2 hash](screenshots/07-pdf2-hash-extracted.png)
*Figure 6.1: Hash extracted from patient_report_2.pdf*

![123456 cracked](screenshots/08-password-123456-cracked.png)
*Figure 6.2: patient_report_2.pdf cracked — password '123456'*

![password cracked](screenshots/09-password-cracked.png)
*Figure 6.3: patient_report_1.pdf cracked — password 'password'*

![PDF3 hash](screenshots/10-pdf3-hash-extracted.png)
*Figure 6.4: Hash extracted from patient_report_3.pdf*

![qpdf decrypt](screenshots/11-qpdf-decrypt.png)
*Figure 6.5: qpdf decrypting patient_report_3.pdf with recovered password*

**Impact:** Weak passwords and outdated RC4 encryption provide no meaningful protection. All three files cracked in under 60 seconds.

---

### Finding 7 — Sensitive Metadata Embedded in Patient PDF Files
**Severity: 🟡 Medium | CVSS: 5.3**

ExifTool analysis of the decrypted `report3_open.pdf` revealed sensitive internal metadata:

| Field | Value |
|---|---|
| **Author** | j.malik |
| **Comments** | DB backup moved to /old before site migration, do not delete |
| **Creator** | Mediroza CMS 1.4.2 |
| **Producer** | Mediroza Lab Reporting Module |
| **Subject** | Full Blood Count |
| **Title** | Pathology Report — E. Thompson |

**Command:**
```bash
exiftool report3_open.pdf
```

**Screenshot:**

![ExifTool](screenshots/12-exiftool-metadata.png)
*Figure 7.1: ExifTool revealing author j.malik and internal comment referencing /old/ directory*

**Impact:** The comment directly pointed to the database backup in `/old/`, creating a secondary discovery path. Staff identity embedded in metadata enables targeted social engineering.

---

## 4. Risk Rating Summary

| # | Finding | Severity |
|---|---|---|
| 1 | robots.txt Discloses Sensitive Directory Paths | 🟢 Low |
| 2 | Directory Listing Enabled — /old/ and /patient/ Exposed | 🔴 Critical |
| 3 | Exposed Database Backup — Staff Salaries and Shareholder Data | 🔴 Critical |
| 4 | SQL Injection — Patient Portal Login Page | 🔴 Critical |
| 5 | Unauthenticated Access to Confidential Patient Lab Reports | 🔴 Critical |
| 6 | Weak PDF Encryption Passwords on Patient Reports | 🟠 High |
| 7 | Sensitive Metadata Embedded in Patient PDF Files | 🟡 Medium |

---

## 5. Recommendations and Remediation

### Finding 1 — robots.txt
Remove sensitive paths from `robots.txt`. Use server-level authentication and access controls instead. `robots.txt` is a public file and provides no security.

### Finding 2 — Directory Listing
Disable directory listing immediately on the web server:
```apache
# Apache — add to .htaccess
Options -Indexes
```
For LiteSpeed: set `noindex` in the virtual host configuration. All sensitive directories must require authentication before serving any content.

### Finding 3 — Exposed Database Backup
- **Delete** `mediroza_db_backup_2019.sql` from the web root immediately
- Store all backups in a private, offline, or cloud bucket location with strict access controls
- **Rotate all 30 staff credentials** — all accounts in the backup are compromised
- Notify affected staff that national IDs, salaries, and contact details were exposed
- Report the breach to the South African Information Regulator under **POPIA**

### Finding 4 — SQL Injection
Rewrite all database queries using parameterised prepared statements:
```php
// Vulnerable (current)
$query = "SELECT * FROM patients WHERE username = '$username'";

// Secure (fixed)
$stmt = $pdo->prepare('SELECT * FROM patients WHERE username = ?');
$stmt->execute([$username]);
```
Disable MySQL error display in PHP: `display_errors = Off` in `php.ini`. Implement a WAF as a secondary layer.

### Finding 5 — Unauthenticated Access to Patient Records
Implement session-based authentication on every page in `/patient/`. Validate a server-side session token on every request. Apply least privilege — patients should only access their own reports, verified by user ID bound to the session, not by URL parameter.

### Finding 6 — Weak PDF Passwords
- Generate cryptographically random passwords of at least 16 characters per file
- Deliver passwords via a separate secure channel (SMS or authenticated notification)
- **Upgrade PDF encryption from RC4 128-bit to AES-256**

### Finding 7 — PDF Metadata
Strip all metadata from PDFs before distribution:
```bash
qpdf --no-copy-encryption input.pdf output.pdf
# or
exiftool -all= output.pdf
```
Establish a policy that all exported documents are sanitised before delivery to patients or external parties.

---

## 6. Conclusion

This black-box penetration test of Mediroza General Hospital's web application identified seven vulnerabilities, four of which are rated Critical. The combination of directory listing, an exposed database backup, SQL injection, and unauthenticated access to patient records represents complete compromise of the application's confidentiality, integrity, and availability.

The attack chain required no sophisticated tools or prior knowledge. Within minutes of visiting the site, `robots.txt` revealed the directory structure; directory listing exposed both the database backup and the patient portal source files; and the SQL injection vulnerability confirmed the application's failure to sanitise user input. Confidential patient medical records, staff salaries, national ID numbers, and shareholder data were all recovered without authentication.

**Immediate priority actions:**
1. Disable directory listing
2. Delete the exposed database backup
3. Fix SQL injection with parameterised queries
4. Enforce authentication on all patient portal pages
5. Rotate all staff credentials

All testing was conducted within the authorised scope of the Networkwalks educational programme. No data was retained beyond the purposes of this report.

---

## 📁 Repository Structure

```
pentest-report-w4-mediroza/
├── README.md
├── report/
│   └── Sandra_Chkumbi_W4_Mediroza_PenTest_Report.docx
└── screenshots/
    ├── 01-robots-txt.png
    ├── 02-old-directory.png
    ├── 03-wget-database.png
    ├── 04-sql-injection.png
    ├── 05-patient-portal-reports.png
    ├── 06-patient-report-opened.png
    ├── 07-pdf2-hash-extracted.png
    ├── 08-password-123456-cracked.png
    ├── 09-password-cracked.png
    ├── 10-pdf3-hash-extracted.png
    ├── 11-qpdf-decrypt.png
    └── 12-exiftool-metadata.png
```

---

## 🔒 Disclaimer

All activities documented in this report were performed strictly within the authorised scope of the Networkwalks cybersecurity educational programme. Written permission was obtained before any testing was conducted. This repository is for educational and portfolio purposes only. Unauthorised penetration testing is illegal.

---

*Sandra Chkumbi · Cybersecurity intern · Cohort B083 · Networkwalks · Week 04 Capstone · Black-Box Pentest — Mediroza General Hospital*
