# 🛡️ Multi-Stage Web Attack: XSS to Admin Takeover & RCE

This project documents a high-impact **Attack Chain** discovered and exploited within a simulated laboratory environment (WorldWap). It demonstrates the critical danger of combining minor vulnerabilities to achieve a full system compromise.

## 🔗 Technical Documentation
Detailed Research: [Full Exploitation Report](./Multi-Stage-Web-Attack-XSS-to-Admin-Takeover-and-RCE.md)

## ☣️ Attack Vector Overview
The exploitation follows a structured **Cyber Kill Chain**:

1.  **Initial Access:** Exploiting a **Stored XSS** in the birthday component.
2.  **Session Hijacking:** Stealing the Administrator's cookie via a JavaScript payload.
3.  **Privilege Escalation:** Gaining access to the Admin Dashboard.
4.  **Remote Code Execution (RCE):** Leveraging admin permissions to upload a reverse shell.



## 🛠️ Tools & Technologies
* **Enumeration:** Nmap, Gobuster.
* **Interception:** Burp Suite Professional.
* **Payloads:** Custom JavaScript, Base64 Obfuscation, PHP Reverse Shells.
* **Environment:** Kali Linux.

## 🛡️ Mitigation & Remediation
This research includes a comprehensive guide on implementing technical controls:
* **Input Sanitization** (DOMPurify).
* **Content Security Policy (CSP)**.
* **Anti-CSRF Tokens**.
* **HttpOnly & Secure Flags** for session management.

---
**Disclaimer:** This research is for educational purposes only. All testing was conducted in a controlled, authorized environment.
**Researcher:** Victor Hugo Mierez
