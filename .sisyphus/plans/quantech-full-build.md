# Quantech Drive — Full Build Plan (Product Hunt MVP)

## TL;DR

> **Quick Summary**: Build the Quantech Drive MVP — a cross-platform encrypted virtual drive that gives users up to 5TB of organized, encrypted storage on their existing hardware. Ship a single Go binary with zero backend dependencies for Product Hunt launch.
>
> **Deliverables**:
> - `quantech` CLI binary (Linux + macOS) with commands: `init`, `mount`, `umount`, `status`
> - FUSE-based virtual drive with AES-256-GCM encryption and 24-word recovery key
> - Chunked storage (4 MiB) with bbolt metadata and WAL journal
> - Capacity enforcement (5TB free tier)
> - Load test/validation tool
> - CI pipeline (GitHub Actions) for Linux + macOS builds
>
> **Estimated Effort**: Large (11 major tasks, ~3-4 weeks AI-assisted)
> **Parallel Execution**: YES — 3 waves
> **Critical Path**: Task 1 → Task 3 → Task 6 → Task 7 → Task 9 → Task 11

---

## Context

### Original Request
Build the entire Quantech system from the 9,002-line QuanTech.md design document. Optimize for fastest Product Hunt viral launch. Solo founder using AI agents, no funding.

### Interview Summary
**Key Discussions**:
- Test strategy: TDD (Red-Green-Refactor) — write failing tests first, then implement
- Plan ordering: Optimize for Product Hunt MVP, not the document's Phase 0-7 sequence
- The "quantum" angle is marketing/branding — MVP is a solid encrypted drive, not quantum computing
- Backend (Cloudflare Workers), acceleration engine, resource sharing, and quantum simulation are all deferred to post-MVP

**Research Findings**:
- Project is 100% greenfield — only `QuanTech.md` exists
- The document contains a complete MVP specification (lines 7499-7912) with on-disk layouts, crypto specs, metadata schemas, and pseudo-Go code
- 12 well-scoped GitHub issues already defined (lines 8129-8348)
- Document transitions from "FractalAccel" to "Quantech" naming — must enforce "Quantech" consistently

### Metis Review
**Identified Gaps** (addressed):

| Gap | Resolution |
|-----|-----------|
| RocksDB vs LMDB never decided | Use **bbolt** (pure Go) for MVP — eliminates CGo complexity, cross-compiles trivially. RocksDB deferred to performance optimization phase |
| FUSE library never decided | Use **winfsp/cgofuse** — single API covering Linux, macOS, and future Windows |
| "5TB" claim ambiguity | Soft config limit on logical capacity. `quantech status` shows both logical cap and physical disk remaining |
| Per-chunk nonce reuse risk | Enforce immutable chunks (content-addressed). Update = new chunk ID + delete old. Deterministic nonce is safe |
| Recovery key wordlist unclear | Use BIP39 standard wordlist (well-known, auditable) |
| No concurrent mount protection | Add lockfile at mount time |
| No journal replay spec | Implement basic append-only WAL with replay on mount |
| Go module path undefined | Use `github.com/binetic-ai/quantech-drive` (from spec line 8371) |
| Symlinks/hardlinks unspec'd | Explicitly unsupported in MVP — return ENOSYS |
| File permissions unspec'd | Report current user's uid/gid for all files |
| Mount point creation | `quantech mount` creates mount point if it doesn't exist |
| Maximum single file size | Limited only by 5TB cap — chunk list in metadata handles up to 1.3M chunks |

---

## Work Objectives

### Core Objective
Ship a single Go binary (`quantech`) that creates, mounts, and manages an encrypted virtual drive with up to 5TB capacity — ready for Product Hunt demo within 3-4 weeks of AI-assisted development.

### Concrete Deliverables
- `quantech` binary for Linux (amd64, arm64) and macOS (amd64, arm64)
- FUSE virtual filesystem mountable as a normal drive
- AES-256-GCM per-chunk encryption with Argon2id KDF
- 24-word BIP39 recovery key
- CLI commands: `init`, `mount`, `umount`, `status`
- Comprehensive test suite (TDD)
- GitHub Actions CI for build + test
- Load test validation tool

### Definition of Done
- [ ] `go build ./...` succeeds with zero errors on Linux and macOS
- [ ] `go test ./...` passes with 100% of tests green
- [ ] `quantech init --encrypt=yes` creates correct directory structure and 92-byte keys.bin
- [ ] `quantech mount` exposes a working FUSE filesystem
- [ ] Files written to the mount persist across unmount/remount cycles
- [ ] 100MB file write+read roundtrip with encryption produces identical content
- [ ] Writing beyond 5TB capacity limit returns ENOSPC
- [ ] `quantech status` outputs valid JSON with drive metrics

### Must Have
- AES-256-GCM encryption with Argon2id KDF (time=3, mem=64MiB, p=4)
- 24-word BIP39 recovery key shown at init
- 4 MiB chunk size with content-addressed immutable chunks
- 92-byte keys.bin binary format (magic + salt + nonce + ciphertext+tag)
- Capacity enforcement at 5TB default
- Structured logging to `logs/quantech.log`
- Write-ahead journal for crash recovery
- Lockfile to prevent concurrent mount

### Must NOT Have (Guardrails)
- **No network calls** — MVP is 100% local, no HTTP clients, no Cloudflare, no telemetry
- **No GUI** — CLI only, no Electron, no web UI, no TUI frameworks
- **No acceleration engine** — zero CPU/GPU/SIMD optimization code
- **No quantum simulation** — zero quantum circuit code
- **No resource sharing** — zero P2P/mesh/donation code
- **No reactive database** — zero Convex-style subscriptions or WebSockets
- **No token purchasing flow** — token format defined but no payment integration
- **No recovery escrow service** — Phase 7 per spec
- **No Windows support** — Linux + macOS only in Phase 1
- **No Docker/Kubernetes** — Go compiles to static binary
- **No protobuf/gRPC** — JSON for all metadata serialization
- **No auto-updater** — Phase 6 per spec
- **No erasure coding** — single-copy chunks only (S0 tier)
- **No generated code** — no codegen tools
- **No name "FractalAccel"** — use "Quantech" consistently everywhere
- **Max 500 lines per file** — per project rules

---

## Verification Strategy (MANDATORY)

> **UNIVERSAL RULE: ZERO HUMAN INTERVENTION**
>
> ALL tasks in this plan MUST be verifiable WITHOUT any human action.
> This is NOT conditional — it applies to EVERY task, regardless of test strategy.
>
> **FORBIDDEN** — acceptance criteria that require:
> - "User manually tests..." / "User visually confirms..."
> - "User interacts with..." / "Ask user to verify..."
> - ANY step where a human must perform an action
>
> **ALL verification is executed by the agent** using tools (Bash, interactive_bash, etc.). No exceptions.

### Test Decision
- **Infrastructure exists**: NO (greenfield)
- **Automated tests**: TDD (Red-Green-Refactor)
- **Framework**: Go standard `testing` package + `go test`

### TDD Workflow (Every Task)

Each implementation task follows RED-GREEN-REFACTOR:

1. **RED**: Write failing test first
   - Test file: `{package}/{name}_test.go`
   - Test command: `go test ./{package}/... -v -run TestName`
   - Expected: FAIL (test exists, implementation doesn't)
2. **GREEN**: Implement minimum code to pass
   - Command: `go test ./{package}/... -v`
   - Expected: PASS
3. **REFACTOR**: Clean up while keeping green
   - Command: `go test ./... -v`
   - Expected: PASS (full suite still green)

### Test Infrastructure Setup (Task 1)

- [ ] Go module initialized: `go mod init github.com/binetic-ai/quantech-drive`
- [ ] `go test ./...` runs (even if no tests yet)
- [ ] CI pipeline runs `go test ./...` on every push

### Agent-Executed QA Scenarios (MANDATORY — ALL tasks)

> Every task includes Agent-Executed QA Scenarios as PRIMARY verification.
> The executing agent DIRECTLY verifies by running commands, checking outputs.

**Verification Tool by Deliverable Type:**

| Type | Tool | How Agent Verifies |
|------|------|-------------------|
| Go packages | Bash (`go test`) | Run tests, assert PASS |
| CLI commands | Bash / interactive_bash | Run command, parse output, assert fields |
| FUSE filesystem | interactive_bash (tmux) | Mount drive, write/read files, verify content |
| Binary builds | Bash (`go build`) | Build, assert exit code 0 |
| CI pipeline | Bash (`gh`) | Push, check workflow status |

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately):
├── Task 1: Repo scaffold + CI + Go module
└── (No other tasks — foundation must exist first)

Wave 2 (After Wave 1):
├── Task 2: Config package + init command skeleton
├── Task 3: Crypto package (keys, KDF, AES-GCM)
├── Task 4: Metadata store (bbolt)
└── Task 5: Chunk store (plain, no encryption)

Wave 3 (After Wave 2):
├── Task 6: Wire encryption into chunk store
├── Task 7: FUSE filesystem (Linux + macOS)
├── Task 8: Capacity enforcement
├── Task 9: CLI commands (mount/umount/status)
├── Task 10: Logging + error handling
└── Task 11: Load test + validation

Critical Path: 1 → 3 → 6 → 7 → 9 → 11
Parallel Speedup: ~40% faster than fully sequential
```

### Dependency Matrix

| Task | Depends On | Blocks | Can Parallelize With |
|------|------------|--------|---------------------|
| 1 | None | 2, 3, 4, 5 | None (foundation) |
| 2 | 1 | 7, 9 | 3, 4, 5 |
| 3 | 1 | 6 | 2, 4, 5 |
| 4 | 1 | 7 | 2, 3, 5 |
| 5 | 1 | 6, 7 | 2, 3, 4 |
| 6 | 3, 5 | 7 | 8, 10 |
| 7 | 2, 4, 5, 6 | 9, 11 | 8, 10 |
| 8 | 4, 5 | 11 | 6, 7, 10 |
| 9 | 2, 7 | 11 | 8, 10 |
| 10 | 1 | 11 | 6, 7, 8, 9 |
| 11 | 7, 8, 9 | None (final) | None |

### Agent Dispatch Summary

| Wave | Tasks | Recommended Agents |
|------|-------|-------------------|
| 1 | 1 | task(category="quick", load_skills=[], run_in_background=false) |
| 2 | 2, 3, 4, 5 | dispatch all 4 in parallel after Wave 1 |
| 3 | 6, 7, 8, 9, 10, 11 | dispatch in dependency order; 6+8+10 parallel, then 7+9, then 11 |

---

## TODOs

- [ ] 1. Repository Scaffold, Go Module, and CI Pipeline

  **What to do**:
  - Initialize Git repository in `/Users/b0s5i/Downloads/QuanTech-Drive`
  - Create Go module: `go mod init github.com/binetic-ai/quantech-drive`
  - Create directory structure:
    ```
    cmd/quantech/         # CLI entry point
    config/               # Config loading/saving
    crypto/               # Keys, KDF, AES-GCM
    chunkstore/           # Chunk I/O
    metastore/            # bbolt metadata
    fs/                   # FUSE filesystem
    drive/                # High-level drive logic
    internal/journal/     # WAL for crash recovery
    internal/testutil/    # Shared test helpers
    ```
  - Create `cmd/quantech/main.go` with Cobra CLI scaffold (subcommands: `init`, `mount`, `umount`, `status`)
  - Create `.gitignore` for Go projects
  - Create `.editorconfig`
  - Create `LICENSE` placeholder file with "Proprietary. All rights reserved. © 2026 Binetic.ai"
  - Create GitHub Actions workflow `.github/workflows/ci.yml`:
    - Matrix: `[ubuntu-latest, macos-latest]` × `[amd64]`
    - Steps: checkout, setup-go (1.22+), `go build ./...`, `go test ./...`
  - RED: Write `cmd/quantech/main_test.go` — test that CLI root command exists and `--help` returns exit 0
  - GREEN: Implement minimal Cobra root command
  - Verify: `go build ./cmd/quantech` produces binary, `go test ./...` passes

  **Must NOT do**:
  - Do not add any business logic
  - Do not create Windows-specific files
  - Do not add Docker/container files
  - Do not install RocksDB or any CGo dependencies

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Scaffold task with well-defined outputs, no complex logic
  - **Skills**: [`git-master`]
    - `git-master`: Needed for repo initialization and first commit

  **Parallelization**:
  - **Can Run In Parallel**: NO (foundation task)
  - **Parallel Group**: Wave 1 (solo)
  - **Blocks**: Tasks 2, 3, 4, 5, 6, 7, 8, 9, 10, 11
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `QuanTech.md:8390-8420` — Go module layout proposal from the spec (directories, packages)
  - `QuanTech.md:8401-8417` — Cobra CLI scaffold description with subcommands

  **Documentation References**:
  - `QuanTech.md:7920-8119` — GitHub README draft (use as template for repo README)
  - `QuanTech.md:8129-8142` — Issue 1 spec: "Setup repo and basic CI"

  **External References**:
  - `https://cobra.dev/` — Cobra CLI framework docs
  - `https://go.dev/doc/modules/` — Go module reference

  **WHY Each Reference Matters**:
  - Lines 8390-8420 define the exact package layout the rest of the plan depends on
  - Lines 8401-8417 show the Cobra subcommand structure that Tasks 2, 9 will extend
  - Lines 7920-8119 contain a ready-to-use README that should be the repo's README.md

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `cmd/quantech/main_test.go`
  - [ ] Test covers: root command exists, `--help` exits 0
  - [ ] `go test ./cmd/quantech/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Go module compiles successfully
    Tool: Bash
    Preconditions: Go 1.22+ installed
    Steps:
      1. cd /Users/b0s5i/Downloads/QuanTech-Drive
      2. go build ./...
      3. Assert: exit code 0
    Expected Result: Build succeeds with no errors
    Evidence: Build output captured

  Scenario: CLI binary runs with --help
    Tool: Bash
    Preconditions: Binary built
    Steps:
      1. ./cmd/quantech/quantech --help 2>&1 || go run ./cmd/quantech --help 2>&1
      2. Assert: output contains "quantech" (case-insensitive)
      3. Assert: output contains "init"
      4. Assert: output contains "mount"
      5. Assert: exit code 0
    Expected Result: Help text shows all subcommands
    Evidence: stdout captured

  Scenario: All tests pass
    Tool: Bash
    Preconditions: Go module initialized
    Steps:
      1. go test ./... -v
      2. Assert: output contains "PASS"
      3. Assert: exit code 0
    Expected Result: All tests green
    Evidence: Test output captured

  Scenario: CI workflow file is valid YAML
    Tool: Bash
    Preconditions: None
    Steps:
      1. cat .github/workflows/ci.yml | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin)" 2>&1
      2. Assert: exit code 0
    Expected Result: Valid YAML
    Evidence: Exit code captured

  Scenario: Directory structure is correct
    Tool: Bash
    Preconditions: None
    Steps:
      1. ls -d cmd/quantech config crypto chunkstore metastore fs drive internal/journal internal/testutil
      2. Assert: exit code 0 (all directories exist)
    Expected Result: All package directories created
    Evidence: ls output captured
  ```

  **Evidence to Capture:**
  - [ ] Build output in `.sisyphus/evidence/task-1-build.txt`
  - [ ] Test output in `.sisyphus/evidence/task-1-tests.txt`
  - [ ] CLI help output in `.sisyphus/evidence/task-1-help.txt`

  **Commit**: YES
  - Message: `feat(scaffold): initialize Go module, CLI skeleton, and CI pipeline`
  - Files: `go.mod`, `go.sum`, `cmd/quantech/main.go`, `cmd/quantech/main_test.go`, `.github/workflows/ci.yml`, `.gitignore`, `.editorconfig`, `README.md`, all package directories with `.gitkeep`
  - Pre-commit: `go test ./...`

---

- [ ] 2. Config Package and `quantech init` Command

  **What to do**:
  - RED: Write `config/config_test.go`:
    - `TestDriveConfigDefaults` — default config has correct values (5TB, version 1)
    - `TestSaveAndLoadConfig` — roundtrip: save config.json, load it, fields match
    - `TestInitCreatesDirectoryLayout` — init creates all required subdirs
    - `TestInitWithEncryption` — init with encrypt=yes creates keys.bin
  - GREEN: Implement:
    - `config/config.go`:
      - `DriveConfig` struct: `DriveID`, `Version`, `MountPoint`, `MaxLogicalBytes`, `EncryptionEnabled`, `EscrowOptIn`
      - `NewDriveConfig(opts)` — generates drive ID (`qd-` + 8 hex chars), sets defaults
      - `SaveConfig(rootPath, cfg)` — writes `config/config.json`
      - `LoadConfig(rootPath)` — reads and parses config
    - `drive/init.go`:
      - `InitDrive(rootPath, mountPoint, maxSize, encrypt)`:
        - Creates directory tree: `config/`, `meta/`, `data/chunks/`, `data/shared_chunks/`, `data/journal/`, `logs/`
        - Generates config, saves it
        - If encrypt: calls crypto package for key generation (stubbed in this task, wired in Task 3)
    - Wire `init` subcommand in `cmd/quantech/main.go`:
      - Flags: `--root` (default `/quantech`), `--mount-point`, `--size` (default `5TB`), `--encrypt` (default `yes`)
      - Calls `drive.InitDrive()`

  **Must NOT do**:
  - Do not implement actual encryption — that's Task 3
  - Do not implement metadata store — that's Task 4
  - Do not create Windows-specific paths

  **Recommended Agent Profile**:
  - **Category**: `unspecified-low`
    - Reason: Straightforward struct + JSON serialization + directory creation
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 3, 4, 5)
  - **Blocks**: Tasks 7, 9
  - **Blocked By**: Task 1

  **References**:

  **Pattern References**:
  - `QuanTech.md:7558-7571` — config.json example with exact field names and values
  - `QuanTech.md:7535-7555` — on-disk directory layout (exact tree structure)
  - `QuanTech.md:8146-8159` — Issue 2: "Implement `quantech init` and on-disk layout creation"

  **API/Type References**:
  - `QuanTech.md:7558-7571` — DriveConfig JSON schema (drive_id, version, mount_point, max_logical_bytes, encryption_enabled, escrow_opt_in)

  **WHY Each Reference Matters**:
  - Lines 7558-7571 define the EXACT config.json fields the rest of the system depends on
  - Lines 7535-7555 define the directory tree that ALL other tasks write into
  - Lines 8146-8159 list the CLI flags (`--root`, `--mount-point`, `--size`, `--encrypt`)

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `config/config_test.go`
  - [ ] Tests cover: defaults, save/load roundtrip, init directory creation
  - [ ] `go test ./config/... -v` → PASS
  - [ ] `go test ./drive/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: quantech init creates correct directory layout
    Tool: Bash
    Preconditions: Binary built
    Steps:
      1. rm -rf /tmp/qtest-init && mkdir -p /tmp/qtest-init
      2. go run ./cmd/quantech init --root=/tmp/qtest-init --mount-point=/tmp/qmount-init --size=5TB --encrypt=no
      3. Assert: exit code 0
      4. ls -d /tmp/qtest-init/config /tmp/qtest-init/meta /tmp/qtest-init/data/chunks /tmp/qtest-init/data/shared_chunks /tmp/qtest-init/data/journal /tmp/qtest-init/logs
      5. Assert: all directories exist (exit code 0)
      6. cat /tmp/qtest-init/config/config.json | python3 -c "import json,sys; d=json.load(sys.stdin); assert d['version']==1; assert d['max_logical_bytes']==5500000000000; assert d['encryption_enabled']==False; print('OK')"
      7. Assert: output is "OK"
    Expected Result: All directories and config.json created correctly
    Evidence: .sisyphus/evidence/task-2-init-layout.txt

  Scenario: quantech init generates unique drive IDs
    Tool: Bash
    Preconditions: Binary built
    Steps:
      1. rm -rf /tmp/qtest-id1 /tmp/qtest-id2
      2. go run ./cmd/quantech init --root=/tmp/qtest-id1 --mount-point=/tmp/qm1 --encrypt=no
      3. go run ./cmd/quantech init --root=/tmp/qtest-id2 --mount-point=/tmp/qm2 --encrypt=no
      4. ID1=$(cat /tmp/qtest-id1/config/config.json | python3 -c "import json,sys; print(json.load(sys.stdin)['drive_id'])")
      5. ID2=$(cat /tmp/qtest-id2/config/config.json | python3 -c "import json,sys; print(json.load(sys.stdin)['drive_id'])")
      6. Assert: $ID1 starts with "qd-"
      7. Assert: $ID2 starts with "qd-"
      8. Assert: $ID1 != $ID2
    Expected Result: Each init generates a unique drive ID
    Evidence: .sisyphus/evidence/task-2-unique-ids.txt

  Scenario: init fails gracefully on existing root
    Tool: Bash
    Preconditions: /tmp/qtest-init exists from previous scenario
    Steps:
      1. go run ./cmd/quantech init --root=/tmp/qtest-init --mount-point=/tmp/qmount-init --encrypt=no 2>&1
      2. Assert: output contains "already" or "exists" (warning/error)
    Expected Result: Graceful error, no data corruption
    Evidence: .sisyphus/evidence/task-2-existing-root.txt
  ```

  **Commit**: YES
  - Message: `feat(config): implement drive config, init command, and directory layout`
  - Files: `config/config.go`, `config/config_test.go`, `drive/init.go`, `drive/init_test.go`, `cmd/quantech/main.go` (updated)
  - Pre-commit: `go test ./...`

---

- [ ] 3. Crypto Package — Keys, KDF, AES-256-GCM

  **What to do**:
  - RED: Write `crypto/keys_test.go`:
    - `TestGenerateDriveKey` — DK is 32 bytes, non-zero
    - `TestGenerateRecoveryKey` — 24 words, space-separated, from BIP39 wordlist
    - `TestEncryptDecryptDriveKeyRoundtrip` — encrypt DK with password+recovery, decrypt matches original
    - `TestDecryptWithWrongPasswordFails` — wrong password → error
    - `TestDecryptWithWrongRecoveryKeyFails` — wrong recovery key → error
    - `TestKeysFileIs92Bytes` — serialized keys.bin is exactly 92 bytes
    - `TestKeysMagicBytes` — first 16 bytes match magic string `"QDKYv1\0\0\0\0\0\0\0\0\0\0"`
  - RED: Write `crypto/chunk_crypto_test.go`:
    - `TestEncryptDecryptChunk` — encrypt chunk with DK + chunk_id nonce → decrypt → matches original
    - `TestEncryptDifferentChunksProducesDifferentCiphertext` — same data, different chunk_ids → different ciphertext
    - `TestDecryptWithWrongKeyFails` — wrong DK → error
    - `TestDecryptTamperedDataFails` — flipped bit → authentication failure
  - GREEN: Implement:
    - `crypto/keys.go`:
      - `GenerateDriveKey() ([]byte, error)` — 32 random bytes from `crypto/rand`
      - `GenerateRecoveryKey() (string, error)` — 24 words from BIP39 wordlist
      - `DeriveKEK(password, recoveryKey string, salt []byte) []byte` — Argon2id(time=3, mem=64MiB, threads=4, keyLen=32)
      - `EncryptDriveKey(dk []byte, password, recoveryKey string) ([]byte, error)` — returns 92-byte blob (magic + salt + nonce + ciphertext+tag)
      - `DecryptDriveKey(data []byte, password, recoveryKey string) ([]byte, error)` — parses 92-byte blob, derives KEK, decrypts
      - `SaveKeysFile(path string, data []byte) error`
      - `LoadKeysFile(path string) ([]byte, error)`
    - `crypto/chunk_crypto.go`:
      - `EncryptChunk(dk, plaintext []byte, chunkID string) ([]byte, error)` — AES-256-GCM, nonce=SHA256(chunkID)[:12], returns ciphertext||tag
      - `DecryptChunk(dk, ciphertext []byte, chunkID string) ([]byte, error)` — reverse
  - Wire key generation into `drive/init.go`:
    - When `--encrypt=yes`: call `GenerateDriveKey()`, `GenerateRecoveryKey()`, prompt for password, `EncryptDriveKey()`, save to `config/keys.bin`
    - Print recovery key to stdout with clear "SAVE THIS" warning

  **Must NOT do**:
  - Do not implement key escrow or server-side recovery
  - Do not implement token/license signing (that's backend, not MVP)
  - Do not add any network calls

  **Recommended Agent Profile**:
  - **Category**: `ultrabrain`
    - Reason: Cryptographic code requires careful correctness — nonce handling, KDF parameters, authentication tag verification. Security-critical.
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 2, 4, 5)
  - **Blocks**: Task 6
  - **Blocked By**: Task 1

  **References**:

  **Pattern References**:
  - `QuanTech.md:7574-7583` — keys.bin binary format: 16B magic + 16B salt + 12B nonce + 48B (ciphertext+tag) = 92 bytes
  - `QuanTech.md:8075-8091` — Encryption spec: DK 32 bytes, 24-word mnemonic, Argon2id, AES-256-GCM, per-chunk nonce = SHA256(chunk_id)[:12]
  - `QuanTech.md:8162-8183` — Issue 3: key generation, recovery key, AES-GCM implementation

  **API/Type References**:
  - `QuanTech.md:8087-8089` — Per-chunk encryption spec: nonce = first 12 bytes of SHA256(chunk_id), AES-256-GCM with DK, chunk file = ciphertext || tag

  **External References**:
  - `golang.org/x/crypto/argon2` — Argon2id KDF
  - `crypto/aes` + `crypto/cipher` — Go stdlib AES-GCM
  - `github.com/tyler-smith/go-bip39` — BIP39 mnemonic wordlist

  **WHY Each Reference Matters**:
  - Lines 7574-7583 define the EXACT binary layout of keys.bin — byte offsets matter
  - Lines 8075-8091 specify the Argon2id parameters (time=3, mem=64MiB, p=4) which MUST be exact for interop
  - Per-chunk nonce derivation from chunk_id is a security-critical detail that must be implemented exactly

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `crypto/keys_test.go`
  - [ ] Test file created: `crypto/chunk_crypto_test.go`
  - [ ] Tests cover: DK generation, recovery key generation, encrypt/decrypt roundtrip, wrong password, wrong recovery key, 92-byte format, magic bytes, chunk encrypt/decrypt, tampered data detection
  - [ ] `go test ./crypto/... -v` → PASS (all tests)

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Keys.bin is exactly 92 bytes with correct magic
    Tool: Bash
    Preconditions: Binary built
    Steps:
      1. rm -rf /tmp/qtest-crypto && mkdir -p /tmp/qtest-crypto/config
      2. go run ./cmd/quantech init --root=/tmp/qtest-crypto --mount-point=/tmp/qm-crypto --encrypt=yes <<< "testpassword123"
      3. FILESIZE=$(stat -f%z /tmp/qtest-crypto/config/keys.bin 2>/dev/null || stat -c%s /tmp/qtest-crypto/config/keys.bin)
      4. Assert: $FILESIZE equals "92"
      5. MAGIC=$(xxd -l 6 -p /tmp/qtest-crypto/config/keys.bin)
      6. Assert: $MAGIC starts with "51444b597631" (hex for "QDKYv1")
    Expected Result: keys.bin is 92 bytes with correct magic header
    Evidence: .sisyphus/evidence/task-3-keysbin.txt

  Scenario: Recovery key is 24 words
    Tool: Bash
    Preconditions: Binary built
    Steps:
      1. rm -rf /tmp/qtest-recovery
      2. OUTPUT=$(go run ./cmd/quantech init --root=/tmp/qtest-recovery --mount-point=/tmp/qm-recovery --encrypt=yes <<< "testpass" 2>&1)
      3. RECOVERY_LINE=$(echo "$OUTPUT" | grep -i "recovery" | head -1)
      4. WORD_COUNT=$(echo "$RECOVERY_LINE" | tr ' ' '\n' | wc -l)
      5. Assert: $WORD_COUNT >= 24
    Expected Result: 24-word recovery key displayed
    Evidence: .sisyphus/evidence/task-3-recovery.txt

  Scenario: All crypto tests pass
    Tool: Bash
    Preconditions: Go module set up
    Steps:
      1. go test ./crypto/... -v -count=1
      2. Assert: output contains "PASS"
      3. Assert: exit code 0
      4. Assert: output contains "TestEncryptDecryptDriveKeyRoundtrip"
      5. Assert: output contains "TestDecryptWithWrongPasswordFails"
    Expected Result: All crypto tests green
    Evidence: .sisyphus/evidence/task-3-tests.txt
  ```

  **Commit**: YES
  - Message: `feat(crypto): implement drive key, recovery key, AES-256-GCM encryption`
  - Files: `crypto/keys.go`, `crypto/keys_test.go`, `crypto/chunk_crypto.go`, `crypto/chunk_crypto_test.go`, `go.mod` (updated deps), `go.sum`
  - Pre-commit: `go test ./crypto/...`

---

- [ ] 4. Metadata Store (bbolt)

  **What to do**:
  - RED: Write `metastore/meta_test.go`:
    - `TestOpenAndCloseMetaStore` — open bbolt DB at path, close without error
    - `TestPutGetObjectMeta` — store object metadata, retrieve by ID, fields match
    - `TestPutGetChunkMeta` — store chunk metadata, retrieve by ID, fields match
    - `TestGetNonexistentObjectReturnsError` — missing key returns specific error
    - `TestGetNonexistentChunkReturnsError`
    - `TestUpdateDriveInfo` — store and retrieve total_logical_bytes, used_bytes
    - `TestDeleteObjectMeta` — delete object, subsequent get returns error
    - `TestDeleteChunkMeta` — delete chunk, subsequent get returns error
    - `TestListObjects` — store multiple objects, list returns all
  - GREEN: Implement:
    - `metastore/types.go`:
      - `ObjectMeta` struct: `ObjectID`, `SizeBytes`, `Chunks []ChunkRef`, `MTime`, `CTime`
      - `ChunkRef` struct: `ChunkID`, `Offset`, `Size`
      - `ChunkMeta` struct: `ChunkID`, `Size`, `Checksum`, `Layout`, `RefCount`
      - `DriveInfo` struct: `TotalLogicalBytes`, `UsedBytes`
    - `metastore/meta.go`:
      - `OpenMetaStore(path string) (*MetaStore, error)` — opens bbolt at `meta/meta.db`
      - `Close() error`
      - `PutObjectMeta(meta *ObjectMeta) error` — key: `obj:<object_id>`, value: JSON
      - `GetObjectMeta(objectID string) (*ObjectMeta, error)`
      - `DeleteObjectMeta(objectID string) error`
      - `ListObjects() ([]*ObjectMeta, error)`
      - `PutChunkMeta(meta *ChunkMeta) error` — key: `chunk:<chunk_id>`, value: JSON
      - `GetChunkMeta(chunkID string) (*ChunkMeta, error)`
      - `DeleteChunkMeta(chunkID string) error`
      - `GetDriveInfo() (*DriveInfo, error)`
      - `UpdateDriveInfo(info *DriveInfo) error`

  **Must NOT do**:
  - Do not use RocksDB or LMDB — use bbolt (pure Go)
  - Do not implement graph/layout metadata (S1+ tiers)
  - Do not add any caching layers

  **Recommended Agent Profile**:
  - **Category**: `unspecified-low`
    - Reason: Standard CRUD operations on an embedded KV store, well-defined interfaces
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 2, 3, 5)
  - **Blocks**: Tasks 7, 8
  - **Blocked By**: Task 1

  **References**:

  **Pattern References**:
  - `QuanTech.md:7591-7638` — Metadata schema: key prefixes `obj:<object_id>`, `chunk:<chunk_id>`, `drive:info` with full JSON examples
  - `QuanTech.md:8185-8201` — Issue 4: metadata store integration spec with method signatures

  **API/Type References**:
  - `QuanTech.md:7596-7617` — ObjectMeta JSON structure (object_id, size_bytes, chunks[], mtime, ctime)
  - `QuanTech.md:7621-7633` — ChunkMeta JSON structure (chunk_id, size, checksum, layout, refcount)

  **External References**:
  - `https://pkg.go.dev/go.etcd.io/bbolt` — bbolt documentation (Go embedded KV store)

  **WHY Each Reference Matters**:
  - Lines 7591-7638 define the exact key format and JSON value structure — the FUSE layer and chunk store depend on these exact interfaces
  - Lines 8185-8201 list the required method signatures that the rest of the system calls

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `metastore/meta_test.go`
  - [ ] Tests cover: open/close, put/get object, put/get chunk, nonexistent key, drive info, delete, list
  - [ ] `go test ./metastore/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Metadata store tests all pass
    Tool: Bash
    Preconditions: Go module with bbolt dependency
    Steps:
      1. go test ./metastore/... -v -count=1
      2. Assert: output contains "PASS"
      3. Assert: exit code 0
      4. Assert: output contains "TestPutGetObjectMeta"
      5. Assert: output contains "TestPutGetChunkMeta"
      6. Assert: output contains "TestGetNonexistentObjectReturnsError"
    Expected Result: All metadata tests green
    Evidence: .sisyphus/evidence/task-4-tests.txt

  Scenario: bbolt file is created on disk
    Tool: Bash
    Preconditions: metastore package compiled
    Steps:
      1. go test ./metastore/... -v -run TestOpenAndCloseMetaStore
      2. Assert: PASS
    Expected Result: bbolt opens and closes without error
    Evidence: .sisyphus/evidence/task-4-bbolt.txt
  ```

  **Commit**: YES
  - Message: `feat(metastore): implement bbolt-based metadata store with object and chunk CRUD`
  - Files: `metastore/types.go`, `metastore/meta.go`, `metastore/meta_test.go`, `go.mod` (add bbolt), `go.sum`
  - Pre-commit: `go test ./metastore/...`

---

- [ ] 5. Chunk Store — Plain Read/Write (No Encryption)

  **What to do**:
  - RED: Write `chunkstore/store_test.go`:
    - `TestGenerateChunkID` — chunk ID starts with `ch:`, is 67 chars total (`ch:` + 64 hex)
    - `TestChunkIDsAreUnique` — 100 generated IDs are all different
    - `TestWriteReadRoundtrip` — write 4MiB of data, read back, content matches
    - `TestReadNonexistentChunkReturnsError`
    - `TestWriteCreatesDirectoryStructure` — chunk path has 2-level prefix directories
    - `TestDeleteChunk` — write, delete, read returns error
    - `TestChunkPathMapping` — chunk ID `ch:abcd1234...` maps to `ab/cd/ch:abcd1234...`
  - GREEN: Implement:
    - `chunkstore/store.go`:
      - `const DefaultChunkSize = 4 * 1024 * 1024` (4 MiB)
      - `type ChunkStore struct` with `rootPath string`
      - `NewChunkStore(rootPath string) *ChunkStore`
      - `GenerateChunkID() (string, error)` — `"ch:" + hex(SHA256(random_bytes))`, 64 hex chars
      - `chunkPath(chunkID string) string` — maps `ch:abcd1234...` → `<root>/ab/cd/ch:abcd1234...`
      - `WriteChunk(chunkID string, data []byte) error` — mkdir -p, write, fsync
      - `ReadChunk(chunkID string) ([]byte, error)` — read full file
      - `DeleteChunk(chunkID string) error` — remove file
      - `ChunkExists(chunkID string) bool`

  **Must NOT do**:
  - Do not add encryption in this task — that's Task 6
  - Do not add metadata integration — chunks are standalone I/O
  - Do not add deduplication logic (post-MVP)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-low`
    - Reason: Straightforward file I/O with directory sharding, well-defined spec
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 2, 3, 4)
  - **Blocks**: Tasks 6, 7, 8
  - **Blocked By**: Task 1

  **References**:

  **Pattern References**:
  - `QuanTech.md:7641-7681` — Chunk ID format, path mapping, write path pseudo-Go code
  - `QuanTech.md:7686-7700` — Read path pseudo-Go code
  - `QuanTech.md:8205-8219` — Issue 5: chunk store spec with method signatures

  **API/Type References**:
  - `QuanTech.md:7645-7649` — Chunk ID format: `"ch:" + hex(SHA256(random + counter))`, path: `/<p0><p1>/<p2><p3>/<chunk_id>`
  - `QuanTech.md:7654-7657` — Default chunk size: 4 MiB

  **WHY Each Reference Matters**:
  - Lines 7641-7681 contain actual pseudo-Go code for WriteChunk that can be adapted directly
  - Lines 7645-7649 define the exact path sharding scheme (2-char prefixes) used by all disk operations
  - Lines 8205-8219 confirm the exact method signatures expected

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `chunkstore/store_test.go`
  - [ ] Tests cover: ID generation, uniqueness, write/read roundtrip, nonexistent chunk, directory creation, delete, path mapping
  - [ ] `go test ./chunkstore/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Chunk store write/read roundtrip
    Tool: Bash
    Preconditions: Go module compiled
    Steps:
      1. go test ./chunkstore/... -v -run TestWriteReadRoundtrip
      2. Assert: PASS
    Expected Result: Written data matches read data
    Evidence: .sisyphus/evidence/task-5-roundtrip.txt

  Scenario: All chunk store tests pass
    Tool: Bash
    Preconditions: Go module compiled
    Steps:
      1. go test ./chunkstore/... -v -count=1
      2. Assert: output contains "PASS"
      3. Assert: exit code 0
    Expected Result: All tests green
    Evidence: .sisyphus/evidence/task-5-tests.txt
  ```

  **Commit**: YES
  - Message: `feat(chunkstore): implement chunk store with 4MiB chunks and 2-level directory sharding`
  - Files: `chunkstore/store.go`, `chunkstore/store_test.go`
  - Pre-commit: `go test ./chunkstore/...`

---

- [ ] 6. Wire Encryption into Chunk Store

  **What to do**:
  - RED: Write additional tests in `chunkstore/store_test.go` or `chunkstore/encrypted_store_test.go`:
    - `TestEncryptedWriteReadRoundtrip` — write with encryption, read back, plaintext matches
    - `TestEncryptedChunkFileIsDifferentFromPlaintext` — ciphertext on disk != plaintext
    - `TestEncryptedChunkSizeIsLargerThanPlaintext` — ciphertext includes GCM tag (16 bytes)
    - `TestDecryptWithWrongKeyFails` — reading with wrong DK returns error
  - GREEN: Implement:
    - `chunkstore/encrypted_store.go`:
      - `type EncryptedChunkStore struct` — wraps `ChunkStore` + holds DK
      - `NewEncryptedChunkStore(rootPath string, dk []byte) *EncryptedChunkStore`
      - `WriteChunk(chunkID string, plaintext []byte) error` — encrypt with crypto.EncryptChunk(dk, plaintext, chunkID), then write ciphertext
      - `ReadChunk(chunkID string) ([]byte, error)` — read ciphertext, decrypt with crypto.DecryptChunk(dk, ciphertext, chunkID)
      - Implements same interface as plain ChunkStore
    - Define a `ChunkWriter` / `ChunkReader` interface that both plain and encrypted stores implement
  - REFACTOR: Ensure both plain and encrypted stores satisfy the same interface

  **Must NOT do**:
  - Do not change the plain ChunkStore
  - Do not add key management here — DK is passed in from outside

  **Recommended Agent Profile**:
  - **Category**: `unspecified-low`
    - Reason: Wrapper pattern — delegates to existing crypto and chunkstore packages
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (after Wave 2)
  - **Parallel Group**: Wave 3a (with Tasks 8, 10)
  - **Blocks**: Task 7
  - **Blocked By**: Tasks 3, 5

  **References**:

  **Pattern References**:
  - `QuanTech.md:8222-8239` — Issue 6: wire encryption into chunk store, modify WriteChunk/ReadChunk
  - `QuanTech.md:8087-8089` — Per-chunk encryption spec: nonce from chunk_id, ciphertext || tag

  **API/Type References**:
  - `crypto/chunk_crypto.go` (from Task 3) — `EncryptChunk()`, `DecryptChunk()` functions
  - `chunkstore/store.go` (from Task 5) — `WriteChunk()`, `ReadChunk()` signatures

  **WHY Each Reference Matters**:
  - Lines 8222-8239 describe exactly how to modify the chunk store for encryption
  - The crypto package from Task 3 provides the actual encrypt/decrypt functions

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `chunkstore/encrypted_store_test.go`
  - [ ] Tests cover: encrypted roundtrip, ciphertext differs from plaintext, wrong key fails
  - [ ] `go test ./chunkstore/... -v` → PASS (all tests including plain + encrypted)

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Encrypted chunk is unreadable without key
    Tool: Bash
    Preconditions: chunkstore and crypto packages compiled
    Steps:
      1. go test ./chunkstore/... -v -run TestDecryptWithWrongKeyFails
      2. Assert: PASS
    Expected Result: Wrong key produces authentication error
    Evidence: .sisyphus/evidence/task-6-wrongkey.txt

  Scenario: All encrypted store tests pass
    Tool: Bash
    Preconditions: Go module compiled
    Steps:
      1. go test ./chunkstore/... -v -count=1
      2. Assert: output contains "PASS"
      3. Assert: exit code 0
    Expected Result: All tests green
    Evidence: .sisyphus/evidence/task-6-tests.txt
  ```

  **Commit**: YES
  - Message: `feat(chunkstore): add encrypted chunk store wrapper with AES-256-GCM per-chunk encryption`
  - Files: `chunkstore/encrypted_store.go`, `chunkstore/encrypted_store_test.go`, `chunkstore/interface.go`
  - Pre-commit: `go test ./chunkstore/...`

---

- [ ] 7. FUSE Filesystem (Linux + macOS)

  **What to do**:
  - RED: Write `fs/fuse_test.go` (integration tests — require FUSE):
    - `TestMountAndUnmount` — mount FUSE, verify mount point exists, unmount cleanly
    - `TestCreateAndReadFile` — create file, read back, content matches
    - `TestCreateDirectory` — mkdir, verify with readdir
    - `TestWriteLargeFile` — write 20MB file (5 chunks), read back, matches
    - `TestDeleteFile` — create file, delete, read returns error
    - `TestReadDir` — create multiple files, readdir lists all
    - `TestFilePersistsAcrossRemount` — write, unmount, remount, read matches
    - `TestSymlinksReturnEnosys` — symlink attempt returns ENOSYS
  - GREEN: Implement:
    - `fs/quantechfs.go`:
      - `type QuantechFS struct` — holds metastore, chunkstore (encrypted or plain), config
      - `NewQuantechFS(meta *metastore.MetaStore, chunks chunkstore.ChunkReadWriter, cfg *config.DriveConfig) *QuantechFS`
      - Implement cgofuse `FileSystemInterface`:
        - `Init()` — initialize, load metadata
        - `Destroy()` — cleanup
        - `Getattr(path)` — return file attributes from metastore
        - `Readdir(path)` — list directory entries from metastore
        - `Create(path, flags, mode)` — create new file, store empty ObjectMeta
        - `Open(path, flags)` — open existing file
        - `Read(path, buf, offset)` — read chunks from chunkstore, decrypt if needed, assemble
        - `Write(path, buf, offset)` — split into chunks, write to chunkstore, update metadata
        - `Truncate(path, size)` — resize file, update chunks
        - `Unlink(path)` — delete file, decrement chunk refcounts, delete orphan chunks
        - `Mkdir(path, mode)` — create directory in metadata
        - `Rmdir(path)` — remove empty directory
        - `Rename(oldpath, newpath)` — rename in metadata
        - `Statfs(path)` — return capacity info (total=max_logical_bytes, used=used_bytes)
    - `drive/mount.go`:
      - `MountDrive(rootPath, password string) error`:
        - Load config
        - If encrypted: load keys.bin, prompt password, decrypt DK
        - Open metastore
        - Create chunkstore (encrypted or plain based on config)
        - Create lockfile at `<root>/.quantech.lock`
        - Create mount point if not exists
        - Start FUSE mount
      - `UnmountDrive(rootPath string) error`:
        - Unmount FUSE
        - Close metastore
        - Remove lockfile
    - `internal/journal/wal.go`:
      - Simple append-only WAL for multi-chunk writes
      - `BeginTransaction(txID)` — write begin marker
      - `RecordChunkWrite(txID, chunkID, objectID)` — log intent
      - `CommitTransaction(txID)` — write commit marker
      - `Replay()` — on mount, replay uncommitted transactions (delete orphan chunks)

  **Must NOT do**:
  - Do not implement Windows filesystem (WinFsp/Dokany)
  - Do not implement advanced FUSE operations (xattrs, mmap, fallocate)
  - Do not implement file locking (flock)
  - Do not implement hard links or symlinks — return ENOSYS

  **Recommended Agent Profile**:
  - **Category**: `ultrabrain`
    - Reason: FUSE filesystem is complex systems programming — requires careful handling of file operations, chunk assembly, metadata consistency, and crash recovery. Most technically challenging task in the plan.
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (depends on Tasks 2, 4, 5, 6)
  - **Parallel Group**: Wave 3b (after 3a completes)
  - **Blocks**: Tasks 9, 11
  - **Blocked By**: Tasks 2, 4, 5, 6

  **References**:

  **Pattern References**:
  - `QuanTech.md:7486-7499` — Virtual drive integration: FUSE mount at `/mnt/quantech`, operations → meta.db + chunks
  - `QuanTech.md:7720-7730` — Filesystem interface: WriteFile (split→encrypt→write→update meta), ReadFile (lookup→read→decrypt→reassemble)
  - `QuanTech.md:8242-8257` — Issue 7: FUSE filesystem MVP with required operations list
  - `QuanTech.md:8530-8553` — FUSE implementation prompt with bazil.org/fuse methods

  **API/Type References**:
  - `metastore/types.go` (from Task 4) — ObjectMeta, ChunkRef, ChunkMeta, DriveInfo
  - `chunkstore/interface.go` (from Task 6) — ChunkReadWriter interface
  - `config/config.go` (from Task 2) — DriveConfig with mount point and encryption flag

  **External References**:
  - `https://pkg.go.dev/github.com/winfsp/cgofuse/fuse` — cgofuse API reference
  - `https://github.com/winfsp/cgofuse` — cgofuse examples and usage

  **WHY Each Reference Matters**:
  - Lines 7720-7730 define the EXACT read/write flow: chunk splitting, encryption pipeline, metadata updates
  - Lines 8242-8257 list the required FUSE operations for MVP
  - cgofuse provides a single API for Linux + macOS (+ future Windows)

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `fs/fuse_test.go`
  - [ ] Test file created: `internal/journal/wal_test.go`
  - [ ] Tests cover: mount/unmount, create/read file, large file (multi-chunk), delete, readdir, persist across remount, symlink ENOSYS
  - [ ] `go test ./fs/... -v` → PASS
  - [ ] `go test ./internal/journal/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Mount drive and write/read a file
    Tool: interactive_bash (tmux)
    Preconditions: quantech binary built, FUSE installed
    Steps:
      1. rm -rf /tmp/qtest-fuse /tmp/qmount-fuse
      2. go run ./cmd/quantech init --root=/tmp/qtest-fuse --mount-point=/tmp/qmount-fuse --encrypt=no
      3. tmux new-session -d -s qfuse "go run ./cmd/quantech mount --root=/tmp/qtest-fuse"
      4. sleep 3
      5. echo "hello quantech drive" > /tmp/qmount-fuse/test.txt
      6. CONTENT=$(cat /tmp/qmount-fuse/test.txt)
      7. Assert: $CONTENT equals "hello quantech drive"
      8. fusermount -u /tmp/qmount-fuse || umount /tmp/qmount-fuse
    Expected Result: File written and read back successfully
    Evidence: .sisyphus/evidence/task-7-readwrite.txt

  Scenario: Large file persists across remount
    Tool: interactive_bash (tmux)
    Preconditions: FUSE drive mounted
    Steps:
      1. rm -rf /tmp/qtest-persist /tmp/qmount-persist
      2. go run ./cmd/quantech init --root=/tmp/qtest-persist --mount-point=/tmp/qmount-persist --encrypt=yes <<< "testpass"
      3. tmux new-session -d -s qpersist "go run ./cmd/quantech mount --root=/tmp/qtest-persist --password=testpass"
      4. sleep 3
      5. dd if=/dev/urandom of=/tmp/qmount-persist/bigfile.bin bs=1M count=20
      6. md5sum /tmp/qmount-persist/bigfile.bin > /tmp/expected_hash.txt
      7. fusermount -u /tmp/qmount-persist || umount /tmp/qmount-persist
      8. sleep 1
      9. tmux new-session -d -s qpersist2 "go run ./cmd/quantech mount --root=/tmp/qtest-persist --password=testpass"
      10. sleep 3
      11. md5sum /tmp/qmount-persist/bigfile.bin > /tmp/actual_hash.txt
      12. diff /tmp/expected_hash.txt /tmp/actual_hash.txt
      13. Assert: exit code 0 (hashes match)
      14. fusermount -u /tmp/qmount-persist || umount /tmp/qmount-persist
    Expected Result: Data persists and matches after remount
    Evidence: .sisyphus/evidence/task-7-persist.txt

  Scenario: Lockfile prevents concurrent mount
    Tool: Bash
    Preconditions: Drive already mounted from previous scenario
    Steps:
      1. go run ./cmd/quantech mount --root=/tmp/qtest-fuse 2>&1
      2. Assert: output contains "lock" or "already mounted" or error
      3. Assert: exit code != 0
    Expected Result: Second mount attempt fails with clear error
    Evidence: .sisyphus/evidence/task-7-lockfile.txt
  ```

  **Commit**: YES (groups Tasks 7)
  - Message: `feat(fs): implement FUSE filesystem with chunked I/O, encryption, and WAL journal`
  - Files: `fs/quantechfs.go`, `fs/fuse_test.go`, `drive/mount.go`, `drive/mount_test.go`, `internal/journal/wal.go`, `internal/journal/wal_test.go`, `go.mod` (add cgofuse)
  - Pre-commit: `go test ./...`

---

- [ ] 8. Capacity Enforcement (5TB Free Tier)

  **What to do**:
  - RED: Write `drive/capacity_test.go`:
    - `TestCapacityCheckAllowsWriteUnderLimit` — writing below cap succeeds
    - `TestCapacityCheckBlocksWriteOverLimit` — writing beyond cap returns ErrNoSpace
    - `TestCapacityTrackingAccumulates` — multiple writes accumulate correctly
    - `TestCapacityFreedOnDelete` — deleting file reduces used bytes
    - `TestCapacityReflectsInDriveInfo` — metastore.DriveInfo.UsedBytes matches actual
  - GREEN: Implement:
    - `drive/capacity.go`:
      - `type CapacityManager struct` — holds metastore ref, max_logical_bytes from config
      - `NewCapacityManager(meta *metastore.MetaStore, maxBytes int64) *CapacityManager`
      - `CheckWrite(deltaBytes int64) error` — returns `ErrNoSpace` if would exceed limit
      - `RecordWrite(deltaBytes int64) error` — updates DriveInfo.UsedBytes in metastore
      - `RecordDelete(deltaBytes int64) error` — decrements used bytes
      - `GetUsage() (used, max int64, err error)` — returns current usage
    - Wire into FUSE Write/Create/Unlink operations in `fs/quantechfs.go`

  **Must NOT do**:
  - Do not implement dynamic capacity changes (tier upgrades)
  - Do not implement thin provisioning alerts
  - Do not add physical disk space checks (that's a future enhancement)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple arithmetic check + metastore update, well-defined interface
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3a (with Tasks 6, 10)
  - **Blocks**: Task 11
  - **Blocked By**: Tasks 4, 5

  **References**:

  **Pattern References**:
  - `QuanTech.md:8274-8289` — Issue 9: capacity enforcement spec — track total_logical_bytes, reject writes exceeding max, ENOSPC
  - `QuanTech.md:7563-7565` — config.json `max_logical_bytes` default: 5500000000000 (5TB)

  **WHY Each Reference Matters**:
  - Lines 8274-8289 define the exact enforcement behavior: check delta, deny with ENOSPC
  - Lines 7563-7565 confirm the default cap value used in config

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `drive/capacity_test.go`
  - [ ] Tests cover: allow under limit, block over limit, accumulation, freed on delete
  - [ ] `go test ./drive/... -v -run TestCapacity` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Writing beyond capacity returns ENOSPC
    Tool: interactive_bash (tmux)
    Preconditions: Binary built, FUSE installed
    Steps:
      1. rm -rf /tmp/qtest-cap /tmp/qmount-cap
      2. go run ./cmd/quantech init --root=/tmp/qtest-cap --mount-point=/tmp/qmount-cap --size=10MB --encrypt=no
      3. tmux new-session -d -s qcap "go run ./cmd/quantech mount --root=/tmp/qtest-cap"
      4. sleep 3
      5. dd if=/dev/urandom of=/tmp/qmount-cap/fill.bin bs=1M count=20 2>&1
      6. Assert: output contains "No space left" or "ENOSPC" or dd reports error
      7. fusermount -u /tmp/qmount-cap || umount /tmp/qmount-cap
    Expected Result: Write blocked at capacity limit
    Evidence: .sisyphus/evidence/task-8-enospc.txt
  ```

  **Commit**: YES (groups with Task 7 or standalone)
  - Message: `feat(drive): add capacity enforcement with 5TB free tier limit`
  - Files: `drive/capacity.go`, `drive/capacity_test.go`
  - Pre-commit: `go test ./drive/...`

---

- [ ] 9. CLI Commands — mount, umount, status

  **What to do**:
  - RED: Write `cmd/quantech/commands_test.go`:
    - `TestStatusOutputContainsRequiredFields` — JSON output has: drive_id, mount_point, used_bytes, max_logical_bytes, encryption_enabled, mounted
    - `TestStatusShowsEncryptionFlag` — encryption_enabled reflects config
    - `TestMountRequiresInitializedDrive` — mount on non-initialized root → error
  - GREEN: Implement:
    - `cmd/quantech/cmd_mount.go`:
      - Cobra `mount` command
      - Flags: `--root`, `--password` (if encrypted, prompt interactively if not provided)
      - Calls `drive.MountDrive(rootPath, password)`
      - Runs FUSE in foreground (or `--daemon` flag for background)
    - `cmd/quantech/cmd_umount.go`:
      - Cobra `umount` command
      - Flags: `--root`
      - Calls `drive.UnmountDrive(rootPath)`
    - `cmd/quantech/cmd_status.go`:
      - Cobra `status` command
      - Flags: `--root`, `--json` (default true)
      - Outputs JSON with: drive_id, mount_point, used_bytes, max_logical_bytes, encryption_enabled, mounted (check lockfile)
      - Human-readable format option: `--human`

  **Must NOT do**:
  - Do not implement `activate`, `share-resources`, or `diagnose` — those are post-MVP
  - Do not add interactive TUI
  - Do not add auto-mount on boot

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: CLI wiring — connecting existing packages to Cobra commands
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 3b (after Task 7 FUSE is ready)
  - **Blocks**: Task 11
  - **Blocked By**: Tasks 2, 7

  **References**:

  **Pattern References**:
  - `QuanTech.md:8293-8315` — Issue 10: mount/umount/status CLI spec with exact output fields
  - `QuanTech.md:8014-8041` — CLI usage examples from README: `quantech init`, `quantech mount`, `quantech status`, `quantech umount`

  **WHY Each Reference Matters**:
  - Lines 8293-8315 define the exact CLI behavior, flags, and output format
  - Lines 8014-8041 show the user-facing usage that Product Hunt users will see

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `cmd/quantech/commands_test.go`
  - [ ] Tests cover: status JSON fields, encryption flag, mount without init
  - [ ] `go test ./cmd/quantech/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: quantech status outputs valid JSON with required fields
    Tool: Bash
    Preconditions: Drive initialized at /tmp/qtest-status
    Steps:
      1. rm -rf /tmp/qtest-status
      2. go run ./cmd/quantech init --root=/tmp/qtest-status --mount-point=/tmp/qm-status --encrypt=yes <<< "testpass"
      3. go run ./cmd/quantech status --root=/tmp/qtest-status 2>&1
      4. Assert: output is valid JSON (pipe to python3 -c "import json,sys; json.load(sys.stdin)")
      5. Assert: JSON contains keys: drive_id, mount_point, max_logical_bytes, encryption_enabled
      6. Assert: encryption_enabled is true
    Expected Result: Status outputs complete JSON
    Evidence: .sisyphus/evidence/task-9-status.txt

  Scenario: mount without init fails gracefully
    Tool: Bash
    Preconditions: /tmp/qtest-noinit does NOT exist
    Steps:
      1. rm -rf /tmp/qtest-noinit
      2. go run ./cmd/quantech mount --root=/tmp/qtest-noinit 2>&1
      3. Assert: exit code != 0
      4. Assert: output contains "not initialized" or "not found" or "config"
    Expected Result: Clear error message, no crash
    Evidence: .sisyphus/evidence/task-9-noinit.txt
  ```

  **Commit**: YES
  - Message: `feat(cli): implement mount, umount, and status commands`
  - Files: `cmd/quantech/cmd_mount.go`, `cmd/quantech/cmd_umount.go`, `cmd/quantech/cmd_status.go`, `cmd/quantech/commands_test.go`
  - Pre-commit: `go test ./cmd/quantech/...`

---

- [ ] 10. Logging and Error Handling

  **What to do**:
  - RED: Write `internal/logging/logger_test.go`:
    - `TestLoggerWritesToFile` — log message appears in log file
    - `TestLoggerIncludesTimestamp` — log entries have timestamps
    - `TestLoggerIncludesLevel` — log entries have INFO/WARN/ERROR levels
    - `TestLogFileRotation` — if log file exceeds 10MB, rotates
  - GREEN: Implement:
    - `internal/logging/logger.go`:
      - Use `log/slog` (Go 1.21+ structured logging) or `zerolog`
      - `InitLogger(logDir string) (*slog.Logger, error)` — create logger writing to `<root>/logs/quantech.log`
      - JSON-formatted log entries
      - Log levels: DEBUG, INFO, WARN, ERROR
      - File rotation: max 10MB, keep 3 files
    - Wire logger into all major operations:
      - Drive init, mount, unmount
      - Chunk write/read errors
      - Encryption errors
      - Capacity limit hits
      - FUSE operation errors

  **Must NOT do**:
  - Do not add remote logging or telemetry
  - Do not add verbose debug logging for every FUSE call (performance impact)
  - Do not use fmt.Println for user-facing output — use the CLI output layer

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Logging setup is straightforward plumbing
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3a (with Tasks 6, 8)
  - **Blocks**: Task 11
  - **Blocked By**: Task 1

  **References**:

  **Pattern References**:
  - `QuanTech.md:8318-8330` — Issue 11: structured logging spec — Zap/logrus, log events, rotation

  **External References**:
  - `https://pkg.go.dev/log/slog` — Go standard library structured logging (Go 1.21+)

  **WHY Each Reference Matters**:
  - Lines 8318-8330 define what events to log and the log file location
  - slog is the Go standard library approach — no external dependency needed

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `internal/logging/logger_test.go`
  - [ ] Tests cover: writes to file, timestamps, levels
  - [ ] `go test ./internal/logging/... -v` → PASS

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Log file created on init
    Tool: Bash
    Preconditions: Binary built
    Steps:
      1. rm -rf /tmp/qtest-log
      2. go run ./cmd/quantech init --root=/tmp/qtest-log --mount-point=/tmp/qm-log --encrypt=no
      3. ls /tmp/qtest-log/logs/quantech.log
      4. Assert: file exists
      5. cat /tmp/qtest-log/logs/quantech.log | head -5
      6. Assert: output contains "init" or "created" or structured JSON
    Expected Result: Log file exists with init events
    Evidence: .sisyphus/evidence/task-10-logfile.txt

  Scenario: Logger handles unwritable log directory gracefully
    Tool: Bash
    Preconditions: Go module compiled
    Steps:
      1. go test ./internal/logging/... -v -run TestLoggerHandlesUnwritableDir
      2. Assert: PASS (logger returns error or falls back to stderr)
    Expected Result: Logger doesn't panic on unwritable directory
    Evidence: .sisyphus/evidence/task-10-unwritable.txt
  ```

  **Commit**: YES
  - Message: `feat(logging): add structured logging with slog and file rotation`
  - Files: `internal/logging/logger.go`, `internal/logging/logger_test.go`, plus wiring edits across packages
  - Pre-commit: `go test ./...`

---

- [ ] 11. Load Test and End-to-End Validation

  **What to do**:
  - Implement `cmd/qloadtest/main.go`:
    - Assumes Quantech Drive is mounted at a given path
    - Creates files of various sizes: 10 × 10KB, 10 × 1MB, 5 × 10MB, 1 × 100MB
    - Writes random data, records SHA256 hashes
    - Reads back each file, computes hash, compares
    - Deletes files, verifies they're gone
    - Measures and reports throughput (MB/s write, MB/s read)
    - Exits 0 if all pass, 1 if any fail
  - Run full end-to-end test:
    - `quantech init --encrypt=yes`
    - `quantech mount`
    - Run load test
    - `quantech status` — verify used_bytes
    - `quantech umount`
    - `quantech mount` (remount) — verify data persists
    - Run load test read-only
    - `quantech umount`

  **Must NOT do**:
  - Do not benchmark against competitors
  - Do not test network scenarios
  - Do not test Windows

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Integration testing across all components, requires careful orchestration of mount/unmount cycles
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (final integration task)
  - **Parallel Group**: Sequential (after ALL other tasks)
  - **Blocks**: None (final task)
  - **Blocked By**: Tasks 7, 8, 9, 10

  **References**:

  **Pattern References**:
  - `QuanTech.md:8334-8348` — Issue 12: load test spec — file sizes, hash verification, throughput metrics
  - `QuanTech.md:8599-8609` — Load test tool spec: `cmd/qloadtest/main.go` with file creation, hash verification, throughput

  **WHY Each Reference Matters**:
  - Lines 8334-8348 define what the load test must validate
  - Lines 8599-8609 provide the exact program structure and file size distribution

  **Acceptance Criteria**:

  **Agent-Executed QA Scenarios:**

  ```
  Scenario: Full end-to-end with encryption
    Tool: interactive_bash (tmux)
    Preconditions: All packages compiled, FUSE installed
    Steps:
      1. rm -rf /tmp/qtest-e2e /tmp/qmount-e2e
      2. go run ./cmd/quantech init --root=/tmp/qtest-e2e --mount-point=/tmp/qmount-e2e --encrypt=yes <<< "e2epassword"
      3. tmux new-session -d -s qe2e "go run ./cmd/quantech mount --root=/tmp/qtest-e2e --password=e2epassword"
      4. sleep 3
      5. go run ./cmd/qloadtest --mount-path=/tmp/qmount-e2e
      6. Assert: exit code 0
      7. Assert: output contains "PASS" or "All tests passed"
      8. go run ./cmd/quantech status --root=/tmp/qtest-e2e
      9. Assert: used_bytes > 0
      10. fusermount -u /tmp/qmount-e2e || umount /tmp/qmount-e2e
      11. sleep 1
      12. tmux new-session -d -s qe2e2 "go run ./cmd/quantech mount --root=/tmp/qtest-e2e --password=e2epassword"
      13. sleep 3
      14. go run ./cmd/qloadtest --mount-path=/tmp/qmount-e2e --read-only
      15. Assert: exit code 0 (data persists)
      16. fusermount -u /tmp/qmount-e2e || umount /tmp/qmount-e2e
    Expected Result: All files write, read, persist, and verify correctly with encryption
    Evidence: .sisyphus/evidence/task-11-e2e.txt

  Scenario: Full end-to-end without encryption
    Tool: interactive_bash (tmux)
    Preconditions: All packages compiled, FUSE installed
    Steps:
      1. rm -rf /tmp/qtest-plain /tmp/qmount-plain
      2. go run ./cmd/quantech init --root=/tmp/qtest-plain --mount-point=/tmp/qmount-plain --encrypt=no
      3. tmux new-session -d -s qplain "go run ./cmd/quantech mount --root=/tmp/qtest-plain"
      4. sleep 3
      5. go run ./cmd/qloadtest --mount-path=/tmp/qmount-plain
      6. Assert: exit code 0
      7. fusermount -u /tmp/qmount-plain || umount /tmp/qmount-plain
    Expected Result: All operations work without encryption
    Evidence: .sisyphus/evidence/task-11-plain.txt

  Scenario: All unit tests pass (full suite)
    Tool: Bash
    Preconditions: All packages compiled
    Steps:
      1. go test ./... -v -count=1
      2. Assert: exit code 0
      3. Assert: output does NOT contain "FAIL"
    Expected Result: Every test across every package passes
    Evidence: .sisyphus/evidence/task-11-allpass.txt

  Scenario: Build binary for release
    Tool: Bash
    Preconditions: All tests pass
    Steps:
      1. CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -o quantech-linux-amd64 ./cmd/quantech
      2. Assert: file exists and is executable
      3. file quantech-linux-amd64
      4. Assert: output contains "ELF" or "executable"
    Expected Result: Release binary builds successfully
    Evidence: .sisyphus/evidence/task-11-build.txt
  ```

  **Commit**: YES
  - Message: `feat(validation): add load test tool and end-to-end validation`
  - Files: `cmd/qloadtest/main.go`
  - Pre-commit: `go test ./...`

---

## Commit Strategy

| After Task | Message | Key Files | Verification |
|------------|---------|-----------|--------------|
| 1 | `feat(scaffold): initialize Go module, CLI skeleton, and CI pipeline` | main.go, go.mod, ci.yml | `go build ./...` |
| 2 | `feat(config): implement drive config, init command, and directory layout` | config.go, init.go | `go test ./config/... ./drive/...` |
| 3 | `feat(crypto): implement drive key, recovery key, AES-256-GCM encryption` | keys.go, chunk_crypto.go | `go test ./crypto/...` |
| 4 | `feat(metastore): implement bbolt-based metadata store with object and chunk CRUD` | meta.go, types.go | `go test ./metastore/...` |
| 5 | `feat(chunkstore): implement chunk store with 4MiB chunks and 2-level directory sharding` | store.go | `go test ./chunkstore/...` |
| 6 | `feat(chunkstore): add encrypted chunk store wrapper with AES-256-GCM per-chunk encryption` | encrypted_store.go | `go test ./chunkstore/...` |
| 7 | `feat(fs): implement FUSE filesystem with chunked I/O, encryption, and WAL journal` | quantechfs.go, mount.go, wal.go | `go test ./...` |
| 8 | `feat(drive): add capacity enforcement with 5TB free tier limit` | capacity.go | `go test ./drive/...` |
| 9 | `feat(cli): implement mount, umount, and status commands` | cmd_mount.go, cmd_umount.go, cmd_status.go | `go test ./cmd/quantech/...` |
| 10 | `feat(logging): add structured logging with slog and file rotation` | logger.go | `go test ./internal/logging/...` |
| 11 | `feat(validation): add load test tool and end-to-end validation` | main.go (qloadtest) | Full e2e test |

---

## Success Criteria

### Verification Commands
```bash
# Build succeeds
go build ./...  # Expected: exit 0, no errors

# All tests pass
go test ./... -v  # Expected: all PASS, exit 0

# Init creates correct structure
./quantech init --root=/tmp/q --mount-point=/tmp/qm --encrypt=yes <<< "pass"
# Expected: directories created, keys.bin is 92 bytes, config.json valid

# Mount works
./quantech mount --root=/tmp/q --password=pass &
echo "test" > /tmp/qm/hello.txt
cat /tmp/qm/hello.txt  # Expected: "test"

# Status works
./quantech status --root=/tmp/q  # Expected: valid JSON with all fields

# Capacity enforcement
./quantech init --root=/tmp/qcap --mount-point=/tmp/qcapm --size=10MB --encrypt=no
./quantech mount --root=/tmp/qcap &
dd if=/dev/urandom of=/tmp/qcapm/big.bin bs=1M count=20  # Expected: ENOSPC error

# Clean unmount
./quantech umount --root=/tmp/q  # Expected: clean exit
```

### Final Checklist
- [ ] All "Must Have" items present and tested
- [ ] All "Must NOT Have" items absent (grep codebase)
- [ ] All TDD tests pass (`go test ./... -v`)
- [ ] Binary builds for Linux amd64
- [ ] Binary builds for macOS amd64
- [ ] E2E load test passes with encryption
- [ ] E2E load test passes without encryption
- [ ] Data persists across unmount/remount
- [ ] Capacity enforcement blocks writes at limit
- [ ] Status command outputs valid JSON
- [ ] Log file created and contains structured entries
- [ ] No hardcoded secrets or credentials in source
- [ ] All files under 500 lines
