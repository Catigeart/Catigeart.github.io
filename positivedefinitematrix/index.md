# 生成实数随机可逆矩阵与随机正定矩阵（C++）


由于最近手搓库函数，需要生成正定矩阵作为输入，因此上网查了下生成的思路，发现讨论者寥寥，即使有也不是c/cpp的底层实现，因此自己搓了个生成函数。生成正定矩阵的思路很简单，只需一个可逆矩阵乘以其转置即可。随机生成可逆矩阵的基本思路为从一个单位矩阵出发，随机进行若干次一、二、三类初等变换构造得出。

以下代码的内存组织基于列主序，关于列主序和主维度的介绍参考如下：

[Blas矩阵乘法参数详解](https://zhuanlan.zhihu.com/p/104287878)

C++实现代码如下，计算效率未经优化，仅供参考。

```cpp
#include <algorithm>
#include <cstdlib>
#include <ctime>

void gen_d_full_rank_matrix(int n, double* A, int lda) {
    srand((unsigned int)time(nullptr));
    // 随机生成标准正交矩阵（相当于单位阵做1类初等变换）
    int j_arr[n];
    for (int i = 0; i < n; i++) {
        j_arr[i] = i;
    }
    std::random_shuffle(j_arr, j_arr + n);
    for (int i = 0; i < n; i++) {
        A[i + j_arr[i] * lda] = 1;
    }
    // 随机进行n次3类初等变换
    // 2类初等变换可以视为3类初等变换加回本行，因此忽略
    for (int p = 0; p < n; p++) {
        int src_row = 0 + rand() % n;
        int dst_row = 0 + rand() % n;
        double k = (double) (-50 + rand() % 100) / 10.0;
        for (int j = 0; j < n; j++) {
            A[dst_row + j * lda] += k * A[src_row + j * lda];
        }
    }
}

void gen_d_positive_definite_matrix(int n, double* A, int lda) {
    // 可逆矩阵的转置相乘必为正定矩阵
    double *M = (double*) malloc(n * n * sizeof(double));
    gen_d_full_rank_matrix(n, M, n);
    for (int j = 0; j < n; j++) {
        for (int i = 0; i < n; i++) {
            for (int k = 0; k < n; k++) {
                A[i + j * lda] = M[i + k * lda] * M[j + k * lda]; // M * M^T
            }
        }
    }
}
```


