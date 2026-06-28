<p align="center">
  <img src="docs/images/enumerator-hero-1.jpg" alt="Enumerator overview" width="1000">
</p>

# Enumerator

fscan-driven Active Directory recon orchestrator with a command-line interface and a local web dashboard.

## Overview

`enumerator.sh` is an Active Directory recon aggregator that orchestrates the same tools you would normally run by hand, routes checks by open port, collects loot, and in authenticated mode builds an actionable attack path. The local dashboard is a localhost-only web front-end that launches scans, streams output live, parses results into readable views, and exports a PDF report.  

## Table of Contents

- [Overview](#overview)
- [Components](#components)
- [Installation](#installation)
- [CLI Usage](#cli-usage)
- [Dashboard UI](#dashboard-ui)
- [Output Structure](#output-structure)
- [Pivoting](#pivoting)
- [Reporting](#reporting)
- [Authorized Use](#authorized-use)

## Components

### CLI Orchestrator

The CLI component performs port discovery, per-service enumeration, vulnerability lead generation, loot collection, and summary generation.

### Local Dashboard

The dashboard provides a local web interface for launching scans, watching them live, reviewing results, browsing loot, and exporting a report.

## Installation

Describe here:
- required dependencies;
- supported environment;
- how to make `enumerator.sh` executable;
- how to start the dashboard.

## CLI Usage

### Unauthenticated mode

```bash
./enumerator.sh --unauth 10.10.10.10
```

### Authenticated mode

```bash
./enumerator.sh --auth 10.129.230.181 -u support -p 'Password' -d support.htb
```

Authenticated mode extends the unauthenticated checks with credentialed LDAP enumeration, share loot, BloodHound collection and analysis, ADCS checks, Kerberoast paths, and additional post-enumeration actions.

## Dashboard UI

The local dashboard is designed as a web front-end for `enumerator.sh`.

Main capabilities:
- launch a new scan;
- watch live console output;
- inspect overview, ports, credentials, loot, web findings, BloodHound paths, and commands;
- export a full PDF report.

The dashboard binds to localhost and is intended to be accessed locally or through an SSH tunnel.

## Output Structure

Each run is written to a timestamped folder and includes:
- `SUMMARY.txt`
- `commands.log`
- raw scan output
- per-service output files
- loot, credentials, hashes, downloaded shares, BloodHound JSON, analysis files, and flags

This keeps the workflow auditable and reproducible.

## Pivoting

Enumerator supports pivot-aware operation.

Two common options:
- `chisel` for SOCKS-based pivoting;
- `ligolo-ng` for routed TUN-based pivoting.

For SOCKS workflows, use the dedicated pivot flags instead of launching the entire script under `proxychains`.

## Reporting

The dashboard can export a sectioned PDF report that includes:
- executive summary;
- scope;
- ports and services;
- loot and credentials;
- BloodHound attack paths;
- web findings;
- command history.

## Authorized Use

This tool is intended only for systems you own or for environments where you have explicit authorization to test, such as labs or sanctioned engagements.
