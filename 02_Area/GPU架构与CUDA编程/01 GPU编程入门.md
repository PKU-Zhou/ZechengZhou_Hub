## 线程如何组织成线程块

假设声明了如下的block：
```cpp
dim3 threadsPerBlock(16, 16);
```
则意味着，每个block有256线程（8个warp）

并且，每个warp实际上被组织成了如下的（16，2）的形状，攻击8个warp
![[Excalidraw/Drawing 2026-11-11 02.25.57.svg]]

## 如何访问矩阵中的元素？
对于一个形状为（M,N）的矩阵A，其如果要访问其中位置为（row,col）的元素，方式为：
```cpp
	A[row * N + col]
```

即如下图所示：
![[Excalidraw/Drawing 2026-11-11 03.03.06.svg]]


## 矩阵乘法的实现
实现一：
```cpp
#include <cuda_runtime.h>
__global__ void matrix_multiplication_kernel(const float* A,
                                            const float* B,
                                            float* C,
                                            int M,
                                            int N,
                                            int K) {


    int col_idx = threadIdx.y + blockDim.y * blockIdx.y;
    int row_idx = threadIdx.x + blockDim.x * blockIdx.x;

    if((row_idx < M)&&(col_idx < K)){
        float acc = 0.0f;
        for(int i=0;i<N;i++){
            acc += A[row_idx * N + i] * B[i * K + col_idx];
        }
        C[row_idx * K + col_idx] = acc;
    }
 }

// A, B, C are device pointers (i.e. pointers to memory on the GPU)

extern "C" void solve(const float* A, const float* B, float* C, int M, int N, int K) {

    dim3 threadsPerBlock(16, 16);
    dim3 blocksPerGrid((M + threadsPerBlock.x - 1) / threadsPerBlock.x,
                       (K + threadsPerBlock.y - 1) / threadsPerBlock.y);

    matrix_multiplication_kernel<<<blocksPerGrid, threadsPerBlock>>>(A, B, C, M, N, K);
    cudaDeviceSynchronize();

}
```

实现二：
```cpp
#include <cuda_runtime.h>
__global__ void matrix_multiplication_kernel(const float* A,
                                            const float* B,
                                            float* C,
                                            int M,
                                            int N,
                                            int K) {


    int row_idx = threadIdx.y + blockDim.y * blockIdx.y;
    int col_idx = threadIdx.x + blockDim.x * blockIdx.x;

    if((row_idx < M)&&(col_idx < K)){
        float acc = 0.0f;
        for(int i=0;i<N;i++){
            acc += A[row_idx * N + i] * B[i * K + col_idx];
        }
        C[row_idx * K + col_idx] = acc;
    }
 }

// A, B, C are device pointers (i.e. pointers to memory on the GPU)

extern "C" void solve(const float* A, const float* B, float* C, int M, int N, int K) {

    dim3 threadsPerBlock(16, 16);
    dim3 blocksPerGrid((K + threadsPerBlock.x - 1) / threadsPerBlock.x,
                       (M + threadsPerBlock.y - 1) / threadsPerBlock.y);

    matrix_multiplication_kernel<<<blocksPerGrid, threadsPerBlock>>>(A, B, C, M, N, K);
    cudaDeviceSynchronize();

}
```


两种实现，仅仅是调换了 row 和 col，但是结果大相径庭。
实现一运行直接超时，实现二则能够正常完成。


### 展开分析

循环核心
```cpp
	acc += A[row_idx * N + i] * B[i * K + col_idx];
```

A的访存：row的系数为N
B的访存：col的系数为1

这意味着，前后相邻的两个thread访存的时候，获取A的元素，步长为N；获取B的元素，步长则是1

我们希望，获取A的元素，跨的次数少一些；获取B的元素，则可以便捷地连续访问。


在实现一中：
row是threadIdx.x方向，也就是lane维，是连续、快速变化的维度
col是threadIdx.y方向，也就是warp维，也就是跨行、缓慢变化的维度；

导致，大跨步地多次访问A，带来庞大的开销


在实现2中：
正好相反，
row是threadIdx.y方向，也就是warp维，也就是跨行、缓慢变化的维度；
col是threadIdx.x方向，也就是lane维，是连续、快速变化的维度

此时，系数N的影响被弱化，B则快速地连续访问

以上为定性分析，便于记忆。
根据核心任务的访存特性，优先将lane维的连续、快速访存，安排给系数较低的矩阵，可以降低访存开销。
