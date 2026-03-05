# aplsaft

A minimal SFTP client library for Dyalog APL, built on libssh2 via `⎕NA`.

No Conga. No external APL dependencies. One file.

## Requirements

- Dyalog APL 20.0+
- libssh2 (macOS: `brew install libssh2`; Linux: system package)
- macOS or Linux (64-bit)

## Quick start

```apl
]link.import # /path/to/SFTP.apln

⍝ Connect
conn ← SFTP.ConnectWithPass 'host' 22 'user' 'password'
conn ← SFTP.ConnectWithKey  'host' 22 'user' '/path/to/key' ''

⍝ Browse
listing ← SFTP.List conn '/remote/dir'    ⍝ → vector of (name attrs) pairs
attrs   ← SFTP.Stat conn '/remote/file'   ⍝ → namespace: filesize, permissions, …

⍝ Transfer
data ← SFTP.Get conn '/remote/file'                    ⍝ → byte vector
SFTP.Download conn '/remote/file' '/local/file'         ⍝ write to disk
SFTP.Put conn 'hello world' '/remote/file'              ⍝ chars → UTF-8
SFTP.Put conn (⎕UCS 'hello') '/remote/file'             ⍝ raw bytes
SFTP.Upload conn '/local/file' '/remote/file'           ⍝ read from disk

⍝ Manage
SFTP.Mkdir  conn '/remote/dir'
SFTP.Rmdir  conn '/remote/dir'
SFTP.Delete conn '/remote/file'
SFTP.Rename conn '/old/path' '/new/path'

⍝ Disconnect
SFTP.Disconnect conn
```

## API reference

### Connection

| Function | Arguments | Returns |
|---|---|---|
| `ConnectWithPass` | `host port user password` | `conn` |
| `ConnectWithKey` | `host port user keyfile passphrase` | `conn` |
| `Disconnect` | `conn` | — |

`conn` is an opaque namespace. Pass it as the first argument to every operation.

### File operations

| Function | Arguments | Returns |
|---|---|---|
| `Get` | `conn remotepath` | byte vector |
| `Download` | `conn remotepath localpath` | — |
| `Put` | `conn data remotepath` | — |
| `Upload` | `conn localpath remotepath` | — |

`Put` accepts character vectors (auto-encoded as UTF-8) or integer byte vectors (0–255, written as-is).

### Directory operations

| Function | Arguments | Returns |
|---|---|---|
| `List` | `conn path` | vector of `(name attrs)` pairs |
| `Stat` | `conn path` | namespace with `filesize uid gid permissions atime mtime` |
| `Mkdir` | `conn path` | — |
| `Rmdir` | `conn path` | — |
| `Delete` | `conn path` | — |
| `Rename` | `conn oldpath newpath` | — |

`List` entries include `.` and `..`. Filter them with:

```apl
names ← ⊃¨ SFTP.List conn '/dir'
names ← (~names∊(,'.') '..')/names
```

### Errors

All errors signal `810` with a descriptive message. Trap with `:Trap 810` or `:Trap 0`.

## Files

| File | Purpose |
|---|---|
| `SFTP.apln` | The library — single namespace, all code inline |
| `Example.aplf` | Literate walkthrough demonstrating every operation |
| `GenTest.apln` | Model-based generative tester: `GenTest.Run 100 42` |

## Architecture

The library binds ~20 libssh2 functions and 6 libc functions via `⎕NA`:

- **libc**: `getaddrinfo`, `freeaddrinfo`, `socket`, `connect`, `close`, `memcpy`
- **libssh2**: session management, authentication, and SFTP operations

Connection flow: DNS resolve → TCP connect → SSH handshake → authenticate → SFTP init.

All bindings are loaded lazily on first use. `Init` auto-detects the OS via `uname -s` and selects the correct library paths and `addrinfo` struct offsets for macOS (BSD) or Linux (POSIX).
