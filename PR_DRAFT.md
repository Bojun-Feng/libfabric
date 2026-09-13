# prov/opx: Guard CUDA bounce-buffer allocation and release

With CUDA-dlopen support compiled in, OPX calls `ofi_cudaHostAlloc()` and `ofi_cudaFreeHost()` whenever `HAVE_CUDA` is set, even if the CUDA HMEM interface failed to initialize. This can dispatch through unavailable CUDA runtime operations.

Check `ofi_hmem_is_initialized(FI_HMEM_CUDA)` before using the CUDA allocation and release functions. Fall back to the existing `malloc()` and `free()` paths when CUDA is not initialized. Using the same condition for allocation and release keeps the allocator pair matched.

Fixes #12736

Test description: Built OPX with CUDA-dlopen/HMEM and without CUDA. An out-of-tree harness covering the production allocation and release blocks reproduced the CUDA-uninitialized baseline failure and passed initialized, uninitialized, allocation-failure, non-CUDA, and repeated allocation/release cases after the change. No OPX or GPU hardware testing was performed.
