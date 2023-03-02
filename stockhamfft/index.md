# 从Cooley-Tukey FFT到Stockham FFT


# 从Cooley-Tukey FFT到Stockham FFT

在阅读本文之前，你应当已理解了Cooley-Tukey FFT算法：
- 关于FFT运算的理解与入门，推荐观看该视频：[快速傅里叶变换(FFT)——有史以来最巧妙的算法？](https://www.bilibili.com/video/BV1za411F76U/)；
- 关于FFT的进一步理解以及蝶形运算的理解，推荐阅读《算法导论》第20章；
- 关于库利-图基算法的进一步理解可参考网上各种博客。

关于Stockham FFT算法：

- 关于Stockham FFT的比较数学的介绍，推荐该文章：[原地且自动整序的FFT算法](https://zhuanlan.zhihu.com/p/331015924)
- 本文参考文档：[OTFFT文档中的Stockham FFT介绍](http://wwwa.pikara.ne.jp/okojisan/otfft-en/stockham1.html)

库利-图基算法是经典的FFT算法，然而会导致得到的FFT序列顺序改变，因此需要位逆序的方法排布子FFT的顺序。Stockham FFT在变换的过程中调整顺序。这种变换不能通过原地操作实现，通过将计算的中间结果存储到另一片区域，下次变换的时候再存储回来，如此往复即为Stockham FFT算法（图片来自网络）：

![img](https://img1.baidu.com/it/u=2170727395,132090109&fm=253&fmt=auto&app=138&f=PNG?w=500&h=268)

下面参考OTFFT的介绍和代码，从Cooley-Tukey FFT开始，讨论如何一步步得到Stockham FFT。

## 1 Cooley-Tukey FFT算法及变形

原始Cooley-Tukey FFT算法代码如下所示：

```cpp
#include <complex>
#include <cmath>
#include <utility>

typedef std::complex<double> complex_t;

void butterfly(int n, int q, complex_t* x) // Butterfly operation
// n : sequence length
// q : block start point
// x : input/output squence
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n > 1) {
        for (int p = 0; p < m; p++) {
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            const complex_t a = x[q + p + 0];
            const complex_t b = x[q + p + m];
            x[q + p + 0] =  a + b;
            x[q + p + m] = (a - b) * wp;
        }
        butterfly(n/2, q + 0, x);
        butterfly(n/2, q + m, x);
    }
}

void bit_reverse(int n, complex_t* x) // Bitreversal operation
// n : squence length
// x : input/output sequence
{
    for (int i = 0, j = 1; j < n-1; j++) {
        for (int k = n >> 1; k > (i ^= k); k >>= 1);
        if (i < j) std::swap(x[i], x[j]);
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    butterfly(n, 0, x);
    bit_reverse(n, x);
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    butterfly(n, 0, x);
    bit_reverse(n, x);
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

上述算法需要`bit_reverse()`重排顺序。Stockham FFT的目标是去掉重排序这一步。为了得到Stockham FFT算法，让我们先把Cooley-Tukey FFT改写成不需要调用独立的`bit_reverse()`函数的形式：

```cpp
#include <complex>
#include <cmath>

typedef std::complex<double> complex_t;

void fft0(int n, int q, complex_t* x, complex_t* y)
// n : sequence length
// q : block start point
// x : input/output sequence
// y : work area
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n > 1) {
        for (int p = 0; p < m; p++) { // Butterfly operation
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            const complex_t a = x[q + p + 0];
            const complex_t b = x[q + p + m];
            y[q + p + 0] =  a + b;
            y[q + p + m] = (a - b) * wp;
        }
        fft0(n/2, q + 0, y, x);
        fft0(n/2, q + m, y, x);
        for (int p = 0; p < m; p++) { // Composition of even components and odd components
            x[q + 2*p + 0] = y[q + p + 0]; // Even components 
            x[q + 2*p + 1] = y[q + p + m]; // Odd components 
        }
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    complex_t* y = new complex_t[n]; // Allocation of work arera
    fft0(n, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

这样改写过后，蝶形运算和重排序被合并到一个函数处理，`fft()`也被调整为入口函数`fft()`和`fft0()`。在`fft0()`中，通过`x[], y[]`两个数组来回存放，在完成蝶形运算后再把重排序的结果放到一个数组中去的办法实现了重排序。

这种做法可能比原始的Cooley-Tukey FFT更慢，但这是逐步得到Stockham FFT的第一步。

## 2 递归版本的Stockham FFT

分析上面的代码，可以发现有两个位置要对数组访问，一个是蝶形运算，另一个是奇数下标和偶数下标的重新组合。如果我们能同时执行这两个操作，那么多余的访存操作就可以被消除。通过这种改动，我们就得到了递归版本的Stockham FFT：

```cpp
#include <complex>
#include <cmath>

typedef std::complex<double> complex_t;

void fft1(int n, int s, int q, complex_t* x, complex_t* y);

void fft0(int n, int s, int q, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// q : selection of even or odd
// x : input/output sequence
// y : work area
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) {}
    else {
        for (int p = 0; p < m; p++) {
            // Butterfly operation and composition of even components and odd components
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            const complex_t a = x[q + s*(p + 0)];
            const complex_t b = x[q + s*(p + m)];
            y[q + s*(2*p + 0)] =  a + b;
            y[q + s*(2*p + 1)] = (a - b) * wp;
        }
        fft1(n/2, 2*s, q + 0, y, x); // Even place FFT (y:input, x:output)
        fft1(n/2, 2*s, q + s, y, x); // Odd place FFT (y:input, x:output)
    }
}

void fft1(int n, int s, int q, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// q : selection of even or odd
// x : input sequence
// y : output sequence
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) { y[q] = x[q]; }
    else {
        for (int p = 0; p < m; p++) {
            // Butterfly Operation and composition of even components and odd components
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            const complex_t a = x[q + s*(p + 0)];
            const complex_t b = x[q + s*(p + m)];
            y[q + s*(2*p + 0)] =  a + b;
            y[q + s*(2*p + 1)] = (a - b) * wp;
        }
        fft0(n/2, 2*s, q + 0, y, x); // Even place FFT (y:input/output, x:work area)
        fft0(n/2, 2*s, q + s, y, x); // Odd place FFT (y:input/output, x:work area)
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 1, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 1, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

观察这段代码，有两个特点：

1. 利用两块内存空间，在蝶形操作的时候将`x[q + s*(p + 0)], x[q + s*(p + m)]`的相互运算结果存放到`y[q + s*(2*p + 0)], y[q + s*(2*p + 1)]`中，从而合并了蝶形运算和重排序操作；

2. 是互相调用的递归关系。关于这个代码设计，原文的解释是这样：

   > "This code has become a mutual recursion in order to minimize the accesses to an array. Stockham algorithm requires work area y for sorting. It saves sorted results once in the work area. If the saved results are written back immediately to x, the program is able to avoid becoming a mutual recursion. But, if we do so, it becomes meaningless that we have reduced the accesses to an array by executing butterfly operation and composition at the same time. For this reason, this code has become a mutual recursion."
   >
   > 百度翻译：为了最小化对数组的访问，这段代码变成了相互递归。Stockham算法需要工作区y进行排序。它将排序结果保存在工作区中一次。如果保存的结果立即写回x，则程序能够避免成为相互递归。但是，如果我们这样做，那么通过同时执行蝶形运算和组合来减少对数组的访问就变得毫无意义了。由于这个原因，这段代码变成了相互递归。

   我暂时还没有搞懂。

从而，我们可以得到Stockham FFT的数学推导式，其中$n=2^{L-h},~m=2^{-1}n,~s=2^h,~x_h(q,p)=x_h[q+sp]$：
$$
x_{h+1}(q,p)=x_h(q,p)+x_h(q,p+m) \\\\
x_{h+1}(q+s,p)=(x_h(q,p)-x_h(q,p+m))W_N^{sp} \\\\
q=0,1,...,s-1 \\\\
p=0,1,...,m-1
$$
$x_h[]$是第h步计算。当从$x_0[]$开始，得到$x_L[]$，FFT计算就完成了。

上述程序是基于频域抽取的FFT算法（DIF-FFT），基于时域抽取的FFT算法（DIT-FFT）如下所示：

```cpp
#include <complex>
#include <cmath>

typedef std::complex<double> complex_t;

void fft1(int n, int s, int q, complex_t* x, complex_t* y);

void fft0(int n, int s, int q, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// q : selection of even or odd
// x : input/output sequence
// y : work area
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) {}
    else {
        fft1(n/2, 2*s, q + 0, y, x); // Even place FFT(x:input, y:output)
        fft1(n/2, 2*s, q + s, y, x); // Odd place FFT(x:input, y:output)
        for (int p = 0; p < m; p++) {
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            const complex_t a = y[q + s*(2*p + 0)];
            const complex_t b = y[q + s*(2*p + 1)] * wp;
            x[q + s*(p + 0)] = a + b;
            x[q + s*(p + m)] = a - b;
        }
    }
}

void fft1(int n, int s, int q, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// q : selection of even or odd
// x : output sequence
// y : input sequence
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) { x[q] = y[q]; }
    else {
        fft0(n/2, 2*s, q + 0, y, x); // Even place FFT(y:input/output, x:work area)
        fft0(n/2, 2*s, q + s, y, x); // Odd place FFT(y:input/output, x:work area)
        for (int p = 0; p < m; p++) {
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            const complex_t a = y[q + s*(2*p + 0)];
            const complex_t b = y[q + s*(2*p + 1)] * wp;
            x[q + s*(p + 0)] = a + b;
            x[q + s*(p + m)] = a - b;
        }
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 1, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 1, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

$m=2^h,~t=2^{L-h},~s=2^{-1}t,~x_h(q,p)=x_h[q+tp]$，上述过程的的数学形式化推导式如下：
$$
\begin{eqnarray*}
    x_{h+1}(q,p)     & = & x_h(q,p) + x_h(q + s,p)W_N^{sp} \\\\
    x_{h+1}(q,p + m) & = & x_h(q,p) - x_h(q + s,p)W_N^{sp} \\\\
    q                & = & 0,1,\ldots,s-1 \\\\
    p                & = & 0,1,\ldots,m-1 \\\\
\end{eqnarray*}
$$

## 3 Stockham FFT的迭代优化

观察上个Stockham FFT的版本，变量q可改写为循环迭代的形式：

```cpp
#include <complex>
#include <cmath>

typedef std::complex<double> complex_t;

void fft1(int n, int s, complex_t* x, complex_t* y);

void fft0(int n, int s, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// x : input/output sequence
// y : work area
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) {}
    else {
        for (int q = 0; q < s; q++) { // Iteration for recursive-call
            for (int p = 0; p < m; p++) {
                const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
                const complex_t a = x[q + s*(p + 0)];
                const complex_t b = x[q + s*(p + m)];
                y[q + s*(2*p + 0)] =  a + b;
                y[q + s*(2*p + 1)] = (a - b) * wp;
            }
        }
        fft1(n/2, 2*s, y, x); // The number of recursive-calls is one because we tie calls with an iteration.
    }
}

void fft1(int n, int s, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// x : input sequence
// y : output sequence
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) { for (int q = 0; q < s; q++) y[q] = x[q]; }
    else {
        for (int q = 0; q < s; q++) { // Iteration for recursive-call
            for (int p = 0; p < m; p++) {
                const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
                const complex_t a = x[q + s*(p + 0)];
                const complex_t b = x[q + s*(p + m)];
                y[q + s*(2*p + 0)] =  a + b;
                y[q + s*(2*p + 1)] = (a - b) * wp;
            }
        }
        fft0(n/2, 2*s, y, x); // The number of recursive-calls is one because we tie calls with an iteration.
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 1, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    complex_t* y = new complex_t[n]; // Allocation of work area
    fft0(n, 1, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

当递归的深度变深时，步长s会变大，导致访存操作的跳跃比较大，因此可以将p和q的循环对调：

```cpp
#include <complex>
#include <cmath>

typedef std::complex<double> complex_t;

void fft1(int n, int s, complex_t* x, complex_t* y);

void fft0(int n, int s, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// x : input/output sequence
// y : work area
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) {}
    else {
        for (int p = 0; p < m; p++) {
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            for (int q = 0; q < s; q++) {
                const complex_t a = x[q + s*(p + 0)];
                const complex_t b = x[q + s*(p + m)];
                y[q + s*(2*p + 0)] =  a + b;
                y[q + s*(2*p + 1)] = (a - b) * wp;
            }
        }
        fft1(n/2, 2*s, y, x);
    }
}

void fft1(int n, int s, complex_t* x, complex_t* y)
// n : sequence length
// s : stride
// x : input sequence
// y : output sequence
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) { for (int q = 0; q < s; q++) y[q] = x[q]; }
    else {
        for (int p = 0; p < m; p++) {
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            for (int q = 0; q < s; q++) {
                const complex_t a = x[q + s*(p + 0)];
                const complex_t b = x[q + s*(p + m)];
                y[q + s*(2*p + 0)] =  a + b;
                y[q + s*(2*p + 1)] = (a - b) * wp;
            }
        }
        fft0(n/2, 2*s, y, x);
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    complex_t* y = new complex_t[n];
    fft0(n, 1, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    complex_t* y = new complex_t[n];
    fft0(n, 1, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

最后，去除掉代码的重复部分：

```cpp
#include <complex>
#include <cmath>

typedef std::complex<double> complex_t;

void fft0(int n, int s, bool eo, complex_t* x, complex_t* y)
// n  : sequence length
// s  : stride
// eo : x is output if eo == 0, y is output if eo == 1
// x  : input sequence(or output sequence if eo == 0)
// y  : work area(or output sequence if eo == 1)
{
    const int m = n/2;
    const double theta0 = 2*M_PI/n;

    if (n == 1) { if (eo) for (int q = 0; q < s; q++) y[q] = x[q]; }
    else {
        for (int p = 0; p < m; p++) {
            const complex_t wp = complex_t(cos(p*theta0), -sin(p*theta0));
            for (int q = 0; q < s; q++) {
                const complex_t a = x[q + s*(p + 0)];
                const complex_t b = x[q + s*(p + m)];
                y[q + s*(2*p + 0)] =  a + b;
                y[q + s*(2*p + 1)] = (a - b) * wp;
            }
        }
        fft0(n/2, 2*s, !eo, y, x);
    }
}

void fft(int n, complex_t* x) // Fourier transform
// n : sequence length
// x : input/output sequence
{
    complex_t* y = new complex_t[n];
    fft0(n, 1, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] /= n;
}

void ifft(int n, complex_t* x) // Inverse Fourier transform
// n : sequence length
// x : input/output sequence
{
    for (int p = 0; p < n; p++) x[p] = conj(x[p]);
    complex_t* y = new complex_t[n];
    fft0(n, 1, 0, x, y);
    delete[] y;
    for (int k = 0; k < n; k++) x[k] = conj(x[k]);
}
```

得到了Stockham FFT代码的最终版本。

