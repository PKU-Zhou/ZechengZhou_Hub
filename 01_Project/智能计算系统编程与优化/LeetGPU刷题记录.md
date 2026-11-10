## 1104
```cpp
#include <cuda_runtime.h>

__global__ void vector_add(const float* A, const float* B, float* C, int N) {
    int tid = threadIdx.x + blockDim.x * blockIdx.x;
    if(tid < N){
        C[tid] = A[tid] + B[tid];
    }
}

  

// A, B, C are device pointers (i.e. pointers to memory on the GPU)
extern "C" void solve(const float* A, const float* B, float* C, int N) {
    int threadsPerBlock = 256;
    int blocksPerGrid = (N + threadsPerBlock - 1) / threadsPerBlock;
    vector_add<<<blocksPerGrid, threadsPerBlock>>>(A, B, C, N);
    cudaDeviceSynchronize();
}
```

vector_add是线程函数，实现时要牢记，针对每个线程的功能进行设计。
调用时，借助<<<blocksPerGrid, threadsPerBlock>>>，告诉GPU如何安排这些线程。


## 1107

```cpp
#include <cuda_runtime.h>

  

__global__ void reduction_sum_kernel(const float* input, float* output, int N){

    // 用共享内存缓存中间值，避免频繁访全局内存

    extern __shared__ float sdata[];

  

    int local_thread_id  = threadIdx.x;

    int global_thread_id = threadIdx.x + blockDim.x * blockIdx.x;

  

    // 第一步：把全局数组的值读到共享内存

    sdata[local_thread_id] = global_thread_id<N?input[global_thread_id]:0.0f;

    __syncthreads();

    // 树状归约：步长从N/2开始逐次折半

    for(int step = blockDim.x / 2; step > 0; step >>= 1){

        if(local_thread_id < step){

            sdata[local_thread_id] += sdata[local_thread_id + step];

        }

        __syncthreads(); // 每步做完块内同步，保证所有线程看到最新值

    }

  

    // 最后由第0号线程把结果写回输出

    if(local_thread_id == 0){

        atomicAdd(output, sdata[0]);

    }

}

  
  

// input, output are device pointers

extern "C" void solve(const float* input, float* output, int N) {

    int threadPerBlock = 256;

    int blockPerGrid = (N + threadPerBlock -1)/threadPerBlock;

    size_t smem = threadPerBlock * sizeof(float);   // ← 必须传，单位是字节

  

    reduction_sum_kernel<<<blockPerGrid,threadPerBlock,smem>>>(input, output, N);

    cudaDeviceSynchronize();

}
```

 - *extern __shared__ float sdata[];*  用于声明当前block的专有的共享内存空间；
 - *size_t smem = threadPerBlock * sizeof(float);* 使用kernel前，确定所需的共享内存规模；
 - *atomicAdd(output, sdata[0]);* 将每个block的结果相加；
 - 总体思路：分block进行tree-reduction