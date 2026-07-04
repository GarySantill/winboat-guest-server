# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WinBoat Guest Server is a Windows-only HTTP API server that runs as a Windows service (`NT AUTHORITY\SYSTEM`) inside a KVM virtual machine guest. It acts as a guest agent for the WinBoat virtualization platform, exposing REST endpoints on port 7148 for the host to query.

## Build

This project targets Windows only — all source files are gated with `//go:build windows`. Cross-compile from Linux/macOS:

```bash
GOOS=windows GOARCH=amd64 go build -o winboat_guest_server.exe \
  -ldflags="-X main.Version=1.0.0 -X main.CommitHash=$(git rev-parse --short HEAD) -X main.BuildTimestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  .
```

```bash
go mod tidy
go mod download
```

There are no tests in this codebase.

## Architecture

### File Responsibilities

- **main.go** — HTTP server (Gorilla mux), all REST endpoint handlers, authentication logic, update mechanism
- **argon2.go** — Argon2id password hash parsing and constant-time verification
- **securekey.go** — Windows registry read/write with custom ACLs (SYSTEM-only write, Everyone read, Administrators denied write)
- **util.go** — `checkErr()` fatal wrapper and `copyFile()` helper

### API Endpoints

| Method | Path | Auth Required | Description |
|--------|------|--------------|-------------|
| GET | `/health` | No | Liveness check |
| GET | `/version` | No | Build metadata |
| GET | `/metrics` | No | CPU/RAM/disk via gopsutil |
| GET | `/apps` | No | Installed apps via PowerShell |
| GET | `/rdp/status` | No | Active RDP session detection (`quser.exe`) |
| POST | `/get-icon` | No | Icon extraction from EXE/LNK path |
| POST | `/auth/set-hash` | No | One-time initial password hash setup |
| POST | `/update` | Yes (password) | ZIP upload → service update |

### Authentication & Security Model

- Single shared password stored as an Argon2id hash (`$argon2id$v=19$m=65536,t=3,p=4$...`) in the Windows registry at `HKLM\SOFTWARE\WinBoatSecureStore`
- Registry ACL is enforced in `securekey.go`: SYSTEM=full control, Everyone=read, Administrators=**explicitly denied** write — so only the service process itself (running as SYSTEM) can modify the hash
- `/auth/set-hash` is a one-time endpoint: it returns 403 if a hash already exists in the registry
- No TLS — security relies on network isolation between host and guest

### PowerShell Integration

Go shells out to PowerShell scripts in `scripts/` for Windows-native operations:
- `scripts/apps.ps1` — Multi-source app discovery (registry, Start Menu, UWP/AppxManifest, Chocolatey, Scoop) + base64-encoded icon extraction
- `scripts/get-icon.ps1` — Extracts and resizes icon from a given EXE/LNK path
- `scripts/update.ps1` — Called after ZIP upload: stops service, backs up, extracts ZIP, restarts service
- `scripts/time-sync.bat` — Scheduled at startup to sync VM clock

All PowerShell invocations use `-ExecutionPolicy Bypass` and `-OutputFormat Text`.

### Update Flow

`POST /update` → password verified → ZIP magic bytes checked (`PK\x03\x04`) → written to temp dir → `update.ps1` launched via `ShellExecuteW` (non-blocking, `runas` verb) → PowerShell waits 2s, stops service, extracts ZIP over install dir, restarts service.

### Installation

`install.bat` (run as Administrator on guest):
1. Creates `C:\Program Files\WinBoat\`
2. Copies files from `C:\OEM\`
3. Registers service via `nssm.exe` as `NT AUTHORITY\SYSTEM`
4. Opens Windows Firewall on port 7148
5. Schedules time-sync task at startup
