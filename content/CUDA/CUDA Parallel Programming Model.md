---
tags:
  - "#parallel"
  - "#cuda"
---
See also: [[CUDA]]

Zen of SIMT: "Single Instruction Multiple Threads"
## Kernels

chatgpt 省流：程序员可以定义 “kernel” 函数 (用 __global__ 指定) —— 当 kernel 被调用 (launch) 时，它会被 _并行执行 N 次_（由 N 个 CUDA 线程执行），而不是像普通函数那样只执行一次。每个线程都有唯一的线程 ID，可在 kernel 内通过内建变量 (threadIdx, blockIdx 等) 访问。

---

CUDA C++ extends C++ by allowing the programmer to define C++ functions, called _kernels_, that, when called, are executed N times **in parallel** by N different **_CUDA threads_**, as opposed to only once like regular C++ functions.

A kernel is defined using the **`__global__`** (call on host, execute on device) declaration specifier and the number of CUDA threads that execute that kernel for a given kernel call is specified using a new `<<<...>>>`_execution configuration_ syntax (see [Execution Configuration](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#execution-configuration)). Each thread that executes the kernel is given a unique **_thread ID_** that is accessible within the kernel through built-in variables.

As an illustration, the following sample code, using the built-in variable `threadIdx`, adds two vectors _A_ and _B_ of size _N_ and stores the result into vector _C_.

```C
// Kernel definition
__global__ void VecAdd(float* A, float* B, float* C)
{
    int i = threadIdx.x;
    C[i] = A[i] + B[i];
}

int main()
{
    ...
    // Kernel invocation with N threads
    VecAdd<<<1, N>>>(A, B, C);
    ...
}
```

Here, each of the _N_ threads that execute `VecAdd()` performs one pair-wise addition.

## Thread Hierarchy

chatgpt 省流: CUDA 的线程层次是分层的 (hierarchical)：在最顶层，是 grid → 由多个 thread-block 组成；每个 block 内包含多个 thread。这样设计允许开发者控制并行粒度 (coarse via blocks, fine via threads)。

---

For convenience, `threadIdx` is a 3-component vector, so that threads can be identified using a one-dimensional, two-dimensional, or three-dimensional _thread index_, forming a one-dimensional, two-dimensional, or three-dimensional block of threads, called a **_thread block_**. This provides a natural way to invoke computation across the elements in a domain such as a vector, matrix, or volume.

The index of a thread and its thread ID relate to each other in a straightforward way: For a one-dimensional block, they are the same; for a two-dimensional block of size _(Dx, Dy)_, the thread ID of a thread of index _(x, y)_ is _(x + y Dx)_; for a three-dimensional block of size _(Dx, Dy, Dz)_, the thread ID of a thread of index _(x, y, z)_ is _(x + y Dx + z Dx Dy)_.

As an example, the following code adds two matrices _A_ and _B_ of size _NxN_ and stores the result into matrix _C_.

```C
// Kernel definition
__global__ void MatAdd(float A[N][N], float B[N][N],
                       float C[N][N])
{
    int i = threadIdx.x;
    int j = threadIdx.y;
    C[i][j] = A[i][j] + B[i][j];
}

int main()
{
    ...
    // Kernel invocation with one block of N * N * 1 threads
    int numBlocks = 1;
    dim3 threadsPerBlock(N, N);
    MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);
    
    // threadsPerBlock <= 1024
    // numberThreads = numBlocks * threadsPerBlock;
    ...
}
```

There is a limit to the number of threads per block, since all threads of a block are expected to reside on the same SM core and must share the limited memory resources of that core. On current GPUs, a thread block may contain up to 1024 threads.

However, a kernel can be executed by multiple equally-shaped thread blocks, so that the total number of threads is equal to the number of threads per block times the number of blocks.

Blocks are organized into a one-dimensional, two-dimensional, or three-dimensional _grid_ of thread blocks as illustrated below. The number of thread blocks in a grid is usually dictated by the size of the data being processed, which typically exceeds the number of processors in the system.
![Grid of Thread Blocks](https://docs.nvidia.com/cuda/cuda-c-programming-guide/_images/grid-of-thread-blocks.png)

The number of threads per block and the number of blocks per grid specified in the `<<<...>>>` syntax can be of type `int` or `dim3`. Two-dimensional blocks or grids can be specified as in the example above.

Each block within the grid can be identified by a one-dimensional, two-dimensional, or three-dimensional unique index accessible within the kernel through the built-in `blockIdx` variable. The dimension of the thread block is accessible within the kernel through the built-in `blockDim` variable.

Extending the previous `MatAdd()` example to handle multiple blocks, the code becomes as follows.

```C
// Kernel definition
__global__ void MatAdd(float A[N][N], float B[N][N],
float C[N][N])
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    int j = blockIdx.y * blockDim.y + threadIdx.y;
    if (i < N && j < N)
        C[i][j] = A[i][j] + B[i][j];
}

int main()
{
    ...
    // Kernel invocation
    dim3 threadsPerBlock(16, 16);
    dim3 numBlocks(N / threadsPerBlock.x, N / threadsPerBlock.y);
    MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);
    ...
}
```

A thread block size of 16x16 (256 threads), although arbitrary in this case, is a common choice. The grid is created with enough blocks to have one thread per matrix element as before. For simplicity, this example assumes that the number of threads per grid in each dimension is evenly divisible by the number of threads per block in that dimension, although that need not be the case.

Thread blocks are required to execute independently. It must be possible to execute blocks in any order, in parallel or in series. This independence requirement allows thread blocks to be scheduled in any order and across any number of cores as illustrated by [Figure 3](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#scalable-programming-model-automatic-scalability), enabling programmers to write code that scales with the number of cores.

Threads within a block can cooperate by sharing data through some _shared memory_ and by synchronizing their execution to coordinate memory accesses. More precisely, one can specify synchronization points in the kernel by calling the **`__syncthreads()`** intrinsic function; `__syncthreads()` acts as a barrier at which all threads in the block must wait before any is allowed to proceed. [Shared Memory](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#shared-memory) gives an example of using shared memory. In addition to `__syncthreads()`, the [Cooperative Groups API](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#cooperative-groups) provides a rich set of thread-synchronization primitives.

For efficient cooperation, shared memory is expected to be a low-latency memory near each processor core (much like an L1 cache) and `__syncthreads()` is expected to be lightweight.


## Memory Hierarchy

chatgpt 省流: GPU 内存不是简单一个平面 (flat memory) —— 存在多级内存 (global, shared, constant, etc.)。合理利用这些不同层次 (尤其是 shared memory) 对性能非常关键。程序员通过语言扩展 (例如 __shared__ 关键字) 指定变量所处的内存区域。

---
CUDA threads may access data from multiple memory spaces during their execution as illustrated below. Each thread has private local memory. Each thread block has shared memory visible to all threads of the block and with the same lifetime as the block. Thread blocks in a thread block cluster can perform read, write, and atomics operations on each other’s shared memory. All threads have access to the same global memory.

There are also two additional read-only memory spaces accessible by all threads: the constant and texture memory spaces. The global, constant, and texture memory spaces are optimized for different memory usages (see [Device Memory Accesses](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#device-memory-accesses)). Texture memory also offers different addressing modes, as well as data filtering, for some specific data formats (see [Texture and Surface Memory](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#texture-and-surface-memory)).

The global, constant, and texture memory spaces are persistent across kernel launches by the same application.

![Memory Hierarchy](https://docs.nvidia.com/cuda/cuda-c-programming-guide/_images/memory-hierarchy.png)

## Heterogeneous Programming

chatgpt 省流: CUDA 支持在主机 (CPU) 与设备 (GPU) 之间协同：主机负责串行 / 控制逻辑 (serial / task-level work)，GPU 负责并行计算 (parallel workloads)，两者协作以发挥最大性能。

---
the CUDA programming model assumes that the CUDA threads execute on a physically separate _device_ that operates as a coprocessor to the _host_ running the C++ program. This is the case, for example, when the kernels execute on a GPU and the rest of the C++ program executes on a CPU.

The CUDA programming model also assumes that both the host and the device maintain their own separate memory spaces in DRAM, referred to as **_host memory_** and **_device memory_**, respectively. Therefore, a program manages the global, constant, and texture memory spaces visible to kernels through calls to the CUDA runtime (described in [Programming Interface](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#programming-interface)). This includes device memory allocation and deallocation as well as data transfer between host and device memory.

Unified Memory provides _managed memory_ to bridge the host and device memory spaces. Managed memory is accessible from all CPUs and GPUs in the system as a single, coherent memory image with a common address space. This capability enables oversubscription of device memory and can greatly simplify the task of porting applications by eliminating the need to explicitly mirror data on host and device. See [Unified Memory Programming](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#um-unified-memory-programming-hd) for an introduction to Unified Memory.

![Heterogeneous Programming](https://docs.nvidia.com/cuda/cuda-c-programming-guide/_images/heterogeneous-programming.png)
Note: Serial code executes on the host while parallel code executes on the device.

## Asynchronous SIMT Programming Model

chatgpt 省流: 随着硬件/CUDA 版本发展 (例如较新 GPU)，CUDA 引入异步 (asynchronous) 模型 — 允许某些操作 (memory copy, barrier, data transfer) 与计算重叠执行，从而提高并发度和资源利用率。

---

In the CUDA programming model a thread is the lowest level of abstraction for doing a computation or a memory operation. Starting with devices based on the **NVIDIA Ampere GPU Architecture**, the CUDA programming model provides acceleration to memory operations via the asynchronous programming model. The asynchronous programming model defines the behavior of asynchronous operations with respect to CUDA threads.

  
The asynchronous programming model defines the behavior of [Asynchronous Barrier](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#aw-barrier) for synchronization between CUDA threads. The model also explains and defines how [cuda::memcpy_async](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#asynchronous-data-copies) can be used to move data asynchronously from global memory while computing in the GPU.