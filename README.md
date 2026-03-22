# AI Security Scanner

> **LLM-powered static analysis tool that leverages Google Gemini 2.5 Flash to detect security vulnerabilities in Python source code — with structured, severity-graded output delivered directly in your terminal.**

---

## What This Is

Most static analysis tools rely on predefined rule sets and pattern matching. This scanner takes a different approach: it uses a large language model to **reason about code semantics** and identify vulnerabilities the way a security engineer would — understanding context, intent, and exploitability rather than just matching known signatures.

Feed it a Python file. Get back a structured vulnerability report, color-coded by severity, in seconds.

---

## Technical Stack

| Layer | Technology |
|---|---|
| AI Model | Google Gemini 2.5 Flash (`gemini-2.5-flash`) |
| AI SDK | `google-generativeai 0.8.6` |
| Auth | `google-auth 2.49.1` + `.env` via `python-dotenv 1.2.2` |
| Terminal UI | `colorama 0.4.6` (cross-platform ANSI color) |
| Transport | `grpcio 1.78.0` + `protobuf 5.29.6` (gRPC to Google API) |
| Data Validation | `pydantic 2.12.5` |
| Crypto Primitives | `cryptography 46.0.5` |
| Environment | Python 3.13, `venv` virtual environment |
| IDE | VS Code with custom workspace settings |
| VCS | Git with `.gitignore` hardened against credential leakage |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLI Entry Point                          │
│                    python scanner.py <file>                     │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      scan_file(filepath)                        │
│  1. Read source file from disk                                  │
│  2. Inject code into structured security prompt template        │
│  3. Call Gemini API via google-generativeai SDK (gRPC/HTTP2)    │
│  4. Receive structured vulnerability report                     │
│  5. Pass response through add_colors_to_output()                │
│  6. Render to terminal with ANSI color codes                    │
└─────────────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Google Gemini 2.5 Flash                      │
│              (Generative Language API via gRPC)                 │
│  - Context-aware code reasoning                                 │
│  - Vulnerability classification by type and severity           │
│  - Remediation code generation                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works

### Prompt Engineering

The scanner uses a carefully structured prompt template that constrains Gemini's output to a machine-readable format. Each vulnerability is returned with five mandatory fields:

```
SEVERITY:     CRITICAL | HIGH | MEDIUM | LOW
TYPE:         Vulnerability category (e.g. SQL Injection, Hardcoded Credentials)
DESCRIPTION:  One-sentence explanation of the issue
IMPACT:       One-sentence description of potential damage
FIX:          Corrected code snippet
```

This prompt engineering approach ensures consistent, parseable output across diverse codebases — no post-processing regex required.

### Severity Color Mapping

```python
CRITICAL  →  colorama.Fore.RED
HIGH      →  colorama.Fore.YELLOW
MEDIUM    →  colorama.Fore.BLUE
LOW       →  colorama.Fore.GREEN
```

`colorama.init(autoreset=True)` prevents ANSI color bleed between output segments, maintaining clean terminal rendering on Windows, macOS, and Linux.

### Model Selection Rationale

`gemini-2.5-flash` was selected over heavier Gemini variants for its balance of reasoning capability and inference latency. For a CLI tool where response time matters, flash models deliver security-relevant reasoning without the overhead of full pro-tier inference.

---

## Installation

```bash
# Clone the repo
git clone https://github.com/yourusername/AI_security_scanner.git
cd AI_security_scanner

# Create and activate a virtual environment (Python 3.13)
python -m venv venv
source venv/bin/activate          # Linux/macOS
venv\Scripts\activate             # Windows

# Install dependencies
pip install -r requirements.txt

# Configure your Google API key
echo "GOOGLE_API_KEY=your_key_here" > .env
```

> API key is loaded at runtime via `python-dotenv` and never hardcoded. The `.env` file is excluded from version control via `.gitignore`.

---

## Usage

```bash
python scanner.py vulnerable.py
```

### Example Output

```
[*] Scanning: vulnerable.py
[*] Sending to Gemini 2.5 Flash...

SEVERITY: CRITICAL
TYPE: SQL Injection
DESCRIPTION: User input is concatenated directly into a SQL query string.
IMPACT: An attacker can manipulate the query to extract, modify, or delete arbitrary database records.
FIX:
    cursor.execute("SELECT * FROM users WHERE username = ?", (username,))

SEVERITY: HIGH
TYPE: Hardcoded Credentials
DESCRIPTION: Database password is hardcoded as a string literal in source code.
IMPACT: Anyone with read access to the repository can obtain valid credentials.
FIX:
    password = os.environ.get("DB_PASSWORD")

SEVERITY: MEDIUM
TYPE: Weak Cryptography
DESCRIPTION: MD5 is used for password hashing.
IMPACT: MD5 is cryptographically broken; hashes can be reversed via rainbow tables.
FIX:
    import bcrypt
    hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```

---

## Vulnerability Classes Detected

The scanner identifies (but is not limited to):

- **Injection** — SQL, command, LDAP, XPath
- **Broken Authentication** — hardcoded credentials, weak session tokens
- **Cryptographic Failures** — MD5/SHA1 for passwords, weak key sizes, ECB mode
- **Security Misconfiguration** — debug flags, overly permissive CORS, exposed secrets
- **Insecure Deserialization** — unsafe `pickle`, `eval`, `exec` usage
- **OWASP Top 10** coverage through LLM reasoning rather than rule matching

---

## Project Structure

```
AI_security_scanner/
├── scanner.py          # Core scanner: prompt construction, API call, output rendering
├── vulnerable.py       # Intentionally vulnerable test file (SQL injection, hardcoded creds, weak crypto)
├── requirements.txt    # Pinned dependency versions
├── .env                # API key (gitignored — never committed)
├── .gitignore          # Excludes: .env, venv/, __pycache__/, *.pyc
├── .vscode/
│   └── settings.json   # Workspace-level VS Code configuration
└── venv/               # Python 3.13 virtual environment (gitignored)
```

---

## Security Posture of the Scanner Itself

- **No credentials in source**: API key loaded exclusively from environment via `python-dotenv`
- **`.gitignore` hardened**: `.env`, `venv/`, `__pycache__/`, and `*.pyc` excluded from all commits
- **No persistent storage of scanned code**: Files are read into memory, sent to the API, and discarded — nothing is written to disk or logged
- **Dependency pinning**: All 33 transitive dependencies pinned to exact versions in `requirements.txt` to prevent supply chain drift

---

## Future work

- [ ] Recursive directory scanning (`--dir` flag)
- [ ] Structured report export (JSON / HTML)
- [ ] GitHub Actions integration for CI/CD pipeline security gates
- [ ] Custom rule injection to augment LLM analysis
- [ ] Multi-language support (JavaScript, Go, Java)
- [ ] Severity threshold filtering (`--min-severity HIGH`)

---

## License

MIT
