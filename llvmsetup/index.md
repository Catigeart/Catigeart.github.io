# Linux系统无痛编译安装LLVM简明指南




# 1 编译与预编译版本选择

如果对LLVM没有特别需求，只是当作一般编译器使用，安装预编译版也足以应付场面；如果需要对LLVM作个性化定制，或基于LLVM开发，或学习LLVM源码，那手动编译LLVM工程会是更好的选择。
# 2 操作系统选择
此处选择Linux系统。
# 3 编译安装指南
LLVM是一个仍在快速发展的开源项目，已经成为又一个版本怪物，中文互联网上鱼龙混杂的各种LLVM编译步骤指南大多已经过时，或隐藏着各种未知的坑，即使是官方不同出处的文档，也可能存在各自不规整统一之处。
因此，LLVM编译指南靠谱程度依次为：GitHub仓库的Readme.md$>$官方各种文档$>$互联网其他编译指南。而我的建议是——

**直接从GitHub仓库克隆工程到本地进行编译！**

**直接从GitHub仓库克隆工程到本地进行编译！**

**直接从GitHub仓库克隆工程到本地进行编译！**

为什么直接从GitHub仓库clone？一方面，它足够完整，便于利用cmake指令灵活定制并集中编译；另一方面，GitHub上的Readme.md提供的安装指南一定是紧跟版本的，无过时之虞。**如果不需要GitHub上LLVM的最新版本，只需要在把工程clone到本地以后再手动回退到指定tag的版本即可。**

GitHub回退到指定版本（指定Tag）的网上教程很多，此处不再赘述。注意我们只需要硬回退本地版本，不需要回退以后提交到远程仓库的步骤。

综上所述，无痛编译安装LLVM的步骤和方法是：**从GitHub仓库克隆工程到本地，将本地工程回退到指定版本，再按照对应版本的readme.md教程编译LLVM工程。** 
# 4 Linux下编译指定版本LLVM实例
**编译平台**：Ubuntu 20.04.2 LTS
**LLVM版本**：10.0.1

LLVM的GitHub仓库链接：<https://github.com/llvm/llvm-project>

注：编译过程中需要的git, cmake等组件从略，请根据官方指示或编译中的实际踩坑安装所需依赖。
## 4.1 仓库下载
国内下载GitHub仓库往往有速度太慢的问题，互联网对此有相当多的解决方案。此处采用的是CSDN博客<https://blog.csdn.net/haejwcalcv/article/details/108028245>中提到的办法，具体地说，只需**把github.com换成github.com.cnpmjs.org即可**。
克隆仓库命令如下：
```sh
git clone https://github.com.cnpmjs.org/llvm/llvm-project.git
```
注意，如果是在win平台下克隆仓库，还应当禁止代码换行符的自动转换，加上`--config core.autocrlf=false`：
```sh
git clone --config core.autocrlf=false https://github.com.cnpmjs.org/llvm/llvm-project.git
```
## 4.2 回退版本 
因为GitHub仓库维护的是最新版本，因此我们需要用git将版本回退到指定tag。首先，我们需要先查询llvm的tag列表：
```sh
git tag
```
控制台输出如下：
```
llvmorg-1.0.0
llvmorg-1.1.0
llvmorg-1.2.0
llvmorg-1.3.0
llvmorg-1.4.0
llvmorg-1.5.0
llvmorg-1.6.0
llvmorg-1.9.0
llvmorg-10-init
llvmorg-10.0.0
llvmorg-10.0.0-rc1
llvmorg-10.0.0-rc2
llvmorg-10.0.0-rc3
llvmorg-10.0.0-rc4
llvmorg-10.0.0-rc5
llvmorg-10.0.0-rc6
llvmorg-10.0.1
llvmorg-10.0.1-rc1
llvmorg-10.0.1-rc2
llvmorg-10.0.1-rc3
llvmorg-10.0.1-rc4
llvmorg-11-init
llvmorg-11.0.0
:
```
查看控制台输出可得，llvm 10.0.1对应的tag版本是llvmorg-10.0.1。查询该tag版本的信息指令如下：
```sh
git show llvmorg-10.0.1
```
控制台输出如下：
```
tag llvmorg-10.0.1
Tagger: Tom Stellard <tstellar@redhat.com>
Date:   Mon Jul 20 16:55:03 2020 -0700

Tag 10.0.1

commit ef32c611aa214dea855364efd7ba451ec5ec3f74 (HEAD -> main, tag: llvmorg-10.0.1-rc4, tag: llvmorg-10.0.1, origin/release/10.x)
Author: Hubert Tong <hubert.reinterpretcast@gmail.com>
Date:   Thu Apr 30 22:18:54 2020 -0400

    [tests] Revert unhelpful change from d73eed42d1dc
    
    (cherry picked from commit 0e8608b3c38886c224d252c6b126c804645b7761)

diff --git a/llvm/test/CodeGen/X86/asm-modifier.ll b/llvm/test/CodeGen/X86/asm-modifier.ll
index 0b8b240f7f7d..47b185a15766 100644
--- a/llvm/test/CodeGen/X86/asm-modifier.ll
+++ b/llvm/test/CodeGen/X86/asm-modifier.ll
@@ -1,4 +1,4 @@
-; RUN: llc < %s -mtriple=x86_64-unknown-unknown | FileCheck %s
+; RUN: llc < %s | FileCheck %s
:
```
由输出的内容可得，tag llvmorg-10.0.1对应的commit编号是ef32c611aa214dea855364efd7ba451ec5ec3f74，使用reset指令回退版本到该处：
```sh
git reset --hard ef32c611aa214dea855364efd7ba451ec5ec3f74
```
版本回退成功后，输入`git show`可以看到，git仓库的head指针已经回退到了llvm 10.0.1版本。
## 4.3 编译llvm工程
选择github仓库上的release/10.x分支，查阅对应的Readme.md，并根据Readme的指示进行安装。

首先，建立并切换到对应的build目录：
```sh
cd llvm-project
mkdir build
cd build
```
然后，根据编译需求，输入cmake指令build工程。此处选择Ninja方式建立文件，额外编译clang, clang-tool-extra, compiler-rt项目。
```sh
cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;compiler-rt;" ../llvm
```
在build的过程中，可能会出现一些Not found的提示信息，只要没有报错，就可以正常进行。

命令执行完以后，可以**立刻**输入`echo $?`命令查看上一条命令的执行情况，如果返回值为0，则说明build命令执行成功。

因为选择了Ninja方式build文件，此处采用ninja的命令完成工程构建：
```sh
ninja && ninja install
```

同理，命令执行完以后，**立刻**输入`echo $?`命令查看上一条命令的执行情况，如果返回值为0，则说明build命令执行成功。

输入指令`clang -v`验证是否安装成功，返回如下信息：
```
clang version 10.0.1 (https://github.com.cnpmjs.org/llvm/llvm-project.git ef32c611aa214dea855364efd7ba451ec5ec3f74)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /usr/local/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Candidate multilib: .;@m64
Selected multilib: .;@m64
```
可见已成功安装llvm+clang 10.0.1版本。

----
**20210518更新：如安装过程中gg，有可能是交换空间不足，需要手动挂载swap，可以搜索引擎查找相关教程，本文不再列出；如果是其他错误，按错误提示按部就班解决即可。**
