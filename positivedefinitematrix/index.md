# 生成实数随机可逆矩阵与随机正定矩阵（C++）


由于最近手搓库函数，需要生成对称正定矩阵作为输入，因此上网查了下生成的思路，发现讨论者寥寥，即使有也不是c/cpp的底层实现，因此自己搓了个生成函数。生成对称正定矩阵的思路很简单，只需一个可逆矩阵乘以其转置即可。随机生成可逆矩阵的基本思路为从一个单位矩阵出发，随机进行若干次一、二、三类初等变换构造得出。

以下代码的内存组织基于列主序，关于列主序和主维度的介绍参考如下：

[Blas矩阵乘法参数详解](https://zhuanlan.zhihu.com/p/104287878)

随机可逆矩阵和随机对称正定矩阵的C++实现代码如下：

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

然而，在实际测试中，我很快发现这种写法的一个问题：它很优雅，但它生成的矩阵太**稀疏**了，因为它是从“零”开始逐渐填数字的，用这样的对称正定矩阵去做测试的话总感觉不太靠谱。因此，我继续寻找一个更工程一点的实现。随后，我在Stackoverflow里找到了一个构造对称正定矩阵的思路（Matlab代码）：

[Generate matrix symmetric and positive-definite](https://stackoverflow.com/questions/48736724/generate-matrix-symmetric-and-positive-definite)

```matlab
function A = generateSPDmatrix(n)
    A = rand(n);
    A = 0.5 * (A + A');
    A = A + (n * eye(n));
end
```

该思路可能仍然存在微小概率不是对称正定矩阵，但对于做测试来说，这么一点风险是可以承担的。为此，将上述代码改写成C++如下：

```c++
// 生成对称正定矩阵 symmetric and positive definite
void gen_dSPDmatrix(int n, double* A, int lda) {
    double *M = (double*) malloc(n * n * sizeof(double));
    gen_dmat(-1, 1, n, n, M, n); // 随机生成双精度矩阵
    for (int k = 0; k < n; k++) {
        for (int j = 0; j < n; j++) {
            for (int i = 0; i < n; i++) {
                A[i + j * lda] = 0.5 * (M[i + j * n] + M[j + i * n]) + (j == i) * n;
            }
        }
    }
    free(M);
}
```

进一步地，考虑到实际上我们需要用到的只是三角矩阵的形式。也就是说，其实我们不需要构造一个对称的矩阵，只要构造其中的上三角或下三角就可以了。进一步地，将代码改写如下：

```c++
// 生成对称正定矩阵 symmetric and positive definite
void gen_dsy(int n, double* A, int lda, char uplo = 'F') {
    if (uplo == 'U' || uplo == 'u') {
        std::default_random_engine e;
        std::uniform_real_distribution<double> u(-1, 1); // 左闭右闭区间
        // 不百分百保证正定？
        for (int j = 0; j < n; j++) {
            for (int i = 0; i <= j; i++) {
                A[i + j * lda] = u(e) + (j == i) * n;
            }
        }
    }
    else if (uplo == 'L' || uplo == 'l') {
        for (int j = 0; j < n; j++) {
            for (int i = 0; i < n - j; i++) {
                A[i + j * lda] = u(e) + (j == i) * n;
            }
        }
    }
    else { // full
       // https://stackoverflow.com/questions/48736724/generate-matrix-symmetric-and-positive-definite
       // 该方法生成的矩阵有微小概率不是正定矩阵。
       /*
        function A = generateSPDmatrix(n)
            A = rand(n);
            A = 0.5 * (A + A');
            A = A + (n * eye(n));
        end
       */
        double *M = (double*) malloc(n * n * sizeof(double));
        gen_dmat(-1, 1, n, n, M, n); // 随机生成双精度矩阵
        for (int k = 0; k < n; k++) {
            for (int j = 0; j < n; j++) {
                for (int i = 0; i < n; i++) {
                    A[i + j * lda] = 0.5 * (M[i + j * n] + M[j + i * n]) + (j == i) * n;
                }
            }
        }
        free(M);
    }
}
```


