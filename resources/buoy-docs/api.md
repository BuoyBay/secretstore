# SecretStore API

`SecretRecordStorage` exposes Acquire, Release, Load, Save and Delete.
`SecretRecordStore(service As String, account As String)` implements it using the
platform credential store. Call Acquire before any record operation; False means
it was not acquired. Hold the lock through an entire read/renew/write operation
and release in Finally on the same worker thread.

Load returns an empty string only for absent or tombstoned records. Save accepts
nonempty UTF-8 strings up to 1 MiB. Failed chunk or commit writes preserve the
previous committed record. Delete commits a tombstone before chunk cleanup;
errors remain visible and deletion can be retried. Native errors never become
plaintext fallback files.

For offline tests, implement `SecretRecordItems` with Read(account) returning
Optional Of String, Write(account, value), and Remove(account), and pass it as a
third constructor argument. An optional fourth Boolean `singleItem` (default
False) exercises the macOS single-item policy on any host. This changes item
transport, not native locking.

The production constructor uses a single version 2 protected item on macOS,
reducing per-item keychain prompts. Existing version 1 chunked records migrate
on their next successful save. Windows and Linux continue using bounded chunks.
Both formats validate their payload digest; failed writes preserve the previous
record. Compact deletion removes the one item directly.
