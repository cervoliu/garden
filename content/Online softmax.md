---
tags:
  - deep-learning
  - machine-learning
draft: false
---
相比传统的 softmax 计算，online softmax 改进了数值稳定性与效率。Online softmax 在 [[FlashAttention]] 中被使用。

对比两版 1D-softmax CUDA kernel 的实现（计算输入向量 x 的 softmax，输出向量 y）

Naïve softmax:

```c++
__global__ void softmax(float *x, float *y, int N) {
	extern __shared__ float buf[];
	
	// materialize exps in shared memory buf[]
	int tid = threadIdx.x;
	buf[tid] = expf(x[tid]);
	__syncthreads();
	
	float sum = 0.0f;
	for (int i = 0; i < N; ++i)
		sum += buf[i];
	y[tid] = buf[tid] / sum;
}
```

Online softmax:

```c++
__global void online_softmax(float *x, float *y, int N) {
	int tid = threadIdx.x;
	
	float m = -INFINITY; // running max
	float d = 0.0f;      // running denom
	
	for (int i = 0; i < N; ++i) {
		float v = x[i];
		float new_m = fmaxf(m, v);
		d = d * expf(m - new_m) + expf(v - new_m);
		m = new_m;
	}
	y[tid] = expf(x[tid] - m) / d;
}
```

可以看到 online softmax 实际上是在线地维护当前最大值 $m$ 以及 $d = \sum_{j} e^{x_j-m}$。