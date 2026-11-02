# Native secret records

Reusable protected storage for large UTF-8 secret records. Applications supply a
stable service and account; storage survives application rebuilds and changes to
the executable path. The package uses macOS Keychain, Windows Credential Manager,
or Linux Secret Service through the Keychain package. There is no plaintext or
preferences-file fallback.

## Install

From a Buoy consumer project with BPM installed:

```sh
bpm add secretstore --git https://github.com/BuoyBay/secretstore.git --rev refs/heads/main
```

BPM records the resolved commit in its lockfile. Keep that lockfile with the
consumer project for reproducible installs.
BPM also resolves `keychain`.

## API at a glance

- `New SecretRecordStore(service As String, account As String)` implements
  `SecretRecordStorage`.
- `Acquire() As Boolean` obtains a nonblocking native operation lock. `False`
  means another process holds it (or POSIX locking failed); native setup failures
  throw. Hold the lock across load, token renewal, and save; always call `Release()`
  from `Finally`. Acquire is not reentrant. No network operation is performed.
- `Load() As String` returns the committed record or `""` when absent/deleted.
  Missing chunks, invalid metadata, malformed base64, and digest mismatches throw.
- `Save(record As String)` commits a nonempty UTF-8 record of at most 1 MiB.

## Platform and scope

Uses Keychain on macOS, Credential Manager on Windows and Secret Service on Linux. Stores may prompt, be locked or be unavailable; no plaintext fallback is used.

## Build from source

With the native `build-packages` driver from [devtools](https://github.com/BuoyBay/devtools)
on your `PATH`, run from this checkout:

```sh
build-packages --root "$PWD" --compile-consumer secretstore
```

The driver checks source targets and compiles an import consumer. See the
[API guide](docs/API.md) for behavior, limits and any host prerequisites.

## Contributing

[Contributing](CONTRIBUTING.md) · [License](LICENSE)
