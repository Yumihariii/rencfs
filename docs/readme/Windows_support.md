# Windows Support — Implementation Research Note

> **Status:** Research only. This note documents how rencfs *could* mount on Windows
> natively; it does **not** claim that rencfs mounts on Windows today. The Linux
> backend (`src/mount/linux.rs`, via `fuse3`) is the only real mount implementation.
> On every non-Linux target rencfs currently falls back to the `dummy` backend
> (`src/mount/dummy.rs`), which returns `Err(FsError::Other("Dummy implementation"))`
> from `mount()`.

This note is intentionally written as a first milestone *proposal*: the goal is a
**truthful, minimal, verifiable** Windows implementation that can be reviewed and
merged, not a premature claim of full parity with the Linux backend.

---

## 1. Goal and scope of the first milestone

The Linux backend implements the `MountPoint` trait (defined in `src/mount.rs`):

```rust
pub trait MountPoint {
    fn new(
        mountpoint: PathBuf,
        data_dir: PathBuf,
        password_provider: Arc<dyn PasswordProvider>,
        cipher: Cipher,
        allow_root: bool,
        allow_other: bool,
        read_only: bool,
    ) -> Self where Self: Sized;
    async fn mount(mut self) -> FsResult<MountHandle>;
}
```

A Windows backend must implement this same trait behind a `#[cfg(target_os = "windows")]`
gate, mirroring how `linux.rs` is gated with `#[cfg(target_os = "linux")]` in
`src/mount.rs`. **No change to the encryption layer is required** — `EncryptedFs`
(`src/encryptedfs.rs`) is entirely backend-agnostic and exposes all the operations a
mount driver needs. The Windows work is confined to a new `src/mount/windows.rs`.

**First-milestone scope (must be truthful):**

- Mount a real directory on Windows via WinFsp, backed by `EncryptedFs`.
- Read + write files, list directories, create/remove files and directories.
- Correctly handle the *common* metadata (size, mtime, read-only flag).
- Not claim support for advanced/uncommon NTFS features (see §7).

---

## 2. Backend options (recommended first, alternatives considered)

### 2.1 Recommended: `winfsp-rs` (`winfsp` crate)

The most widely used safe Rust binding to WinFsp. Provides a `Filesystem` trait whose
methods mirror the WinFsp C++ `FileSystemContext` callbacks (get file info, open, read,
write, read directory, etc.), plus an event-loop `FileSystemHost`.

- **Pros:** Mature, documented, actively used; maps 1:1 onto WinFsp's model; async
  bridging is handled by the crate's own runtime integration; many examples (memfs,
  passthrough) to crib from.
- **Cons:** The `winfsp` crate is **GPL-3.0** licensed (see §3 licensing). It pulls in a
  large dependency tree.

### 2.2 Alternative: `unifuse` (MIT, cross-platform)

A cross-platform async FUSE abstraction with a `UniFuseFilesystem` trait and two
adapters: `Rfuse3Adapter` (Linux/macOS) and `WinfspAdapter` (Windows). It is MIT
licensed, `#![forbid(unsafe_code)]`, and path-based (no inode map needed on Windows,
since WinFsp is natively path-based).

- **Pros:** Cleaner licensing (MIT); one trait for both platforms (potential future
  unification of the mount layer); zero unsafe.
- **Cons:** Requires **nightly** Rust (edition 2024, RPITIT); it is a *new* abstraction,
  so adopting it means reworking the existing `MountPoint`/`MountHandle` surface rather
  than just adding a Windows module. Its trait is path-based, which does not match the
  inode-based `EncryptedFs` API (needs a path→inode translation layer, which the Linux
  backend does inside `linux.rs`).

### 2.3 Not recommended for the first milestone

- **`winfsp_wrs`** — safe-ish wrapper but requires `rust-bindgen` + Clang at build time
  to parse WinFsp C++ headers; heavier build, smaller ecosystem.
- **Direct WinFsp C API via `windows`/`windows-sys`** — maximum control but the most
  code to hand-write; not worth it until the binding crates prove inadequate.

**Decision:** implement the first milestone with **`winfsp-rs`**, because it is the
closest match to the existing inode-based design and has the richest examples. Keep
`unifuse` as the fallback if the GPL-3 license boundary becomes a blocker (§3).

---

## 3. Licensing / attribution checkpoint (MIT OR Apache-2.0 project)

This is the single most important design constraint and must be settled *before* writing
code, because it determines which crate we may link.

| Component | License | Impact on rencfs (MIT OR Apache-2.0) |
|---|---|---|
| **WinFsp** (C++ runtime) | **GPLv3 + FLOSS special exception** | ✅ Usable. The FLOSS exception explicitly permits use by free/open-source software. rencfs is FLOSS, so no GPL contagion from WinFsp itself. |
| **`winfsp` (winfsp-rs) crate** | **GPL-3.0** | ⚠️ GPL is contagious. Linking it makes the *Windows build* GPL-3. For an `MIT OR Apache-2.0` project this is a license-boundary decision the maintainers must accept or avoid. |
| **`unifuse` crate** | **MIT** | ✅ No contagion. |

Two viable paths:

1. **Accept GPL-3 for the Windows module** (recommended if we use `winfsp-rs`).
   This is defensible because rencfs is *dual-licensed* `MIT OR Apache-2.0`; the Windows
   mount module can be distributed as GPL-3 without affecting the rest of the crate,
   provided the crate's overall license statement is clarified (e.g. note in README that
   the `windows` feature pulls in GPL-3 code). This mirrors how other projects (e.g.
   `winfsp-fs-skeleton`) deliberately keep the GPL boundary at the consumer's link line.
2. **Avoid GPL entirely** by using `unifuse` (MIT), at the cost of adopting nightly Rust
   and a new abstraction.

**Attribution requirement:** WinFsp's FLOSS exception requires that any user-facing
interface/documentation include the copyright notice
`"WinFsp - Windows File System Proxy, Copyright (C) Bill Zissimopoulos"` and a link to
the WinFsp repository. This must be added to rencfs's docs/README and (if applicable) a
NOTICE file. The `winfsp-rs` crate also carries this attribution requirement.

**Recommendation:** Proceed with `winfsp-rs` (GPL-3) for the first milestone and clearly
document the license boundary. If the maintainers reject GPL-3, switch the Windows
backend to `unifuse` (MIT) — the `MountPoint` abstraction isolates the swap.

---

## 4. Proposed architecture — `src/mount/windows.rs`

Mirror the structure of `src/mount/linux.rs`. The Windows module implements the same
`MountPoint` trait and exposes a `MountPointImpl` / `MountHandleInnerImpl` pair.

```
src/mount.rs
 ├─ #[cfg(target_os = "linux")] mod linux;          (existing)
 ├─ #[cfg(target_os = "windows")] mod windows;      (NEW)
 └─ #[cfg(not(any(target_os = "linux", target_os = "windows")))] mod dummy;  (existing)
```

`src/mount/windows.rs` outline:

- `MountPointImpl` holds the same fields as the Linux impl: `mountpoint`, `data_dir`,
  `password_provider`, `cipher`, `allow_root`, `allow_other`, `read_only`.
- `mount()` builds an `Arc<EncryptedFs>` (via `EncryptedFs::new(...)`, same as
  `linux.rs`), then constructs a `winfsp::FileSystemContext`-backed filesystem and starts
  the `FileSystemHost` event loop. It returns a `MountHandle` whose inner future resolves
  when the host exits (mirroring how `MountHandleInnerImpl` in `linux.rs` is a `Future`).
- Because WinFsp callbacks are synchronous and `EncryptedFs` methods are async, the
  Windows module must bridge sync→async, e.g. via `tokio::runtime::Handle::block_on(...)`
  (the same approach `unifuse`'s `WinfspAdapter` uses). This is the main technical
  complexity and must be carefully tested.

### 4.1 Gate changes in `src/mount.rs`

The current gate is:

```rust
#[cfg(target_os = "linux")]
mod linux;
#[cfg(not(target_os = "linux"))]
mod dummy;
```

Change to:

```rust
#[cfg(target_os = "linux")]
mod linux;
#[cfg(target_os = "windows")]
mod windows;
#[cfg(not(any(target_os = "linux", target_os = "windows")))]
mod dummy;
```

and select the impl types accordingly. `MountHandle`/`MountHandleInner` need no change.

---

## 5. Operation mapping: WinFsp callbacks → `EncryptedFs` methods

The `EncryptedFs` API is inode-based; the mapping below is the contract the Windows
backend must honor. This is the most error-prone part and must be validated by tests.

| WinFsp callback | `EncryptedFs` method | Notes |
|---|---|---|
| `get_file_info` | `get_attr(ino)` | Return `FileAttr` → WinFsp `FileInfo`. |
| `open` / `create` | `open(ino, read, write)` / `create(...)` | Returns a handle (`u64`). |
| `read` | `read(ino, offset, buf, handle)` | Honor the read handle; return bytes read. |
| `write` | `write(ino, offset, buf, handle)` | Return bytes written. |
| `read_directory` | `read_dir(ino)` → `DirectoryEntryIterator` | Iterate entries; translate names to `FileInfo`. |
| `get_security_by_name` / `get_security` | `get_attr` (map uid/gid/permissions) | Windows uses SIDs; see §7. |
| `rename` | `rename(...)` | Map `REPLACE_EXISTING`/`EXCLUSIVE` flags. |
| `set_file_info` | `set_attr(ino, SetFileAttr)` | Set size/timestamps; read-only via attribute. |
| `cleanup` / `flush` / `close` | `flush(handle)` / `release(handle)` | Ensure data is flushed. |
| `read_directory` plus `get_file_info` | `read_dir_plus(ino)` | If needed for efficiency. |
| `statfs` | — (compute from `EncryptedFs`) | WinFsp has no direct statfs; provide sensible defaults. |

The full set of `EncryptedFs` methods available for the mapping (from
`src/encryptedfs.rs`): `create`, `find_by_name`, `remove_dir`, `remove_file`,
`exists_by_name`, `read_dir`, `read_dir_plus`, `get_attr`, `set_attr`, `read`, `write`,
`flush`, `release`, `open`, `set_len`, `rename`, `copy_file_range`, plus
`create_write`/`create_read` helpers.

**Key mapping decisions:**

- **Names are encrypted** (the encrypted filesystem stores ciphertext filenames). The
  `EncryptedFs` layer handles name encryption/decryption internally; the mount layer must
  pass *plaintext* names to/from the OS and let `EncryptedFs` translate. Do **not**
  re-encrypt in the Windows module.
- **Inodes:** `EncryptedFs` uses `u64` inodes internally. WinFsp is path-based, so the
  Windows module needs a path↔inode translation layer (like the one inside `linux.rs`).
  Maintain a `HashMap<PathBuf, u64>` or resolve via `find_by_name` walking from the root.

---

## 6. Windows-only dependency and build shape

- Add a Windows-only dependency in `Cargo.toml`:

  ```toml
  [target.'cfg(target_os = "windows")'.dependencies]
  winfsp = { version = "...", features = [...] }   # or unifuse if license chosen
  ```

  This mirrors the existing Linux-only dep:

  ```toml
  [target.'cfg(target_os = "linux")'.dependencies]
  fuse3 = { version = "0.8.1", features = ["tokio-runtime", "unprivileged"] }
  ```

- **Delay-load / runtime dependency:** WinFsp must be installed on the host
  (`choco install winfsp`). The Rust crate links against the WinFsp DLL at runtime; the
  build itself does not require WinFsp headers when using `winfsp-rs` (unlike
  `winfsp_wrs`, which needs Clang + bindgen). This keeps CI on Linux/macOS unaffected —
  the `windows` module only compiles on `cfg(target_os = "windows")`.
- **MSRV / toolchain:** rencfs's `rust-toolchain.toml` must be checked. If `unifuse` is
  chosen, nightly is required; with `winfsp-rs`, stable should suffice.
- **Feature gate (optional):** Consider a cargo feature `windows-support` so the GPL-3
  dependency is opt-in, keeping the default `MIT OR Apache-2.0` build free of GPL. This
  is the cleanest way to satisfy the license checkpoint.

---

## 7. Windows metadata decisions and unsupported NTFS features (first milestone)

**What we can truthfully support in milestone 1:**

- Regular file read/write and directory listing/creation/removal.
- Basic metadata: size, last-write time, read-only attribute.
- A single security identity (map everything to the current user's SID; ignore
  fine-grained ACLs initially).

**What we explicitly do NOT support in milestone 1 (document, don't claim):**

- **NTFS alternate data streams** (ADS, `name:stream`). Map to a "not supported" /
  single default stream.
- **Hard links** (`GetFileInformation` link count). Report `nlink = 1`; hard-link
  semantics on encrypted filenames are non-trivial.
- **Symlinks / reparse points** — only if the Linux backend's symlink handling is
  replicated; otherwise report as unsupported.
- **POSIX permissions ↔ Windows ACLs** — the Linux backend uses mode bits
  (`allow_root`/`allow_other`). On Windows, map these to a coarse ACL (e.g. owner full
  control, others read-only) rather than attempting full POSIX→NTFS ACL translation.
- **Case sensitivity** — decide and document whether encrypted names are case-sensitive
  (they should be, matching the Linux backend); WinFsp defaults to case-insensitive
  unless configured. This must be a deliberate, documented choice.
- **`statfs`** — provide plausible defaults; no real block accounting.
- **Timestamps** — support mtime/atime best-effort; NTFS `FileTime` precision is
  higher than the Unix `SystemTime` the Linux backend uses; truncate gracefully.

Every "not supported" item above must be handled by returning a graceful error or a
sensible default — **never a panic or silent wrong data** — so the first PR is honest.

---

## 8. Build / CI plan

1. **Local dev on Windows:** install WinFsp (`choco install winfsp`), then
   `cargo run --release -- <mountpoint> <data_dir>`.
2. **CI:** keep the existing Linux/macOS jobs untouched (the `windows` module is
   `cfg`-gated and won't compile on non-Windows). Add a **Windows runner** job
   (`windows-latest`) that:
   - installs WinFsp (`choco install winfsp`),
   - runs `cargo test --target x86_64-pc-windows-msvc`,
   - runs the smoke test (below) if a mount can be performed in CI (may need a
     Windows-specific integration test that skips gracefully when no mount is possible).
3. **Feature-gated build:** if `windows-support` is an opt-in feature, ensure the default
   build stays `cargo check`-clean on Linux/macOS without the WinFsp dependency.
4. **Clippy / fmt:** follow the project's existing `clippy.toml` and `rustfmt`
   conventions; run `cargo clippy --all-targets` and `cargo fmt --check`.

---

## 9. Manual smoke test (Windows host with WinFsp installed)

A reviewer must be able to reproduce this exactly:

1. Install WinFsp: `choco install winfsp`.
2. Build: `cargo build --release`.
3. Create an empty data dir and a mountpoint dir, e.g.:
   ```
   rencfs.exe C:\mnt C:\data
   ```
   (adjust to the actual CLI argument order used by `src/main.rs`).
4. In another shell, verify the mount is live:
   - `dir C:\mnt` → empty.
   - `echo hello > C:\mnt\hello.txt` → then `dir C:\data` shows a ciphertext entry.
   - `type C:\mnt\hello.txt` → prints `hello` (proves read/write round-trip).
   - `mkdir C:\mnt\sub` and create a file inside it, then re-list.
   - Delete the file/dir, confirm the ciphertext disappears from `C:\data`.
5. Unmount (Ctrl-C or `umount` equivalent) and confirm `C:\mnt` is released.
6. **Negative test:** confirm that unsupported features (e.g. an ADS) do not crash the
   driver — they return a graceful error.

If any step fails, the PR must say so explicitly rather than claiming success.

---

## 10. Acceptance criteria for a truthful first implementation PR

A PR that legitimately earns this bounty must:

- [ ] Implement `src/mount/windows.rs` behind `#[cfg(target_os = "windows")]` and wire it
      into `src/mount.rs` (no change to `EncryptedFs`).
- [ ] Implement the `MountPoint` trait and return a real `MountHandle`.
- [ ] Map the WinFsp callbacks to `EncryptedFs` methods per §5, with a path↔inode
      translation layer.
- [ ] Handle the licensing checkpoint: document the GPL-3 boundary (or use `unifuse`/MIT)
      and add the WinFsp attribution notice.
- [ ] Add the Windows-only dependency (feature-gated if the GPL boundary requires it).
- [ ] Compile on `x86_64-pc-windows-msvc` (add a Windows CI job).
- [ ] Pass the smoke test in §9 — read/write/list/create/delete round-trip on a real
      Windows host.
- [ ] Explicitly document unsupported NTFS features (§7) and return graceful errors for
      them.
- [ ] Self-review: no new panics, no silent data corruption, tests pass, no new warnings.

---

## 11. Open questions for the maintainer

1. **License:** accept the GPL-3 boundary for the Windows module (`winfsp-rs`), or prefer
   the MIT `unifuse` path (nightly Rust + new abstraction)? This decides the crate choice.
2. **CLI args:** confirm the exact mountpoint/data-dir argument order the Windows binary
   should accept (match `src/main.rs` / `src/run.rs`).
3. **Case sensitivity:** should encrypted names be case-sensitive on Windows (to match
   Linux) — I recommend yes.
4. **CI:** is a Windows runner acceptable, and can a real mount be exercised there, or
   should the Windows integration test skip gracefully when no mount is possible?

---

*This is a research note; it deliberately does not claim Windows support exists today.
The next step is a first implementation PR that satisfies §10.*
