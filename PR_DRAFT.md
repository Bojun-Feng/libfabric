# prov/opx: Guard CUDA bounce-buffer allocation and release

## Summary

With CUDA-dlopen support compiled in, CUDA initialization can fail or be excluded by `FI_HMEM` while host-memory use remains valid. OPX currently selects its bounce-buffer allocator using only `HAVE_CUDA`, so it can call `ofi_cudaHostAlloc()` through an unavailable runtime operation. Endpoint close makes the same compile-time-only choice for `ofi_cudaFreeHost()`.

Guard both calls with `ofi_hmem_is_initialized(FI_HMEM_CUDA)`, using the existing malloc/free path when CUDA is uninitialized. The initialized state is stable over a valid endpoint lifetime, so the same predicate keeps allocation and release matched without an ownership field. Initialized-CUDA allocation failures still report `FI_ENOMEM`; capacity, flags, diagnostics, and broader error unwinding are unchanged.

This is a single-commit, one-file change to `prov/opx/src/fi_opx_ep.c` (27 insertions, 21 deletions) on base `bb46b951f683e20ae7841b5a3537a7b3fad2f775`. No tests, Makefile changes, helpers, or test hooks are included. It addresses the allocation/release defect reported in ofiwg/libfabric#12736, not every possible endpoint failure.

## Validation

Candidate: `f30082117c4cb18ea216b3a9e3506f2d39cf9567`.

A private, out-of-tree harness compiles the actual allocation/release blocks and buffer-size definition, byte-verified against the committed source. It is not shipped in this contribution. Fresh runs pass initialized and uninitialized paths, allocator pairing, CUDA/host allocation failures, non-CUDA/no-HMEM configurations, NULL release, and 10,000-iteration lifetimes. The frozen baseline fails the three CUDA-uninitialized scenarios; independently removing either guard is also detected. ASan/UBSan, Valgrind, Clang compilation, and GCC analysis pass this private matrix.

A separate private reproducer links extracted blocks to genuine libfabric core initialization. The baseline crashes for missing libcudart, `FI_HMEM=system`, and an available runtime with a missing driver. The revised source linked to its rebuilt library passes 10,000 lifetimes in each case, including Valgrind. This exercises real core initialization and CUDA wrappers, but not full OPX endpoint creation.

Fresh clean GCC OPX builds with CUDA-dlopen/HMEM and without CUDA pass build, `make check`, and install using authentic CUDA and Cornelis HFI headers. Each top-level check suite contains only the existing `fi_info` check. The full endpoint translation unit passes GCC `-Wall -Werror` syntax checks in both configurations, and changed-line clang-format validation is clean. A clean full Clang 15 no-CUDA OPX build/check with `-Wall -Werror` and scoped OPX/TCP/RXM `distcheck` also pass. Existing TCP/core getinfo, EQ, CQ, MR, counter, and domain tests pass against both rebuilt libraries; capability/HMEM skips are not hardware coverage.

## Hardware boundary

No OPA100/CN5000 or GPU hardware was available. Initialized-CUDA allocation is covered with boundary doubles, not actual pinned-memory/GPU execution. Allocation-error scaffolding does not execute complete endpoint unwind. No HFI command-queue enable/close, GPU transfer, TID/RDMA, performance, or private hardware-CI result is claimed. The existing OPX simulator requires external HFI resources and is not a standalone substitute.

## Requested maintainer validation

Please confirm this does not duplicate unpublished provider work, then run private OPA100 and CN5000 CI with CUDA-dlopen OPX builds:

1. With libcudart absent, and separately with `FI_HMEM=system`, repeatedly enable/close endpoints and run host-memory tagged/message traffic. Check for CUDA dispatch, crashes, or scratch-buffer leaks.
2. With CUDA initialized, run existing GPU eager, multi-packet eager, and rendezvous coverage; verify pinned allocation/release and transfer results.
3. Inject pinned-allocation failure and a later enable failure followed by close. Check the allocation error and matching cleanup. If available, include ROCr with CUDA compiled but unavailable.

Please retain device/driver/CUDA versions, initialization logs, commands, and enable/close outcomes. These remain hardware validation requests, not completed results.
