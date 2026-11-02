# SecretStore for Buoy · 0.1.0

Reusable protected storage for large UTF-8 secret records. Applications supply a
stable service and account; storage survives application rebuilds and changes to
the executable path. The package uses macOS Keychain, Windows Credential Manager,
or Linux Secret Service through the Keychain package. There is no plaintext or
preferences-file fallback.

Install `secretstore-0.1.0.buipkg`, `keychain-0.1.0.buipkg`, and the toolchain's
`crypto-1.0.0.buipkg` into the consumer's `packages/` directory, then `Import
secretstore`. Build with the native package builder described in
[CONTRIBUTING](https://github.com/BuoyBay/secretstore/blob/main/CONTRIBUTING.md).

## API

- `New SecretRecordStore(service As String, account As String)` implements
  `SecretRecordStorage`.
- `Acquire() As Boolean` obtains a nonblocking native operation lock. `False`
  means another process holds it (or POSIX locking failed); native setup failures
  throw. Hold the lock across load, token renewal, and save; always call `Release()`
  from `Finally`. Acquire is not reentrant. No network operation is performed.
- `Load() As String` returns the committed record or `""` when absent/deleted.
  Missing chunks, invalid metadata, malformed base64, and digest mismatches throw.
- `Save(record As String)` commits a nonempty UTF-8 record of at most 1 MiB.
- `Delete()` writes a tombstone before removing pieces, so interrupted cleanup
  cannot restore a signed-out record. Cleanup errors throw and can be retried.
- `Release()` closes the lock handle. Repeated release is harmless. The OS also
  releases it when the process exits.

All record operations require the lock. They may block or prompt through the
native store; call them on a worker thread. Native denial, locked/unavailable
stores and backend limits propagate as exceptions, distinct from an absent item.
The package never logs or guesses credentials. Serialization and validation of
application-specific fields remain the caller's responsibility.

## Portability and transactions

On macOS, each record is stored as one version 2 envelope at the stable
service/account identity. Its base64 payload and SHA-256 digest are updated in
one native item write. This avoids a separate keychain authorization request for
each chunk of a large session. Version 1 chunked records remain readable and
migrate on the next successful save; retired chunks are removed best effort.
Migration may still prompt for old items once. A failed replacement preserves
the previous record. Deleting a compact record removes its sole item directly.

Windows and Linux encode records into ASCII chunks of at most 1,800 bytes, below
Windows Credential Manager's 2,560-byte item limit. A versioned manifest and
SHA-256 digest commit one complete generation only after every piece is written. Failure before
the manifest commit preserves the previous record; failed uncommitted and retired
piece cleanup is best effort. A crash during save can leave unreachable encrypted
chunks in the native store. Importing the package makes no native-store calls.

The service must be nonempty, at most 128 UTF-16 units, and contain no slash,
backslash, or NUL. Accounts may contain Unicode and separators, are nonempty,
limited to 400 UTF-16 units, and cannot contain NUL. Use stable, app-specific
reverse-DNS service names. Service/account identity is case-insensitive on Windows
and case-sensitive on macOS/Linux, matching the underlying Keychain package.

Secret-free per-user lock files use SHA-256 of the framed service/account identity,
so different apps cannot collide and arbitrary accounts cannot inject paths.
POSIX uses `flock`; Windows uses `CreateFileW` and `LockFileEx`. Windows lock
identity is lowercased to match ordinary case-insensitive native targets; callers
should keep service/account spelling canonical, especially for Unicode identities.
Locks do not coordinate other applications that access the same native items
without using this package. They do not extend to another OS user or machine.

The native store controls access across executable builds: macOS may prompt to
allow a newly signed build, Linux needs a running unlocked Secret Service, and
Windows uses the signed-in user's Credential Manager. This package does not
bypass native prompts or grant all executables access to stored secrets.

## Injectable storage

`SecretRecordItems` is an injectable Read/Write/Remove transport. The injectable
constructor `SecretRecordStore(service, account, items, singleItem = False)`
supports offline tests of either storage format; set `singleItem` to True to
exercise the macOS format on any host. The injected transport supports alternative storage backends while native locking remains active.

The injectable backend permits exercising either storage policy on a supported
host. Native macOS prompt behavior requires validation in the actual application;
compiler checks do not establish prompt or keychain behavior.
