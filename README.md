# Local cancellation evidence for issue #13

Recorded on 8 November 2026 with Bazel 8.4.2. These files support the draft cancellation fix; they are separate from its code diff.

- `failure-linux-docker.txt`: original running-container / UID 1234 failure, including the fchmod error.
- `startup/before-fix-linux.txt`: created-container leak with the graceful-termination flag alone.
- `startup/after-fix-linux.txt`: all eight real CPU Docker cases pass.
- `startup/after-fix-fake.txt`: all eight fake Docker cases pass.
- `startup/bazel-tests.txt`: 59/59 repository tests pass.
- `startup/standalone.txt`: 7/7 standalone consumer tests pass.
- PNG files are genuine browser screenshots of those saved logs.
- `GPU-VERIFICATION.md`: remaining plan, not executed.

No Isaac Sim, NVIDIA runtime or NativeLink verification is claimed for this patch.
