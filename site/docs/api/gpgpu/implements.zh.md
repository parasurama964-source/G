---
title: 经典 GPGPU 的实现原理
order: 6
---

使用图形渲染 API 实现的经典 GPGPU 的原理可以简单总结为：用纹理映射实现的科学计算 (compulation by texturing)。考虑到兼容性，我们在 WebGL 中也使用了这个方式。

下图来自：「GPGPU 编程技术 - 从 GLSL、CUDA 到 OpenCL」 ![image](https://user-images.githubusercontent.com/3608471/84491693-83f46700-acd7-11ea-8d5a-15edb3285e75.png)

## 渲染到纹理

通常来说图形渲染 API 最终的输出目标就是屏幕，显示渲染结果。但是在 GPGPU 场景中我们只是希望在 CPU 侧读取最终的计算结果。因此会使用到渲染 API 提供的离屏渲染功能 --- 渲染到纹理，其中的关键技术就是使用帧缓存对象（Framebuffer Object - FBO）作为渲染对象。

但是这种方式存在一个明显的限制，对于所有线程，纹理缓存要么是只读的，要么就是只写的，没法实现一个线程在读纹理，另一个在写纹理。本质上是由 GPU 的硬件设计决定的，如果想要实现多个线程同时对同一个纹理进行读/写操作，需要设计复杂的同步机制避免读写冲突，势必会影响到线程并行执行的效率。

因此在经典 GPGPU 的实现中，通常我们会准备两个纹理，一个用来保存输入数据，一个用来保存输出数据。这也是为何我们只允许使用一个 `@out` 声明来输出变量。

我们的数据存储在显存中，使用 RGBA 的纹理格式，每一个图元包含 4 个通道，因此在 GWebGPU 中使用 `vec4[]` 是最省内存的数据格式。如果使用 `float[]`，每个图元中 GBA 三个通道就被浪费了。当然数据类型的决定权在开发者，可以根据实际程序中访问方便程度决定。

## 调用绘制命令

我们的计算逻辑写在片元着色器（Fragment Shader）中，在渲染管线的光栅化阶段，每个像素被分配给一个线程进行着色，达到并行效果。

如果映射到 CPU 中的计算概念，纹理可以看作是数组，而片元着色器执行的程序就是循环语句。

## 什么是纹理映射

一个 3D 模型由很多个三角面组成，理论上每个三角面都可以继续无限细分，但给每个三角面着色是很消耗性能的。更快的做法是贴图，把一张二维位图（纹理）贴在模型的表面，这个过程就是纹理映射。我们不需要为模型每一个顶点定义纹理坐标，只需要定义四个角的坐标，剩余的交给渲染管线做插值即可。

## 乒乓技术

很多算法需要连续运行多次，例如 G6 中使用的布局算法需要迭代多次达到稳定状态。上一次迭代中输出的计算结果，需要作为下一次迭代的输入。在实际实现中，我们会分配两张纹理缓存，每次迭代后对输入和输出纹理进行 swap。

## 参考资料

- 「GPGPU 编程技术 - 从 GLSL、CUDA 到 OpenCL」[🔗](https://book.douban.com/subject/6538230/)
- <http://www.vizitsolutions.com/portfolio/webgl/gpgpu/>
