# 🛡️ Cyber Threat Log Parser

**A powerful, standalone Python tool to parse virtually any log file format and detect bad actor activity & cyber threats.**

[![Python 3.8+](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![No External Dependencies](https://img.shields.io/badge/Dependencies-None-brightgreen.svg)]()

---

## 🚀 Overview

In cybersecurity, logs are the primary source of truth for detecting intrusions, brute-force attacks, web exploitation, and automated scanning. 

**Cyber Threat Log Parser** provides a lightweight, portable "mini-SIEM" capability that works offline, requires zero external libraries, and handles real-world log chaos:

- Apache / Nginx access logs (Combined Log Format)
- Linux `auth.log`, `syslog`, `messages`
- JSON-structured logs (modern apps, Kubernetes, CloudTrail, API gateways, Docker)
- Generic / unstructured text logs (custom applications, firewalls, VPNs, databases)
- Gzipped (`.gz`) files supported natively

It combines **signature-based detection**, **behavioral heuristics**, and **basic reputation checking** to surface real threats quickly.

Perfect for:
- Incident response & forensics
- Proactive threat hunting on archived logs
- Air-gapped / offline environments
- Small teams or homelabs without enterprise SIEM
- Quick triage before feeding into Splunk, Elastic, or Chronicle

---

## ✨ Key Features

### Multi-Format Parsing (Auto-Detection)
- Intelligent cascading parser (JSON → Apache/Nginx → Syslog → Generic fallback)
- Extracts IP, timestamp, HTTP method/path/status, User-Agent, message, etc.
- Robust timestamp parsing across 12+ formats (with automatic year inference for classic syslog)

### Comprehensive Threat Detection

**Signature-Based:**
- SQL Injection (UNION, OR 1=1, DROP TABLE, hex payloads, comments, tautologies, etc.)
- Cross-Site Scripting (XSS) — `<script>`, `javascript:`, `onerror=`, `document.cookie`, `eval()`, etc.
- Path Traversal / LFI / RFI (`../`, `/etc/passwd`, `/proc/self/environ`, wrapper protocols)
- Sensitive path access (`/.env`, `/admin`, `/wp-admin`, `/config.php`, backups, shells, `.git`)
- Malicious / scanner User-Agents (`sqlmap`, `nikto`, `gobuster`, `nmap`, `Acunetix`, `python-requests` in suspicious contexts, etc.)
- Brute-force keywords in **any** log ("Failed password for invalid user", "authentication failure", etc.)

**Behavioral & Heuristic:**
- Configurable brute-force detection (failed auth volume within time window)
- High-volume / scanning / potential DoS from single IP
- Rapid successive requests (automation/bot behavior)
- Multi-vector attacks (same IP triggering multiple rule categories)
- Failed authentication on sensitive endpoints

**Reputation:**
- Load custom bad-IP blocklists
- Built-in example malicious IPs (easy to extend with real threat intel)

**Additional:**
- Whitelisting for legitimate bots (Googlebot, Bingbot, uptime monitors, etc.)
- Full JSON or CSV export with line numbers and raw context
- Verbose mode + rich console reporting with actionable recommendations

---

## 📦 Installation

**Zero dependencies** — just Python 3.8+.

```bash
# Clone the repository
git clone https://github.com/dutchvonrudder/cyber-threat-log-parser.git
cd cyber-threat-log-parser

# Or download the single script directly
curl -O https://raw.githubusercontent.com/dutchvonrudder/cyber-threat-log-parser/main/cyber_threat_log_parser.py
```

That's it. No `pip install`, no virtualenv required.

---

## 🖥️ Usage

### Basic Usage
```bash
python cyber_threat_log_parser.py /var/log/nginx/access.log
```

### Advanced Examples
```bash
# Multiple files + compressed logs + custom thresholds
python cyber_threat_log_parser.py logs/*.log.gz \
    --format auto \
    --brute-force-threshold 3 \
    --time-window 2 \
    --output threats.json

# Linux auth.log / SSH brute force focus
python cyber_threat_log_parser.py /var/log/auth.log --format syslog --verbose

# With external bad-IP list + full detailed export
python cyber_threat_log_parser.py app.log \
    --bad-ips my_blocklist.txt \
    --full \
    --output full_report.json

# Generic / custom application logs
python cyber_threat_log_parser.py custom_app.log --format generic
```

### Command-Line Options
```
--format {auto,apache,nginx,syslog,json,generic}   Log format (default: auto)
--brute-force-threshold INT                        Failed attempts in window (default: 5)
--time-window INT                                  Time window in minutes (default: 5)
--high-volume-threshold INT                        Requests to flag high volume (default: 80)
--bad-ips FILE                                     Text file with one bad IP per line
--output FILE                                      Export full report (.json or .csv)
--full                                             Include ALL detailed threats (can be large)
--all                                              Report every IP-bearing entry (baselining)
--verbose, -v                                      Show progress & debug info
```

---

## 📊 Example Output

When run against a mixed log containing SQLi, XSS, path traversal, scanner activity, and SSH brute force, the tool produces:

```
══════════════════════════════════════════════════════════════════════════════
           🛡️  CYBER THREAT LOG ANALYSIS REPORT  🛡️
══════════════════════════════════════════════════════════════════════════════
Files analyzed          : 1
Total lines scanned     : 44
Entries parsed          : 32
Suspicious entries      : 28
Unique suspicious IPs   : 11
...
🔴 TOP SUSPICIOUS IPs (sorted by severity + volume)

 1. 10.0.0.50
    Requests     : 3
    Reasons      : failed_auth_or_forbidden, malicious_user_agent, multi_vector, sensitive_path_access, sql_injection
    ...

 4. 203.0.113.50
    Requests     : 6
    Reasons      : brute_force (6 attempts in 5 min), brute_force_attempt, known_bad_ip, multi_vector
    ...
```

Full structured JSON export includes line numbers, raw log lines, and parsed fields for easy correlation with other tools.

---

## 🧠 How It Works (Technical Overview)

1. **Parsing Layer** (`LogParser` class)
   - Tries JSON → Apache regex → Syslog regex → Generic IP/timestamp/request extraction
   - Normalizes fields across formats

2. **Detection Layer** (`ThreatDetector` class)
   - Compiles 50+ regex patterns for signatures
   - Scans combined text (request + path + message + UA + original line)
   - Applies behavioral rules on per-IP activity timelines

3. **Aggregation & Reporting**
   - Groups by IP with timestamps
   - Applies sliding-window brute force, rate, and multi-vector logic
   - Ranks by severity + volume

The entire tool is ~550 lines of clean, well-documented Python — easy to audit, fork, or extend.

---

## ⚠️ Limitations & Honest Assessment

- **False positives**: Legitimate scanners, pentests, or noisy apps can trigger rules. Use whitelists and tune thresholds.
- **Binary logs** (Windows `.evtx`): Convert first with `wevtutil` or `evtx_dump`.
- **Advanced evasion**: Double-encoded or heavily obfuscated payloads may need additional rules.
- **No ML / external intel by default**: Pure rules + statistics for maximum portability. Easy to add VirusTotal/Shodan lookups.
- **Batch only**: Not a real-time streaming processor (though easy to integrate with rsyslog/filebeat).

For production SIEM-scale needs, consider feeding this tool’s JSON output into Elastic or Splunk.

---

## 🔧 Extending the Tool

- **New signatures** — Edit the `SQLI_PATTERNS`, `XSS_PATTERNS`, `MALICIOUS_USER_AGENTS`, etc. lists at the top of the script.
- **New log formats** — Add a regex or parser method in the `LogParser` class.
- **Custom rules per environment** — Load from JSON config (trivial addition).
- **Threat intel integration** — Add API calls in `ThreatDetector.detect_threats()` (requires internet + API key).
- **Dashboard** — Wrap with Streamlit/Gradio in <50 lines.

---

## 📜 License

MIT License — free to use, modify, and distribute for defensive security purposes.

---

## 🤝 Contributing

Pull requests are welcome! Please open an issue first to discuss major changes.

Ideas for contributions:
- Additional log format parsers (Windows Event Log JSON export, IIS, etc.)
- GeoIP enrichment (optional)
- HTML report generation
- Rule hot-reloading from external file
- Unit tests

---

## 📚 Related Projects & Inspiration

- [Fail2Ban](https://github.com/fail2ban/fail2ban)
- [OSSEC / Wazuh](https://wazuh.com/)
- [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/ecs-reference.html)
- Classic log analysis papers from SANS / USENIX

---

**Created with ❤️ for the defensive security community by Grok (xAI) — May 2026**

*Stay curious. Stay secure.*