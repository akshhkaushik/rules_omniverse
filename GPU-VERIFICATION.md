# GPU verification handoff — NOT RUN

This plan requires an authorized Linux NVIDIA worker. The Mac/CPU results do
not substitute for it. No NVIDIA software was installed or downloaded for the
local work. Use the repository's existing SDK/image licensing and setup
instructions; the operator supplies the image and accepts its applicable terms.

## Preconditions

1. Review the patch in a separate checkout. Record its source revision or
   patch checksum, Bazel version, Docker version, GPU model and driver.
2. Follow `examples/isaac_sim/README.md`. Match the platform's GPU model and
   driver, and the target's GPU model, to the actual worker. Keep the image
   pinned by digest. Record any test configuration changes explicitly.
3. Confirm the authorized pinned image is present and Docker/NVIDIA runtime
   access works. Use a dedicated test worker with no other builds using that
   image, so container identity and ownership observations are unambiguous.
4. Store evidence on a native Linux filesystem. Never include credentials or
   private Bazel rc files in the evidence bundle.

## A. Successful local run

From that checkout, choose the actual configured execution platform. The
repository's reference platform name below is valid only when its GPU/driver
properties match the worker:

```bash
verify_root="$(mktemp -d /var/tmp/isaac-cancellation-XXXXXX)"
gpu_platform=//examples/isaac_sim:isaac_sim_l40s_linux_x86_64
gpu_target=//examples/isaac_sim:scene_based_sdg

bazel_local=(bazel "--output_base=$verify_root/bazel" --ignore_all_rc_files)
local_flags=("--extra_execution_platforms=$gpu_platform" --remote_executor= --remote_cache=)

nvidia-smi > "$verify_root/nvidia-smi.txt"
docker version > "$verify_root/docker-version.txt"
bazel --version > "$verify_root/bazel-version.txt"
"${bazel_local[@]}" build "${local_flags[@]}" "$gpu_target" \
  > "$verify_root/local-success.log" 2>&1
```

Inspect the real output and its UID/GID. A zero exit alone is insufficient;
confirm expected generated frames and complete output. This first run is a
prerequisite, not cancellation evidence.

## B. Cancel after output begins

1. Run `"${bazel_local[@]}" clean` to prevent the previous local result from
   skipping execution. Remote cache/executor flags remain empty.
2. Run the same local build again, saving a separate log. Once a first frame
   appears, interrupt the Bazel client once with Ctrl-C. Record exit status
   immediately. Use a declared larger frame count if the run finishes too
   quickly; record the change rather than assuming cancellation occurred.
3. Confirm that the specific test container is gone, not merely stopped.
   Compare container IDs recorded before and during the run. Do not remove
   another action's container or use a broad image filter for cleanup.
4. Inspect all nested output ownership. Repeat the exact build and confirm
   complete frames, then run clean. Both must work as the worker user without
   sudo or manual ownership repair.

Keep the cancelled-run log, ownership listing, container-state evidence,
rebuild log, clean log and exact test configuration.

## C. Cancellation caused by a sibling failure

Create a temporary test-only Bazel package with this genrule. Substitute the
absolute path of a marker inside `verify_root` before using the snippet:

```starlark
genrule(
    name = "fail_when_requested",
    outs = ["never.txt"],
    cmd = "while [ ! -f /ABSOLUTE_VERIFY_ROOT/fail_now ]; do sleep 0.05; done; exit 1",
    tags = ["manual", "local", "no-sandbox"],
)
```

Clean the isolated output base and build the real Isaac Sim target together
with this genrule using `--jobs=2 --local_resources=cpu=2`, without
`--keep_going`. Once a real frame appears, create the marker from a second
terminal. The genrule must fail and Bazel must cancel the still-running GPU
action. Repeat the container, ownership, complete rebuild and clean checks
from section B. Remove only the temporary package created for this test.

The local CPU harness cannot perform this GPU verification: its adapter
deliberately replaces the GPU selection and Isaac Sim entrypoint.

## D. NativeLink compatibility

Use the existing example's private rc instructions with the actual authorized
execution and CAS endpoints. Keep local fallback disabled. Record a real
remote execution and valid output, then repeat from another fresh Bazel output
base and inspect the execution log for a remote cache hit. Change a declared
input and confirm remote execution again. Match target/host platform settings
as documented when the client is macOS.

This establishes GPU remote compatibility after the patch; it does not turn
the local cancellation tests into remote-cancellation tests. The issue reports
the failure for local builds.

## Decision

Only report these checks as passing after the actual logs have been reviewed.
Any leftover container, incorrect ownership, incomplete output, failed
rebuild/clean or remote regression keeps the PR on hold. In-flight Docker
creation before an ID is known and forced-termination recovery still need an
explicit scope decision; the current local matrix does not establish them.
