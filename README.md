# aplsaft

Two implementations for SFTP in Dyalog APL, by two implementers.

## SFTP.apln

_cursork's contribution_

It's only one evening and one morning's work. It's tested, and I am happy to
use it. I am _extremely nervous_ about others relying on it. See the
[testing](#Testing) notes.

A minimal SFTP client library for Dyalog APL, built on libssh2 via `⎕NA`. No
Conga. No external APL dependencies. One file.

Everything in the repo is related to SFTP.apln, except SSHClient.aplc as
discussed below.

## SSHClient.aplc

_[Bombardier-C-Kram](https://github.com/Bombardier-C-Kram)'s contribution_ (thanks!) 

A completely separate and more 'pure' Dyalog implementation, in the sense that the
only dependency is Conga, and that much of the code is APL implementing the
protocol.

I (_cursork_) suggest that if you are familiar with APL, this version may be
interesting in the case you want to know some of the details.

## SFTP Namespace Usage

### Requirements

- Dyalog APL 20.0+
- libssh2 (macOS: `brew install libssh2`; Linux: system package)
- macOS or Linux (64-bit)
  - should work under WSL, and uses standard portable libraries, but not yet
    tested in Windows

#### Quick start

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

### API reference

#### Connection

| Function | Arguments | Returns |
|---|---|---|
| `ConnectWithPass` | `host port user password` | `conn` |
| `ConnectWithKey` | `host port user keyfile passphrase` | `conn` |
| `Disconnect` | `conn` | — |

`conn` is an opaque namespace. Pass it as the first argument to every operation.

#### File operations

| Function | Arguments | Returns |
|---|---|---|
| `Get` | `conn remotepath` | byte vector |
| `Download` | `conn remotepath localpath` | — |
| `Put` | `conn data remotepath` | — |
| `Upload` | `conn localpath remotepath` | — |

`Put` accepts character vectors (auto-encoded as UTF-8) or integer byte vectors (0–255, written as-is).

#### Directory operations

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

#### Errors

All errors signal `810` with a descriptive message. Trap with `:Trap 810` or `:Trap 0`.

### Testing

`GenTest.apln` is a model-based generative tester. It has an in-memory model of
the remote filesystem, applies operations randomly (mkdir, put, get, list, stat,
rename, delete, rmdir, download, upload), and checks the server against the
model after each one. A final verification pass re-reads every file and walks
every directory to confirm nothing drifted.

Negative cases are also part of it, eg get/delete/stat on nonexistent paths,
mkdir on existing directories, rmdir on non-empty directories are  all expected to
signal.

```apl
]link.import # /path/to/SFTP.apln
]link.import # /path/to/GenTest.apln

GenTest.Run 100 42 'host' 22 'user' '/path/to/key'     ⍝ 100 iterations, seed 42
GenTest.Run ¯60 7 'host' 22 'user' '/path/to/key'      ⍝ run for 60 seconds, seed 7
```

Positive `n` sets an iteration count; negative `n` sets a time budget in seconds. The seed makes runs reproducible. Connection parameters match `Example`. Output looks like:

```
Connected. Base: /tmp/aplsftp-fuzz-42
  mkdir /KBZQW
  put /KBZQW/AMXRL.dat (312 bytes)
  get /KBZQW/AMXRL.dat OK
  list / OK (1 entries)
  negative: get nonexistent OK
  rename /KBZQW/AMXRL.dat → /HTDNÉ/VPQJF.dat
  ...

=== Verification ===
  OK: /HTDNÉ/VPQJF.dat
  DIR OK: / (2 entries)
  DIR OK: /KBZQW (0 entries)
All 1 files and 3 dirs verified.
Cleaned up.

=== Summary ===
Iterations: 100 in 8.4s
mkdir:     12
put:       15
get:       9
...
Failures: 0
Seed: 42
```

### Architecture

The library binds ~20 libssh2 functions and 6 libc functions via `⎕NA`:

- **libc**: `getaddrinfo`, `freeaddrinfo`, `socket`, `connect`, `close`, `memcpy`
- **libssh2**: session management, authentication, and SFTP operations

Connection flow: DNS resolve → TCP connect → SSH handshake → authenticate → SFTP init.

All bindings are loaded lazily on first use. `Init` auto-detects the OS via `uname -s` and selects the correct library paths and `addrinfo` struct offsets for macOS (BSD) or Linux (POSIX).
