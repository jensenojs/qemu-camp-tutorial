# GPGPU 学习资料

!!! note "主要贡献者"

    - 作者：[@zevorn](https://github.com/zevorn)

本文汇总了 GPGPU（General-Purpose computing on Graphics Processing Units）相关的在线学习资料，涵盖视频课程、在线教程、电子文档和开源工具等，帮助你系统掌握 GPU 架构与编程知识，为训练营中的 GPGPU 建模、CXL 加速器仿真等课程打下基础。

## 视频课程

### 免费课程

!!! tip "推荐视频"

    - [freeCodeCamp: CUDA Programming（12 小时完整课程）][freecodecamp-cuda-video] — 覆盖 CUDA 入门、GPU 架构、Kernel 编写、矩阵乘法优化、Triton、PyTorch 扩展等，配套代码：[GitHub 仓库][freecodecamp-cuda-github]
    - [Coursera: Introduction to Parallel Programming with CUDA][coursera-cuda-intro] — Johns Hopkins 大学，可免费旁听
    - [IIT Kharagpur: GPU Architectures and Programming（NPTEL）][iit-gpu-course] — 覆盖 CPU/GPU 架构、SIMD、CUDA 编程模型

### 付费 / 部分免费

!!! abstract "进阶课程"

    - [Coursera: GPU Programming Specialization][coursera-gpu-spec] — Johns Hopkins 大学，系统学习 GPU 编程
    - [Udemy: CUDA GPU Programming Beginner To Advanced][udemy-cuda] — 从基础到高级的 CUDA 编程
    - [NVIDIA DLI 官方培训][nvidia-dli] — NVIDIA 官方自学 / 讲师课程

## 在线教程 & 文档

### 中文教程

!!! note "中文资源"

    - [HPC Wiki: CUDA 编程入门][hpcwiki-cuda] — 系统的中文入门教程
    - [知乎：CUDA 编程入门极简教程][zhihu-cuda] — 快速上手
    - [中国科大超算中心：GPU 异构计算和 CUDA 程序简介][ustc-gpu] — 高校超算文档
    - [NVIDIA 官方中文博客：CUDA 编程手册系列][nvidia-cn-blog] — 官方中文版
    - [NVIDIA CUDA 编程指南 v1.1 中文全译本（PDF）][nvidia-cuda-guide-cn] — 官方编程指南完整翻译
    - [鲁老师：初识 GPU 编程][lulaoshi-gpu] — Python + CUDA 入门
    - [Ji Zhuoran: GPU 编程手册][jizhuoran-gpu] — 系统学习 GPU 编程

### 英文教程

!!! note "英文资源"

    - [Caltech CS179: GPU Programming][caltech-cs179] — 课程材料公开
    - [Oxford: Course on CUDA Programming][oxford-cuda] — 牛津大学 CUDA 课程资料
    - [freeCodeCamp: Learn CUDA Programming（文字版）][freecodecamp-cuda-text] — 配合视频使用

## GPU 架构电子文档

!!! abstract "PDF / 电子书"

    - [General-Purpose Graphics Processor Architecture（2018）][gpgpu-arch-2018] — 综合性 GPGPU 架构电子书
    - [Caroline Collange: Introduction to GPU Architecture][collange-gpu] — Inria 讲义
    - [AMD: Introduction to GPU Architecture][amd-gpu-intro] — 从 AMD/OpenCL 视角讲解
    - [GPU Architecture and Programming][laas-gpu] — LAAS 课程资料
    - [UPenn: GPU Architecture][upenn-gpu] — 宾夕法尼亚大学讲义
    - [Demystifying NVIDIA GPU Internals][nvidia-gpu-internals] — 深入 NVIDIA GPU 内部机制的研究论文
    - [Blue Porcelain GPGPU 开源架构参考书目][gpgpuarch-books] — 开源 GPGPU 芯片平台推荐书目

## Vulkan / OpenCL 计算着色器

!!! tip "通用计算 API"

    - [Vulkan Tutorial: Compute Shader][vulkan-tutorial-compute] — 从零构建 GPU 粒子系统模拟
    - [Vulkan 官方文档：Compute Shader][vulkan-docs-compute] — 官方教程
    - [Khronos: Getting Started with Vulkan Compute Acceleration][khronos-vulkan] — Vulkan 计算入门指南
    - [Vulkan Compute Example][vulkan-compute-example] — 简单的 Vulkan GPGPU 示例
    - [ArchWiki: GPGPU 概览][archwiki-gpgpu] — 各 GPGPU API（CUDA/OpenCL/Vulkan/HIP/SYCL）对比

## 模拟器 & 练手平台

!!! example "动手实践"

    - [GPGPU-Sim][gpgpu-sim] — 周期精确的 GPU 模拟器，支持 CUDA/OpenCL，适合研究 GPU 微架构
    - [GPGPU-Sim 入门教程][gpgpu-sim-tutorial] — CoffeeBeforeArch 博客教程
    - [Accel-Sim Framework][accel-sim] — 基于 GPGPU-Sim 的加速模拟框架
    - [LeetGPU][leetgpu] — GPU 编程挑战平台（类似 LeetCode）
    - [Class Central: GPU Computing 课程索引][classcentral-gpu] — 800+ 门课程聚合搜索
    - [XPU OJ](https://xpuoj.com/) - 国内本土化的 LeetGPU，国产算力支持！


[freecodecamp-cuda-video]: https://www.youtube.com/watch?v=86FAWCzIe_4
[freecodecamp-cuda-github]: https://github.com/Infatoshi/cuda-course
[coursera-cuda-intro]: https://www.coursera.org/learn/introduction-to-parallel-programming-with-cuda
[iit-gpu-course]: https://www.classcentral.com/course/swayam-gpu-architectures-and-programming-17622
[coursera-gpu-spec]: https://www.coursera.org/specializations/gpu-programming
[udemy-cuda]: https://www.udemy.com/course/cuda-gpu-programming-beginner-to-advanced/
[nvidia-dli]: https://developer.nvidia.com/cuda-education-training
[hpcwiki-cuda]: https://hpcwiki.io/gpu/cuda/
[zhihu-cuda]: https://zhuanlan.zhihu.com/p/34587739
[ustc-gpu]: http://scc.ustc.edu.cn/zlsc/user_doc/html/gpu-computing/gpu-computing.html
[nvidia-cn-blog]: https://developer.nvidia.com/zh-cn/blog/cuda-intro-cn/
[nvidia-cuda-guide-cn]: https://www.nvidia.cn/docs/IO/51635/NVIDIA_CUDA_Programming_Guide_1.1_chs.pdf
[lulaoshi-gpu]: https://lulaoshi.info/gpu/python-cuda/cuda-intro.html
[jizhuoran-gpu]: https://jizhuoran.github.io/intro2GPU/
[caltech-cs179]: https://courses.cms.caltech.edu/cs179/
[oxford-cuda]: https://people.maths.ox.ac.uk/gilesm/cuda/
[freecodecamp-cuda-text]: https://www.freecodecamp.org/news/learn-cuda-programming/
[gpgpu-arch-2018]: https://github.com/tpn/pdfs/blob/master/General-Purpose%20Graphics%20Processor%20Architecture%20(2018).pdf
[collange-gpu]: https://www.irisa.fr/alf/downloads/collange/cours/ada2020_gpu_1.pdf
[amd-gpu-intro]: http://www.haifux.org/lectures/267/Introduction-to-GPUs.pdf
[laas-gpu]: https://homepages.laas.fr/adoncescu/FILS/GPU.pdf
[upenn-gpu]: https://acg.cis.upenn.edu/milom/cis371-Spring12/lectures/GPU-Architecture.pdf
[nvidia-gpu-internals]: https://www.cs.unc.edu/~anderson/papers/rtas24.pdf
[gpgpuarch-books]: https://gpgpuarch.org/en/ref/book/
[vulkan-tutorial-compute]: https://vulkan-tutorial.com/Compute_Shader
[vulkan-docs-compute]: https://docs.vulkan.org/tutorial/latest/11_Compute_Shader.html
[khronos-vulkan]: https://www.khronos.org/blog/getting-started-with-vulkan-compute-acceleration
[vulkan-compute-example]: https://github.com/Glavnokoman/vulkan-compute-example
[archwiki-gpgpu]: https://wiki.archlinux.org/title/General-purpose_computing_on_graphics_processing_units
[gpgpu-sim]: https://github.com/gpgpu-sim/gpgpu-sim_distribution
[gpgpu-sim-tutorial]: https://coffeebeforearch.github.io/2020/03/30/gpgpu-sim-1.html
[accel-sim]: https://accel-sim.github.io/
[leetgpu]: https://www.leetgpu.com/resources
[classcentral-gpu]: https://www.classcentral.com/subject/gpu-computing
