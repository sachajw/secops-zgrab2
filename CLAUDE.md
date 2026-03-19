# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

zgrab2 is a fast, modular L7 network scanner for large-scale Internet surveys. It performs application-layer handshakes and outputs detailed JSON transcripts. Designed to work downstream of ZMap (L4 scanning).

## Commands

```bash
# Build
make                    # Builds ./zgrab2 binary

# Test
make test               # Unit tests across root, lib/output, and modules/
make lint               # gofmt, goimports, golangci-lint, black

# Integration tests (require Docker + Python 3)
make integration-test-clean && make integration-test
TEST_MODULES="http ssh" make integration-test   # Specific modules only

# Run a single Go test
go test ./modules/http/... -run TestHTTP -v
go test ./... -run TestFoo -v

# Use the binary
echo "1.1.1.1" | ./zgrab2 http --port 80
echo "1.1.1.1" | ./zgrab2 tls --port 443
```

## Architecture

### Module System

Each protocol lives in `modules/<protocol>/scanner.go` and must implement three interfaces defined in `module.go`:

- **`ScanModule`** — factory; implements `NewFlags()`, `NewScanner()`, `Description()`
- **`Scanner`** — core scanning logic; implements `Scan()`, `Init()`, `GetDialerGroupConfig()`, etc.
- **`ScanFlags`** — command-line flags; must embed `zgrab2.BaseFlags`

### Scanner Pipeline

1. Input: CSV from stdin or file (`IP, DOMAIN, TAG, PORT`)
2. `bin/bin.go` parses targets, spawns N sender goroutines
3. Each sender calls `RunScanner()` per target → `scanner.Scan(ctx, dialerGroup, target)`
4. Results aggregated into `Grab` struct (maps module name → `ScanResponse`)
5. Output: newline-delimited JSON to stdout or file

### Dialer System

Modules declare network needs via `DialerGroupConfig` returned from `GetDialerGroupConfig()`:
- `TransportAgnosticDialerProtocol` — TCP or UDP
- `NeedSeparateL4Dialer` — set true for protocols needing STARTTLS or raw socket control
- `TLSEnabled` / `TLSFlags` — for TLS wrapping

Most modules use `dialerGroup.Dial(ctx, target)` to get a connection; STARTTLS modules use the separate L4 + TLS dialers.

### Key Files

| File | Role |
|------|------|
| `module.go` | Core interfaces: `Scanner`, `ScanModule`, `ScanFlags`, `DialerGroupConfig` |
| `scanner.go` | `RegisterScan()`, `RunScanner()` — registration and execution |
| `bin/bin.go` | Entry point logic — CLI parsing, worker pool, I/O |
| `bin/default_modules.go` | Imports all registered modules |
| `processing.go` | `Grab`, `ScanTarget`, `ScanResponse` structs; dialer construction |
| `tls.go` | Shared `TLSFlags` and `TLSLog` types |
| `conn.go` | Connection wrapper with byte/time limits |
| `cli.go` | Flag parsing via zflags; INI multi-module config |
| `config.go` | `Config` struct (General, I/O, Networking options) |

### Adding a New Module

1. Create `modules/<proto>/scanner.go` implementing `Module`, `Scanner`, `Flags` (embed `zgrab2.BaseFlags`)
2. Register in module's `RegisterModule()` via `zgrab2.AddCommand("proto", ..., defaultPort, &module)`
3. Create `modules/<proto>.go` wrapper that calls `RegisterModule()` in `init()`
4. Import the wrapper in `bin/default_modules.go`
5. Optionally add integration test: Docker service in `integration_tests/docker-compose.yml`, shell test script, and JSON schema in `zgrab2_schemas/`

### Two Execution Modes

**Single module (CLI):**
```bash
echo "10.0.0.1" | ./zgrab2 http --port 8080
```

**Multi-module (INI config):**
```bash
./zgrab2 multiple --config scan.ini
```
INI sections map to module names; `name` key disambiguates multiple runs of the same protocol; `trigger` key filters by target tag.

### Output Format

```json
{
  "ip": "1.1.1.1",
  "data": {
    "http": {
      "status": "success",
      "protocol": "http",
      "port": 80,
      "result": {},
      "timestamp": "..."
    }
  }
}
```

`ScanStatus` values: `success`, `connection-refused`, `connection-timeout`, `io-timeout`, `protocol-error`, `application-error`, `unknown-error`.
