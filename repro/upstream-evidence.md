# Issue #279: evidence from unmodified upstream SCST

## Scope

This report uses only an unmodified upstream SCST build for the failing run.
The source was commit `5908a8da46c3f98dc1524d31d761bd4dd642b151`,
which includes [PR #378](https://github.com/SCST-project/scst/pull/378).
The fixed run changed only `scst/src/dev_handlers/scst_user.c`. Both runs used
the same Ubuntu 24.04 AArch64 guest kernel, `6.8.0-142-generic`, matching
headers, and the same userspace workload. The original reporter used a
different kernel and SCST version, so this reproduces the reported failure
pattern and storage scenario rather than an identical deployment.

The unmodified run did not create a vmcore. Its evidence is the runner's
backend log, SCST sysfs command snapshots, process states, and the kernel
watchdog trace. The exact [scripts](run.sh), [procedure](README.md), and
[captured files](evidence/) are in this repository.

## Reproduction procedure

1. Check out upstream commit `5908a8da46c3f98dc1524d31d761bd4dd642b151`
   and the proposed code-only fix in separate trees. Build `scst` and
   `scst_local` for the running guest kernel, and verify `scst_user.ko`'s
   `vermagic` matches `uname -r`.
2. Load the modules in a disposable Lima VM. Run [run.sh](run.sh) with
   `SCST_SOURCE_DIR` set to the selected tree,
   `SCST_REPRO_DISPOSABLE=1`, and `SCST_REPRO_HOLD_SECONDS=30`.
3. The runner starts two file-backed `scst_user` devices with `--sgv_shared`,
   full memory reuse, nonblocking commands, and `ON_FREE_CMD_IGNORE`. It
   exposes them as two `scst_local` SG LUNs.
4. It completes two 4 KiB READ(10) commands on A, then starts a 4 KiB READ
   on B. [hold_reused_buffer.c](hold_reused_buffer.c) checks that B receives
   the same buffer address, closes A's handle while B's READ is outstanding,
   and stops the backend with B still holding the buffer.
5. The runner captures SCST's `commands` files and thread states, holds for
   30 seconds, closes B's handle, and waits for both release threads to exit.
   Preserve the logs before restarting the unfixed guest. Repeat on the same
   kernel with the fixed modules.

[README.md](README.md) gives the exact clone, Lima copy, build, module load,
run, and log collection commands.

## Direct observations

| Check | Unmodified upstream | Fixed code |
| --- | --- | --- |
| Workload precondition | [Backend log](evidence/upstream-6.8.0-142/backend.log) prints `SCST_REPRO_A_BUFFER`, `SCST_REPRO_B_HOLDS_A_BUFFER`, and `SCST_REPRO_A_HANDLE_CLOSED` with the same address, `0xea5674001000`. | [Backend log](evidence/fixed-6.8.0-142/backend.log) prints the same three markers with `0xf32268003000`. |
| A during B's outstanding READ | [State snapshot](evidence/upstream-6.8.0-142/state.log) shows an A command in state 7, `ref=1`, `sent_to_user=0`, `scst_cmd=NULL`. | A's handler is removed; its release thread remains visible during the hold. |
| B during the hold | [State snapshot](evidence/upstream-6.8.0-142/state.log) shows a B command in state 3, `sent_to_user=1`, with a non-NULL SCST command. | [State snapshot](evidence/fixed-6.8.0-142/state.log) again shows B in state 3 while it holds the buffer. |
| Cleanup worker after 30 seconds | [State snapshot](evidence/upstream-6.8.0-142/state.log) shows `scst_usr_cleanu` runnable (`R`). The [kernel excerpt](evidence/upstream-6.8.0-142/kernel-excerpt.log) reports a 26-second soft lockup, with `sgv_pool_flush -> dev_user_cleanup_thread` on the stack. | [State snapshot](evidence/fixed-6.8.0-142/state.log) shows `scst_usr_cleanu` sleeping in `msleep` during the hold. |
| After B closes | [Run log](evidence/upstream-6.8.0-142/run.log) reports `SCST_REPRO_CLEANUP_TIMEOUT`; the final [state snapshot](evidence/upstream-6.8.0-142/state.log) still shows release threads in D state. | [Run log](evidence/fixed-6.8.0-142/run.log) reports `SCST_REPRO_CLEANUP_COMPLETE`; the final [state snapshot](evidence/fixed-6.8.0-142/state.log) has no release thread and neither device's commands file remains. |

State 7 is `UCMD_STATE_ON_FREE_SKIPPED`; state 3 is
`UCMD_STATE_EXECING` in the upstream header. The A snapshot also contains a
state-`0x21` detach-session command. It is not the retained buffer command.
The kernel pointers printed by sysfs in this VM are obscured; the matching
userspace address proves the runner's reuse precondition, but does not by
itself prove the two commands reference an identical kernel SGV object.

## Mechanism inferred from upstream source

`dev_user_alloc_sg()` obtains `buf_ucmd` from `sgv_get_priv()`, which returns
the SGV's `allocator_priv`. `sgv_pool_flush()` walks the pool's recycling
lists, so it cannot free an SGV object still checked out to B. In
`dev_user_unjam_dev()`, a retained A command is counted before commands with
`sent_to_user=0` are skipped. With no ready A command,
`dev_user_get_next_cmd(A)` returns `-EAGAIN`. The old
`dev_user_process_cleanup(A)` loop then retries A without returning to the
single outer cleanup worker, delaying B's cleanup. This is the source-based
explanation for the observed runnable worker, soft lockup, and blocked release
threads; the logs are snapshots, not a record of every internal transition.

PR #378 added a second pool flush after unjamming. That handles objects
already returned to the recycling lists. The observed B READ remains active
during the hold, so that flush cannot reclaim its checked-out SGV. The
proposed fix returns A for a later retry when no ready command exists but a
command remains pending. The outer worker can process B and sleeps 100 ms
before retrying deferred devices.

## Interpretation limits

- The upstream run provides no vmcore for `crash` inspection and no direct
  kernel pointer comparison between A's and B's SGV fields.
- `scst_local` is a local SCSI transport. The test exercises storage command
  and device cleanup, without a network target.
- The identical-kernel before/after runs support the fix for this workload;
  they do not establish behavior on every kernel or SCST configuration.
