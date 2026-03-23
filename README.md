# pylast -- Linux Authentication Artifact Parser

A zero-dependency, single-file Python tool for parsing Linux authentication artifacts in digital forensics investigations.

Modern Linux distributions (Debian 13+, Ubuntu 25.04+) have replaced the legacy `last` binary with `wtmpdb`, a SQLite-based login tracking system. Forensic examiners working with disk images from both legacy and modern systems no longer have a single tool that handles all login record formats. **pylast** fills that gap.

## Features

- **Multi-format support** - parses legacy wtmp/btmp/utmp binary files (including `.gz` compressed from logrotate), the newer wtmpdb (SQLite) format, auth.log files (including `.gz` compressed), and systemd journal directories
- **Auto-detection** - automatically identifies whether a file is wtmp, btmp, utmp, or wtmpdb based on file structure, not just filename
- **Cross-architecture** - correctly parses the 384-byte `struct utmp` from x86_64, x86_32, and ARM systems
- **Three output modes** - colorized terminal output, tab-separated values (TSV), or direct SQLite database export
- **SQLite case database** - the `-s` flag writes parsed records into a SQLite database with full audit logging, SHA-256 integrity hashing, and support for overwrite/append workflows
- **Forensically sound** - all timestamps displayed in UTC with microsecond precision, source file tracking per record, and pre/post-parse SHA-256 hashes logged for chain-of-custody documentation
- **Zero dependencies** - runs on Python 3.11+ using only the standard library; no `pip install`, no `venv`, just `chmod +x` and go
- **TTY-aware** - colorized output when interactive, clean plaintext when piped or redirected

## Supported Artifact Types

| Flag | Source | Description |
|------|--------|-------------|
| `-f` | wtmp / btmp / utmp | Legacy 384-byte binary login records (including `.gz`) |
| `-f` | wtmp.db | SQLite-based wtmpdb (Debian 13+, Ubuntu 25.04+) |
| `-a` | auth.log | Syslog authentication logs, including `.gz` rotated files |
| `-j` | journal directory | Systemd binary journal files (parsed via `journalctl`) |

## Authentication Events Detected

From auth.log (`-a`) and journal (`-j`) sources:

- SSH successful logins
- SSH failed logins
- PAM authentication failures
- `sudo` command execution
- `sudo` denial (user not in sudoers)
- `sudo` authentication failures
- `su` session opens
- `su` failures

## Installation

```bash
# Clone the repository
git clone https://github.com/ForensicFoundry/pylast.git
cd pylast

# Make it executable
chmod +x pylast

# Optionally, place it in your PATH
sudo cp pylast /usr/local/bin/
```

No virtual environment or package installation required. Runs on any system with Python 3.11 or later.

## Quick Start

```bash
# Parse a legacy wtmp file
pylast -f /evidence/wtmp

# Parse btmp (failed logins) from multiple files
pylast -f /evidence/btmp.0 /evidence/btmp.1

# Parse a wtmpdb
pylast -f /evidence/wtmp.db

# Parse auth.log files (including gzip-compressed rotated logs)
pylast -a /evidence/auth.log*

# Parse a systemd journal directory
pylast -j /evidence/journal/

# Export to TSV for spreadsheet analysis
pylast -f /evidence/wtmp -t > timeline.tsv

# Export to a SQLite database
pylast -a /evidence/auth.log* -s endpoint001.db

# Show all record types (including internal types normally filtered)
pylast -f /evidence/wtmp --all
```

## Output Modes

### Terminal (default)

Colorized, column-aligned output sorted newest-first. Colors are automatically suppressed when output is piped or redirected.

### TSV (`-t`)

Tab-separated values with a header row. Suitable for import into Excel, LibreOffice Calc, or other analysis tools.

### SQLite (`-s`)

Writes records directly into a SQLite database file. Each source type maps to a dedicated table:

| Source Type | Table Name |
|-------------|------------|
| Legacy wtmp | `wtmp_log` |
| Legacy btmp | `btmp_log` |
| wtmpdb | `wtmpdb_log` |
| Auth.log | `auth_log` |
| Journal | `journal_log` |

A `pylast_log` table tracks every import operation with:
- UTC timestamp of the import
- Source type and file path (one row per source file)
- Action performed (`new`, `refresh`, `append`, or `aborted`)
- Number of records written
- Pre-parse and post-parse SHA-256 hashes of each source file

If the target table already exists, pylast prompts to overwrite, append, or abort.

#### Verifying SHA-256 Hashes

For individual files:

```bash
sha256sum /evidence/auth.log.1
```

For journal directories (combined digest over all `.journal` files sorted by path):

```bash
find /evidence/journal/ -name '*.journal' -print0 | sort -z | xargs -0 cat | sha256sum
```

## Column Schemas

All column headers are space-free for SQL-friendly querying.

**wtmp** (14 columns): Type, PID, User, TTY, TID, Host, IP, Timestamp, Logout, Duration, ExitStatus, ExitCode, SessionID, Source

**btmp** (12 columns): Type, PID, User, TTY, TID, Host, IP, Timestamp, ExitStatus, ExitCode, SessionID, Source

**wtmpdb** (9 columns): Type, User, Login, Logout, Duration, TTY, RemoteHost, Service, Source

**auth.log / journal** (12 columns): Timestamp, Hostname, Event, User, TargetUser, RemoteIP, Port, TTY, Command, PID, Service, Source

## Requirements

- Python 3.11 or later
- `journalctl` (only required for the `-j` flag; all other features work without it)
- Linux, macOS, or any POSIX system with Python (the `-j` flag requires a systemd-based system)

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Error (file not found, invalid format, etc.) |
| 2 | Usage error (invalid arguments) |

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

You are free to use, modify, and distribute this software under the terms of the GPL v3. Any derivative work must also be distributed under the same license.

## Author

Forensic Foundry
