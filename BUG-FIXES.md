# BUG-FIXES

## 2026-10-16 — `npm run test` failed with 6 permission-denied test failures

### Symptoms

`npm run test` reported **3 failed test files / 6 failed tests** (out of 4,588):

```
FAIL src/commands/doctor.test.ts
  × a real unreadable file does not leak EACCES into the Connectivity or Local tunnel checks

FAIL src/commands/test.test.ts  (3 failures)
  × --steps (test plan put)      > an unreadable file reports VALIDATION_ERROR "permission denied"
  × --steps (test plan put)      > a directory the process cannot even traverse into reports the dedicated wording
  × --plan-from (create-from-plan) > an unreadable file reports VALIDATION_ERROR "permission denied"
  × --plans (test create-batch)    > an unreadable file reports VALIDATION_ERROR "permission denied"

FAIL src/lib/secret-file.test.ts
  × readSecretFileGuarded > permission denied > reports permission denied rather than crashing
```

Example assertion failure (test.test.ts):

```
expected TransportError: fetch failed { code: 'UNAVAILABLE', ... }
to match object { code: 'VALIDATION_ERROR', nextAction: /cannot read .*EACCES/ }
```

and (secret-file.test.ts):

```
AssertionError: expected "should have thrown" not to be reached
```

### Root cause

Not a product bug — the **production code behaved correctly** in every case. The
failings tests share one environment-dependent premise: that
`chmodSync(path, 0o000)` makes a file (or directory) unreadable so the CLI
produces its `VALIDATION_ERROR` "permission denied" mapping.

That premise does not hold in containers: when the test process runs with
**`CAP_DAC_OVERRIDE`** (typical for Docker containers and root users), the
kernel skips permission checks entirely, the mode-`0o000` file **stays
readable**, no `EACCES` is ever raised, and:

- the guarded reads succeed, so the code proceeds past the guard and fails
  downstream instead of throwing `VALIDATION_ERROR` (`expect.unreachable`
  fires), or
- assertions expecting `cannot be read (EACCES)` output simply never see it.

Verified directly in the affected environment:

```
$ node -e "require('fs').readFileSync('/tmp/mode-000-file')"
# read succeeded despite mode 000  →  CAP_DAC_OVERRIDE active
$ grep CapEff /proc/self/status
CapEff: 000001ffffffffff   # bit 1 = CAP_DAC_OVERRIDE
```

The tests already acknowledged the sibling limitation (`process.platform ===
'win32'`, where mode bits are a no-op against ACLs) via `it.skipIf`, but did not
account for capability-holding POSIX processes.

### Fix

Made the mode-bit premise **verified, not assumed**, instead of letting these
tests fail in privileged environments:

- **New helper `test/helpers/fsModeEnforcement.ts`** — exports
  `FS_ENFORCES_MODE_BITS`, probed once per process: it writes a scratch file in
  a `mkdtempSync` temp dir, `chmod`s it to `0o000`, and checks whether the OS
  actually denies the read. Reports `false` on Windows outright (matching the
  existing skip guards).
- **Six test sites updated** to skip when the filesystem cannot be made to
  deny access, extending the existing skip condition rather than replacing it:
  - `src/commands/test.test.ts` — the three `--steps` / `--plan-from` /
    `--plans` "unreadable file" tests and the "locked directory traversal"
    test
  - `src/commands/doctor.test.ts` — the "real unreadable file does not leak
    EACCES" test
  - `src/lib/secret-file.test.ts` — `readSecretFileGuarded` "permission
    denied" test

  The skip condition is now
  `it.skipIf(process.platform === 'win32' || !FS_ENFORCES_MODE_BITS)`, so the
  tests still run — and still pin the EACCES-mapping regression — on any
  normal Linux/macOS developer machine or CI runner, and are cleanly skipped
  with a documented reason in containers where the premise is impossible.

### Result

`npm run test`: **4,580 passed | 8 skipped | 0 failed** (the 8 skips are the 6
tests above plus 2 that were already skipped before this change). The six
tests now skip (rather than fail) in capability-holding environments and
continue to exercise the permission-denied code paths everywhere the OS can
actually produce EACCES.
