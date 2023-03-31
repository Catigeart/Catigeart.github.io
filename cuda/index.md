# CUDA学习笔记


## 网课：加速计算基础 —— CUDA C/C++笔记

### 基础内容：线程、内存管理和异步流

- 1-3 01-stream-init-solution.cu

```c
#include <stdio.h>

__global__
void initWith(float num, float *a, int N)
{

  int index = threadIdx.x + blockIdx.x * blockDim.x;
  int stride = blockDim.x * gridDim.x;

  for(int i = index; i < N; i += stride)
  {
    a[i] = num;
  }
}

__global__
void addVectorsInto(float *result, float *a, float *b, int N)
{
  int index = threadIdx.x + blockIdx.x * blockDim.x;
  int stride = blockDim.x * gridDim.x;

  for(int i = index; i < N; i += stride)
  {
    result[i] = a[i] + b[i];
  }
}

void checkElementsAre(float target, float *vector, int N)
{
  for(int i = 0; i < N; i++)
  {
    if(vector[i] != target)
    {
      printf("FAIL: vector[%d] - %0.0f does not equal %0.0f\n", i, vector[i], target);
      exit(1);
    }
  }
  printf("Success! All values calculated correctly.\n");
}

int main()
{
  int deviceId;
  int numberOfSMs;

  cudaGetDevice(&deviceId);
  cudaDeviceGetAttribute(&numberOfSMs, cudaDevAttrMultiProcessorCount, deviceId);

  const int N = 2<<24;
  size_t size = N * sizeof(float);

  float *a;
  float *b;
  float *c;

  cudaMallocManaged(&a, size);
  cudaMallocManaged(&b, size);
  cudaMallocManaged(&c, size);

  cudaMemPrefetchAsync(a, size, deviceId);
  cudaMemPrefetchAsync(b, size, deviceId);
  cudaMemPrefetchAsync(c, size, deviceId);

  size_t threadsPerBlock;
  size_t numberOfBlocks;

  threadsPerBlock = 256;
  numberOfBlocks = 32 * numberOfSMs;

  cudaError_t addVectorsErr;
  cudaError_t asyncErr;

  /*
   * Create 3 streams to run initialize the 3 data vectors in parallel.
   */

  cudaStream_t stream1, stream2, stream3;
  cudaStreamCreate(&stream1);
  cudaStreamCreate(&stream2);
  cudaStreamCreate(&stream3);

  /*
   * Give each `initWith` launch its own non-standard stream.
   */

  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream1>>>(3, a, N);
  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream2>>>(4, b, N);
  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream3>>>(0, c, N);

  addVectorsInto<<<numberOfBlocks, threadsPerBlock>>>(c, a, b, N);

  addVectorsErr = cudaGetLastError();
  if(addVectorsErr != cudaSuccess) printf("Error: %s\n", cudaGetErrorString(addVectorsErr));

  asyncErr = cudaDeviceSynchronize();
  if(asyncErr != cudaSuccess) printf("Error: %s\n", cudaGetErrorString(asyncErr));

  cudaMemPrefetchAsync(c, size, cudaCpuDeviceId);

  checkElementsAre(7, c, N);

  /*
   * Destroy streams when they are no longer needed.
   */

  cudaStreamDestroy(stream1);
  cudaStreamDestroy(stream2);
  cudaStreamDestroy(stream3);

  cudaFree(a);
  cudaFree(b);
  cudaFree(c);
}
```

### 进阶内容：2维和3维的网格和块

可以将网格和线程块定义为最多具有 3 个维度。使用多个维度定义网格和线程块绝不会对其性能造成任何影响，但这在处理具有多个维度的数据时可能非常有用，例如 2D 矩阵。如要定义二维或三维网格或线程块，可以使用 CUDA 的 `dim3` 类型，即如下所示：

```cpp
dim3 threads_per_block(16, 16, 1);
dim3 number_of_blocks(16, 16, 1);
someKernel<<<number_of_blocks, threads_per_block>>>();
```

鉴于以上示例，`someKernel` 内部的变量 `gridDim.x`、`gridDim.y`、`blockDim.x` 和 `blockDim.y` 均将等于 `16`。

- 1-1 01-matrix-multiply-2d-solution.cu：2D矩阵乘法

```c
#include <stdio.h>

#define N  64

__global__ void matrixMulGPU( int * a, int * b, int * c )
{
  int val = 0;

  int row = blockIdx.x * blockDim.x + threadIdx.x;
  int col = blockIdx.y * blockDim.y + threadIdx.y;

  if (row < N && col < N)
  {
    for ( int k = 0; k < N; ++k )
      val += a[row * N + k] * b[k * N + col];
    c[row * N + col] = val;
  }
}

void matrixMulCPU( int * a, int * b, int * c )
{
  int val = 0;

  for( int row = 0; row < N; ++row )
    for( int col = 0; col < N; ++col )
    {
      val = 0;
      for ( int k = 0; k < N; ++k )
        val += a[row * N + k] * b[k * N + col];
      c[row * N + col] = val;
    }
}

int main()
{
  int *a, *b, *c_cpu, *c_gpu;

  int size = N * N * sizeof (int); // Number of bytes of an N x N matrix

  // Allocate memory
  cudaMallocManaged (&a, size);
  cudaMallocManaged (&b, size);
  cudaMallocManaged (&c_cpu, size);
  cudaMallocManaged (&c_gpu, size);

  // Initialize memory
  for( int row = 0; row < N; ++row )
    for( int col = 0; col < N; ++col )
    {
      a[row*N + col] = row;
      b[row*N + col] = col+2;
      c_cpu[row*N + col] = 0;
      c_gpu[row*N + col] = 0;
    }

  dim3 threads_per_block (16, 16, 1); // A 16 x 16 block threads
  dim3 number_of_blocks ((N / threads_per_block.x) + 1, (N / threads_per_block.y) + 1, 1);

  matrixMulGPU <<< number_of_blocks, threads_per_block >>> ( a, b, c_gpu );

  cudaDeviceSynchronize(); // Wait for the GPU to finish before proceeding

  // Call the CPU version to check our work
  matrixMulCPU( a, b, c_cpu );

  // Compare the two answers to make sure they are equal
  bool error = false;
  for( int row = 0; row < N && !error; ++row )
    for( int col = 0; col < N && !error; ++col )
      if (c_cpu[row * N + col] != c_gpu[row * N + col])
      {
        printf("FOUND ERROR at c[%d][%d]\n", row, col);
        error = true;
        break;
      }
  if (!error)
    printf("Success!\n");

  // Free all our allocated memory
  cudaFree(a); cudaFree(b);
  cudaFree( c_cpu ); cudaFree( c_gpu );
}
```

- 1-1 01-heat-conduction-solution.cu：模拟金属银二维热传导的应用程序执行加速操作

```c
#include <stdio.h>
#include <math.h>

// Simple define to index into a 1D array from 2D space
#define I2D(num, c, r) ((r)*(num)+(c))

__global__
void step_kernel_mod(int ni, int nj, float fact, float* temp_in, float* temp_out)
{
  int i00, im10, ip10, i0m1, i0p1;
  float d2tdx2, d2tdy2;

  int j = blockIdx.x * blockDim.x + threadIdx.x;
  int i = blockIdx.y * blockDim.y + threadIdx.y;

  // loop over all points in domain (except boundary)
  if (j > 0 && i > 0 && j < nj-1 && i < ni-1) {
    // find indices into linear memory
    // for central point and neighbours
    i00 = I2D(ni, i, j);
    im10 = I2D(ni, i-1, j);
    ip10 = I2D(ni, i+1, j);
    i0m1 = I2D(ni, i, j-1);
    i0p1 = I2D(ni, i, j+1);

    // evaluate derivatives
    d2tdx2 = temp_in[im10]-2*temp_in[i00]+temp_in[ip10];
    d2tdy2 = temp_in[i0m1]-2*temp_in[i00]+temp_in[i0p1];

    // update temperatures
    temp_out[i00] = temp_in[i00]+fact*(d2tdx2 + d2tdy2);
  }
}

void step_kernel_ref(int ni, int nj, float fact, float* temp_in, float* temp_out)
{
  int i00, im10, ip10, i0m1, i0p1;
  float d2tdx2, d2tdy2;


  // loop over all points in domain (except boundary)
  for ( int j=1; j < nj-1; j++ ) {
    for ( int i=1; i < ni-1; i++ ) {
      // find indices into linear memory
      // for central point and neighbours
      i00 = I2D(ni, i, j);
      im10 = I2D(ni, i-1, j);
      ip10 = I2D(ni, i+1, j);
      i0m1 = I2D(ni, i, j-1);
      i0p1 = I2D(ni, i, j+1);

      // evaluate derivatives
      d2tdx2 = temp_in[im10]-2*temp_in[i00]+temp_in[ip10];
      d2tdy2 = temp_in[i0m1]-2*temp_in[i00]+temp_in[i0p1];

      // update temperatures
      temp_out[i00] = temp_in[i00]+fact*(d2tdx2 + d2tdy2);
    }
  }
}

int main()
{
  int istep;
  int nstep = 200; // number of time steps

  // Specify our 2D dimensions
  const int ni = 200;
  const int nj = 100;
  float tfac = 8.418e-5; // thermal diffusivity of silver

  float *temp1_ref, *temp2_ref, *temp1, *temp2, *temp_tmp;

  const int size = ni * nj * sizeof(float);

  temp1_ref = (float*)malloc(size);
  temp2_ref = (float*)malloc(size);
  cudaMallocManaged(&temp1, size);
  cudaMallocManaged(&temp2, size);

  // Initialize with random data
  for( int i = 0; i < ni*nj; ++i) {
    temp1_ref[i] = temp2_ref[i] = temp1[i] = temp2[i] = (float)rand()/(float)(RAND_MAX/100.0f);
  }

  // Execute the CPU-only reference version
  for (istep=0; istep < nstep; istep++) {
    step_kernel_ref(ni, nj, tfac, temp1_ref, temp2_ref);

    // swap the temperature pointers
    temp_tmp = temp1_ref;
    temp1_ref = temp2_ref;
    temp2_ref= temp_tmp;
  }

  dim3 tblocks(32, 16, 1);
  dim3 grid((nj/tblocks.x)+1, (ni/tblocks.y)+1, 1);
  cudaError_t ierrSync, ierrAsync;

  // Execute the modified version using same data
  for (istep=0; istep < nstep; istep++) {
    step_kernel_mod<<< grid, tblocks >>>(ni, nj, tfac, temp1, temp2);

    ierrSync = cudaGetLastError();
    ierrAsync = cudaDeviceSynchronize(); // Wait for the GPU to finish
    if (ierrSync != cudaSuccess) { printf("Sync error: %s\n", cudaGetErrorString(ierrSync)); }
    if (ierrAsync != cudaSuccess) { printf("Async error: %s\n", cudaGetErrorString(ierrAsync)); }

    // swap the temperature pointers
    temp_tmp = temp1;
    temp1 = temp2;
    temp2= temp_tmp;
  }

  float maxError = 0;
  // Output should always be stored in the temp1 and temp1_ref at this point
  for( int i = 0; i < ni*nj; ++i ) {
    if (abs(temp1[i]-temp1_ref[i]) > maxError) { maxError = abs(temp1[i]-temp1_ref[i]); }
  }

  // Check and see if our maxError is greater than an error bound
  if (maxError > 0.0005f)
    printf("Problem! The Max Error of %.5f is NOT within acceptable bounds.\n", maxError);
  else
    printf("The Max Error of %.5f is within acceptable bounds.\n", maxError);

  free( temp1_ref );
  free( temp2_ref );
  cudaFree( temp1 );
  cudaFree( temp2 );

  return 0;
}
```



- 1-3 01-stream-init-solution.cu

```c
#include <stdio.h>

__global__
void initWith(float num, float *a, int N)
{

  int index = threadIdx.x + blockIdx.x * blockDim.x;
  int stride = blockDim.x * gridDim.x;

  for(int i = index; i < N; i += stride)
  {
    a[i] = num;
  }
}

__global__
void addVectorsInto(float *result, float *a, float *b, int N)
{
  int index = threadIdx.x + blockIdx.x * blockDim.x;
  int stride = blockDim.x * gridDim.x;

  for(int i = index; i < N; i += stride)
  {
    result[i] = a[i] + b[i];
  }
}

void checkElementsAre(float target, float *vector, int N)
{
  for(int i = 0; i < N; i++)
  {
    if(vector[i] != target)
    {
      printf("FAIL: vector[%d] - %0.0f does not equal %0.0f\n", i, vector[i], target);
      exit(1);
    }
  }
  printf("Success! All values calculated correctly.\n");
}

int main()
{
  int deviceId;
  int numberOfSMs;

  cudaGetDevice(&deviceId);
  cudaDeviceGetAttribute(&numberOfSMs, cudaDevAttrMultiProcessorCount, deviceId);

  const int N = 2<<24;
  size_t size = N * sizeof(float);

  float *a;
  float *b;
  float *c;

  cudaMallocManaged(&a, size);
  cudaMallocManaged(&b, size);
  cudaMallocManaged(&c, size);

  cudaMemPrefetchAsync(a, size, deviceId);
  cudaMemPrefetchAsync(b, size, deviceId);
  cudaMemPrefetchAsync(c, size, deviceId);

  size_t threadsPerBlock;
  size_t numberOfBlocks;

  threadsPerBlock = 256;
  numberOfBlocks = 32 * numberOfSMs;

  cudaError_t addVectorsErr;
  cudaError_t asyncErr;

  /*
   * Create 3 streams to run initialize the 3 data vectors in parallel.
   */

  cudaStream_t stream1, stream2, stream3;
  cudaStreamCreate(&stream1);
  cudaStreamCreate(&stream2);
  cudaStreamCreate(&stream3);

  /*
   * Give each `initWith` launch its own non-standard stream.
   */

  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream1>>>(3, a, N);
  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream2>>>(4, b, N);
  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream3>>>(0, c, N);

  addVectorsInto<<<numberOfBlocks, threadsPerBlock>>>(c, a, b, N);

  addVectorsErr = cudaGetLastError();
  if(addVectorsErr != cudaSuccess) printf("Error: %s\n", cudaGetErrorString(addVectorsErr));

  asyncErr = cudaDeviceSynchronize();
  if(asyncErr != cudaSuccess) printf("Error: %s\n", cudaGetErrorString(asyncErr));

  cudaMemPrefetchAsync(c, size, cudaCpuDeviceId);

  checkElementsAre(7, c, N);

  /*
   * Destroy streams when they are no longer needed.
   */

  cudaStreamDestroy(stream1);
  cudaStreamDestroy(stream2);
  cudaStreamDestroy(stream3);

  cudaFree(a);
  cudaFree(b);
  cudaFree(c);
}
```

### 进阶内容：手动内存分配和复制

尽管 `cudaMallocManaged` 和 `cudaMemPrefetchAsync` 函数性能出众并能大幅简化内存迁移，但有时也有必要使用更多手动内存分配方法。这在已知只需在设备或主机上访问数据时尤其如此，并且因免于进行自动按需迁移而能够收回数据迁移成本。

此外，通过手动内存管理，您可以使用非默认流同时开展数据传输与计算工作。在本节中，您将学习一些基本的手动内存分配和拷贝技术，之后会延伸应用这些技术以同时开展数据拷贝与计算工作。

以下是一些用于手动内存管理的 CUDA 命令：

- `cudaMalloc` 命令将直接为处于活动状态的 GPU 分配内存。这可防止出现所有 GPU 分页错误，而代价是主机代码将无法访问该命令返回的指针。
- `cudaMallocHost` 命令将直接为 CPU 分配内存。该命令可“固定”内存(pinned memory)或“页锁定”内存(page-locked memory)，此举允许将内存异步拷贝至 GPU 或从 GPU 异步拷贝至内存。固定内存过多则会干扰 CPU 性能，因此请勿无端使用该命令。释放固定内存时应使用 `cudaFreeHost` 命令。
- 无论是从主机到设备还是从设备到主机，`cudaMemcpy` 命令均可拷贝（而非传输）内存。

除了`cudaMemcpy`之外，还有`cudaMemcpyAsync`，只要固定了主机内存，它就可以从主机到设备或从设备到主机异步复制内存，这可以通过使用`cudaMallocHost`分配它来完成。

与核函数的执行类似，`cudaMemcpyAsync`在默认情况下仅相对于主机是异步的。默认情况下，它在默认流中执行，因此对于GPU上发生的其他CUDA操作而言，它是阻塞操作。但是，`cudaMemcpyAsync`函数将非默认流作为可选的第5个参数。通过向其传递非默认流，可以将内存传输与其他非默认流中发生的其他CUDA操作并发。

一种常见且有用的模式是结合使用固定主机内存，非默认流中的异步内存副本和非默认流中的核函数执行，以使内存传输与核函数的执行重叠。

- 01-overlap-xfer-solution.cu

```c
#include <stdio.h>

__global__
void initWith(float num, float *a, int N)
{

  int index = threadIdx.x + blockIdx.x * blockDim.x;
  int stride = blockDim.x * gridDim.x;

  for(int i = index; i < N; i += stride)
  {
    a[i] = num;
  }
}

__global__
void addVectorsInto(float *result, float *a, float *b, int N)
{
  int index = threadIdx.x + blockIdx.x * blockDim.x;
  int stride = blockDim.x * gridDim.x;

  for(int i = index; i < N; i += stride)
  {
    result[i] = a[i] + b[i];
  }
}

void checkElementsAre(float target, float *vector, int N)
{
  for(int i = 0; i < N; i++)
  {
    if(vector[i] != target)
    {
      printf("FAIL: vector[%d] - %0.0f does not equal %0.0f\n", i, vector[i], target);
      exit(1);
    }
  }
  printf("Success! All values calculated correctly.\n");
}

int main()
{
  int deviceId;
  int numberOfSMs;

  cudaGetDevice(&deviceId);
  cudaDeviceGetAttribute(&numberOfSMs, cudaDevAttrMultiProcessorCount, deviceId);

  const int N = 2<<24;
  size_t size = N * sizeof(float);

  float *a;
  float *b;
  float *c;
  float *h_c;

  cudaMalloc(&a, size);
  cudaMalloc(&b, size);
  cudaMalloc(&c, size);
  cudaMallocHost(&h_c, size);

  size_t threadsPerBlock;
  size_t numberOfBlocks;

  threadsPerBlock = 256;
  numberOfBlocks = 32 * numberOfSMs;

  cudaError_t addVectorsErr;
  cudaError_t asyncErr;

  /*
   * Create 3 streams to run initialize the 3 data vectors in parallel.
   */

  cudaStream_t stream1, stream2, stream3;
  cudaStreamCreate(&stream1);
  cudaStreamCreate(&stream2);
  cudaStreamCreate(&stream3);

  /*
   * Give each `initWith` launch its own non-standard stream.
   */

  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream1>>>(3, a, N);
  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream2>>>(4, b, N);
  initWith<<<numberOfBlocks, threadsPerBlock, 0, stream3>>>(0, c, N);

  for (int i = 0; i < 4; ++i)
  {
    cudaStream_t stream;
    cudaStreamCreate(&stream);

    addVectorsInto<<<numberOfBlocks/4, threadsPerBlock, 0, stream>>>(&c[i*N/4], &a[i*N/4], &b[i*N/4], N/4);
    cudaMemcpyAsync(&h_c[i*N/4], &c[i*N/4], size/4, cudaMemcpyDeviceToHost, stream);
    cudaStreamDestroy(stream);
  }

  addVectorsErr = cudaGetLastError();
  if(addVectorsErr != cudaSuccess) printf("Error: %s\n", cudaGetErrorString(addVectorsErr));

  asyncErr = cudaDeviceSynchronize();
  if(asyncErr != cudaSuccess) printf("Error: %s\n", cudaGetErrorString(asyncErr));

  checkElementsAre(7, h_c, N);

  /*
   * Destroy streams when they are no longer needed.
   */

  cudaStreamDestroy(stream1);
  cudaStreamDestroy(stream2);
  cudaStreamDestroy(stream3);

  cudaFree(a);
  cudaFree(b);
  cudaFree(c);
  cudaFreeHost(h_c);
}
```

## PPT笔记

### 编程模型

变量类型限定词：

|                  变量声明                  | 内存 |  域  | 生命周期 |
| :----------------------------------------: | :--: | :--: | :------: |
|    `__device__ __local__` int LocalVar;    | 本地 | 线程 |   线程   |
|   `__device__ __shared__` int SharedVar;   | 共享 |  块  |    块    |
|        `__device__` int GlobalVar;         | 全局 | 网格 |   应用   |
| `__device__ __constant__` int ConstantVar; | 常量 | 网格 |   应用   |

`__local__`,`__shared__`,`__constant__`的`__device__`可省去。

函数声明：

|              | 执行位置 | 调用位置 |
| :----------: | :------: | :------: |
| `__device__` |   设备   |   设备   |
| `__global__` |   设备   |   主机   |
|  `__host__`  |   主机   |   主机   |

- `__global__` defines a kernel function 
  - Must return void 

- `__device__` and `__host__` can be used together

`cudaMemcpyAsync()`：异步内存复制

`void __syncthreads()`：线程同步

时间函数：

```c
cudaEventCreate(&start);
cudaEventCreate(&stop);
cudaEventRecord(start,0);
// do sth.
cudaEventRecord(stop,0);
cudaEventSynchronize(stop);
cudaEventElapsedTime(&elapsed
Time, start, stop);
cudaEventDestroy(start);
cudaEventDestroy(stop);
```

### 内存

[CUDA编程——GPU架构，由sp，sm，thread，block，grid，warp说起](https://blog.csdn.net/u013171226/article/details/118365934)

[CUDA学习笔记（九）内存](https://blog.csdn.net/qq_36387683/article/details/81125453)

利用内存结构：数据切片

**Typical Structure of a CUDA Program：**

- Global variables declaration 

  - `__host__` 

  - `__device__`... `__global__`, `__constant__`, `__texture__` 

- Function prototypes 

  - `__global__ void kernelOne (…)` 

  - float handyFunction (…) 

- Main () 

  - allocate memory space on the device – cudaMalloc (&d_GlblVarPtr,  bytes ) 

  - transfer data from host to device 
    - cudaMemCpy (d_GlblVarPtr, h_Gl…) 

  - execution configuration setup 

  - kernel call – kernelOne<<>>( args… ); 

  - transfer results from device to host 
    - cudaMemCpy (h_GlblVarPtr,…) 

  - optional: compare against golden (host computed) solution

- Kernel – void kernelOne (type args,…) 

  - variables declaration - `__local__`, `__shared__` 

  - automatic variables transparently assigned to registers or local  memory 

  - Syncthreads ()… 

- Other functions 
  - float handyFunction (int inVar…);

### 线程

warp：调度线程束的基本单位，一束线程执行相同操作。

[CUDA ---- Branch Divergence and Unrolling Loop](https://www.cnblogs.com/1024incn/p/4548056.html)

### 调优

- 通用优化策略
- 内存优化策略：[CUDA bank 及bank conflict](https://blog.csdn.net/weixin_42730667/article/details/106171382)

- 指令优化策略

- 其他策略
