---
tags:
  - compilation
aliases:
  - dynamic translation
  - run-time compilations
---
```
   linalg / vector / scf / memref
                 |
                 v
                gpu (target-agnostic)
          _______|__________________
   nvidia/      vulkan|opencl       \ amd/intel
        v             v              v
      nvgpu.        SPIR-V         amdgpu/xegpu (target-specific)
        |             |              |            
        v             v              v
      nvvm        LLVM dialect   rocdl / LLVM dialect
         \            |             /
          \___________|____________/
                      v
                   LLVM IR
```