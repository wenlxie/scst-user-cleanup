# SCST issue #279 reproduction and crash evidence

This repository holds the debug material for the `scst_user` cleanup stall
reported in [SCST issue #279](https://github.com/SCST-project/scst/issues/279).
It contains no kernel code patch. The proposed fix is a separate code-only
branch in the [SCST fork](https://github.com/wenlxie/scst).

- [Reproducer instructions](repro/README.md) and [scripts](repro/run.sh) use
  unmodified upstream SCST and `scst_local` in a disposable Lima VM. They
  create a storage target locally; no network target is needed.
- [Upstream evidence report](repro/upstream-evidence.md) separates observations
  from the source-based explanation. [Validation](repro/validation.md) records
  the source and kernel versions, failing and fixed runs, and why upstream
  PR #378 does not resolve this checked-out SGV case.
- [Captured run evidence](repro/evidence/) contains backend, process and SCST
  state, run, and kernel excerpts from both runs.
- [Historical dump analysis](crash/analysis.md), [commands](crash/crash.commands),
  and [redacted output](crash/crash.output.txt) are retained separately. They
  are not used as evidence for the unmodified upstream reproduction.

The dump and `vmlinux` are not committed here. The Lima reproduction VM has
no vmcore or matching debug `vmlinux`; the upstream report uses its captured
backend, sysfs, process, and kernel logs.
