# **英特尔** 东方杯-**oneAPI** **黑客松**大赛

本次比赛中，我们需要利用英特尔的oneMKL (oneAPI Math Kernel Library) 库实现2D Real to complex FFT，并与我们常用的FFTW库进行时间性能上的对比。

## 题目描述

1. 调用 oneMKL 相应 API 函数， 产生 2048 * 2048 个 随机单精度实数();
2. 根据 2 产生的随机数据作为输入，实现两维 Real to complex FFT 参考代码;
3. 根据 2 产生的随机数据作为输入， 调用 oneMKL API 计算两维 Real to complex FFT;
4. 结果正确性验证，对 3 和 4 计算的两维 FFT 输出数据进行全数据比对(允许适当精度误 差)， 输出 “结果正确”或“结果不正确”信息;
5. 平均性能数据比对(比如运行 1000 次)，输出 FFT 参考代码平均运行时间和 oneMKL FFT 平均运行时间。

## 解决方法

### 环境配置

首先需要下载安装英特尔oneMKL库，下载链接在[这里](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html)。

注意：需要提前安装oneKML库，并将路径设置为 ``/opt/intel/oneapi``

安装完成后，利用以下命令初始化环境：

```shell
./opt/intel/oneapi/setvars.sh
```



在本次比赛中，我们使用maxOS下的Xcode作为编译器，需要配置编译器环境。

在Xcode中新建一个cpp项目，打开项目配置界面，然后按照如下步骤将项目链接到oneMKL库：

1. 在Build Settings --> Search Paths --> Header Search Paths中添加

   ```shell
   /opt/intel/oneapi/mkl/2023.2.0/include
   ```

2. 在Build Settings --> Search Paths --> Library Search Paths中添加

   ```shell
   /opt/intel/oneapi/mkl/2023.2.0/lib
   ```

3. 在Build Settings --> Linking --> Runpath Search Paths中添加

   ```shell
   /opt/intel/oneapi/compiler/2023.2.0/mac/compiler/lib
   /opt/intel/oneapi/mkl/2023.2.0/lib
   ```

4. 在Build Settings --> Linking --> Other Linker Flags中添加

   ```
   -lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core -lpthread -lm
   ```

至此，环境配置完成。配置完成后如下图：

![config](./figures/config.png)



### 问题解决

1. 利用oneMKL库的API生成2048*2048个随机单精度浮点数

   ```cpp
   #define ROWS 2048
   #define COLS 2048
   #define SIZE 2048 * 2048
   
   float* randomData = new float[SIZE];
   void generate_data()
   {
       VSLStreamStatePtr stream;
       vslNewStream(&stream, VSL_BRNG_MT19937, 666);
       vsRngUniform(VSL_RNG_METHOD_UNIFORM_STD, stream, SIZE, randomData, 0.0, 1.0);
       vslDeleteStream(&stream);
   }
   ```

   

2. 利用FFTW3库实现2D Real to complex FFT

   ```cpp
   fftwf_complex *fftwOutput = (fftwf_complex*) fftw_malloc(sizeof(fftwf_complex) * ROWS * (COLS/ 2 + 1) * 2);
   void fft_by_fft_implement()
   {
       fftwf_plan plan = fftwf_plan_dft_r2c_2d(ROWS, COLS, randomData, fftwOutput, FFTW_ESTIMATE);
       fftwf_execute(plan);
       fftwf_destroy_plan(plan);
   }
   ```

3. 利用oneMKL库API实现2D Real to complex FFT

   ```cpp
   DFTI_DESCRIPTOR_HANDLE descriptor;
   MKL_LONG status;
   MKL_LONG sizes[2] = {ROWS, COLS};
   complex<float>* mklOutput = new complex<float>[SIZE];
   void fft_by_mkl()
   {
       status = DftiCreateDescriptor(&descriptor, DFTI_SINGLE, DFTI_REAL, 2, sizes);
       DftiSetValue(descriptor, DFTI_PLACEMENT, DFTI_NOT_INPLACE);
       DftiSetValue(descriptor, DFTI_CONJUGATE_EVEN_STORAGE, DFTI_COMPLEX_COMPLEX);
       
       status = DftiCommitDescriptor(descriptor);
       status = DftiComputeForward(descriptor, randomData, mklOutput);
       DftiFreeDescriptor(&descriptor);
   }
   ```

4. 将两种方法计算的FFT进行逐位比较，并分别再运行1000次，计算平均性能，运行结果见下图：![result](./figures/result.png)

## 参赛经验

在参加本次东方杯—英特尔 oneAPI 黑客松大赛的过程中，我们通过亲身实践，了解了oneMKL对于FFT这类数学运算的加速效果是十分显著的，同时收获了许多宝贵的经验知识。我们了解了oneMKL库的基本功能及其API的基本用法，这在我们未来的研究和实践中能够为我们提供很大的帮助，也希望能给更多的人带来帮助，将oneMKL库应用于解决现实问题中。