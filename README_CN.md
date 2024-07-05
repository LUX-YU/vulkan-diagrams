# Vulkan图表（Vulkan Diagrams）

## 目录

- [简介（Introduction）](#简介)
- [样板代码（Boilerplate）](#样板代码)
- [渲染通道和交换链（Render Pass and Swapchain）](#渲染通道和交换链)
- [顶点缓冲区（Vertex Buffer）创建](#顶点缓冲区创建)
- [顶点输入和多重绑定（Vertex Input and Multiple Bindings）](#顶点输入和多重绑定)
- [描述符集（Descriptor Sets）](#描述符集)
- [推送常量（Push Constants）](#推送常量)
- [流水线障碍（Pipeline Barriers）](#流水线障碍)
- [流水线阶段和访问类型（Pipeline Stages and Access Types）](#流水线阶段和访问类型)
- [渲染循环（Render Loop）](#渲染循环)
- [光线追踪（Ray Tracing）](#光线追踪)

## 简介（Introduction）

Vulkan图表（Vulkan Diagrams）是一系列设计用来快速参考Vulkan中各种主题的图表。这些图表展示了完成常见任务（例如创建顶点缓冲区）所需的Vulkan对象以及这些对象之间的关系。

为了清晰，省略了一些Vulkan对象的成员，并且某些名称进行了简化（例如将`VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`简化为`SHADER_READ_ONLY_OPTIMAL`）。

## 样板代码（Boilerplate）
这张图表使用了来自我的GPU，一个NVIDIA GeForce GTX 1080的示例值。请注意，内存堆（Memory Heap）2既是`DEVICE_LOCAL`也是`HOST_VISIBLE`，这对于需要从CPU频繁更新的GPU可访问内存很有好处，因为它允许我们直接更新它，无需使用阶段缓冲区（staging buffer）。在我的机器上，它只有224 MB，但在具有可调整大小条（resizeable bar，AMD的市场术语为Smart Access Memory）的机器上，这个堆将会显著更大。

图表中未显示的功能是`vkEnumerateInstanceExtensionProperties(...)`、`vkEnumerateInstanceLayerProperties(...)`和`vkEnumerateDeviceExtensionProperties(...)`，用于分别枚举可用的实例扩展（instance extensions）、层（layers）和设备扩展（device extensions）。

运行[vulkaninfo](https://vulkan.lunarg.com/doc/view/1.2.148.1/windows/vulkaninfo.html)将打印出关于您的GPU可以在Vulkan中查询的规格信息。要了解其他GPU的信息，[gpuinfo](https://vulkan.gpuinfo.org/)是一个好资源。如果您想临时覆盖验证层（validation layer）设置，可以使用[Vulkan Configurator](https://vulkan.lunarg.com/doc/view/1.2.148.1/windows/vkconfig.html)。

另外，[vk-bootstrap](https://github.com/charles-lunarg/vk-bootstrap)是一个处理Vulkan样板（boilerplate）的好库。

![boiler_plate](boiler_plate.png?raw=true "boiler_plate")

## 渲染通道和交换链（Render Pass and Swapchain）

在这张图中，每个命令缓冲区（command buffer）使用单个渲染通道（render pass），每个渲染通道有多个子通道（subpasses）。

您应该尽可能使用多个子通道而不是多个渲染通道。如果一个通道只需要从前一个通道中读取对应的片段，则可以使用之前的子通道作为输入附件（input attachment），不需要额外的渲染通道。[这里有一个如何做到这一点的例子](https://www.saschawillems.de/blog/2018/07/19/vulkan-input-attachments-and-sub-passes/)。如果您需要随机访问之前的通道（例如实现高斯模糊），那么使用多个渲染通道将是适当的。

在图中，我们看到帧缓冲区（frame buffer）中的一个附件拥有一个属于交换链（swapchain）的图像，但这不是强制性的。例如，您可以通过创建自己的`VkImage`并使用使用标志`VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT`来渲染到纹理，以便它可以作为颜色附件写入然后在着色器中采样。

![render_pass_swapchain](render_pass_swapchain.png?raw=true "render_pass_swapchain")
## 顶点缓冲区（Vertex Buffer）创建

NVIDIA有一篇关于内存管理的[好文章](https://developer.nvidia.com/vulkan-memory-management)，我推荐阅读。重点是，通常更倾向于进行大的内存分配以及创建大的缓冲区，并从这些缓冲区中子分配资源。它包括以下对内存对象的描述：

<blockquote>
<b>堆（Heap）</b> - 根据硬件和平台，设备将公开固定数量的堆，您可以从中分配总量的内存。拥有专用内存的独立GPU将不同于与CPU共享内存的移动或集成解决方案。堆支持不同的内存类型，这些类型必须从设备查询。
  
<b>内存类型（Memory type）</b> - 创建资源（如缓冲区）时，Vulkan将提供哪些内存类型与资源兼容的信息。根据额外的使用标志，开发者必须选择正确的类型，并基于该类型选择合适的堆。

<b>内存属性标志（Memory property flags）</b> - 这些标志编码了缓存行为以及我们是否可以将内存映射到主机（CPU），或者GPU是否可以快速访问内存。

<b>内存（Memory）</b> - 这个对象代表从某个堆中以用户定义的大小进行的分配。

<b>资源（Resource，缓冲区/图像）</b> - 查询内存需求并选择一个兼容的分配后，内存会与资源关联到一个特定的偏移量。这个偏移量必须满足提供的对齐要求。之后，我们可以开始使用我们的资源进行实际工作。

<b>子资源（Sub-Resource，偏移量/视图）</b> - 不需要只使用资源的全部范围，就像在OpenGL中我们可以绑定范围（例如，变化顶点缓冲区的起始偏移量）或使用视图（例如，纹理数组的单独切片和mipmap）。
</blockquote>

另外，[Vulkan内存分配器（VMA）](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)是一个处理内存分配的好库。

![vertex_buffer](vertex_buffer.png?raw=true "vertex_buffer")

## 顶点输入和多重绑定（Vertex Input and Multiple Bindings）

`VkPipelineVertexInputStateCreateInfo`允许我们指定顶点在内存中的存储方式。它由`VkVertexInputAttributeDescription`数组和`VkVertexInputBindingDescription`数组组成。

顾名思义，我们将为每个绑定有一个绑定描述。在这个例子中，我们看到绑定0的输入率是`VK_VERTEX_INPUT_RATE_VERTEX`，这意味着它将对每个_顶点_递增到下一组数据`stride`。另一方面，绑定1的输入率是`VK_VERTEX_INPUT_RATE_INSTANCE`，所以我们只会对每个_实例_递增到下一组数据`stride`。我们在`vkCmdDraw`中指定我们绘制的实例和顶点数量。

我们为与该绑定关联的结构体的每个成员有一个顶点属性。例如，绑定0有3个顶点属性，因为绑定到绑定0的顶点缓冲区是`Vertex`结构体的缓冲区，其成员包括`position`、`normal`和`texCoord`。绑定1只有2个顶点属性，因为`InstanceData`只有2个成员。每个顶点属性的格式由该属性的大小和类型决定，因此一些常见选择包括：

```
float: VK_FORMAT_R32_SFLOAT
vec2: VK_FORMAT_R32G32_SFLOAT
vec3: VK_FORMAT_R32G32B32_SFLOAT
vec4: VK_FORMAT_R32G32B32A32_SFLOAT
ivec2: VK_FORMAT_R32G32_SINT
uvec4: VK_FORMAT_R32G32B32A32_UINT
double: VK_FORMAT_R64_SFLOAT
```


在这个例子中，我们一次调用`vkCmdBindVertexBuffer`并同时绑定两个缓冲区的数组，但请注意，如果在一个调用中指定`firstBinding = 0`，在另一个中指定`firstBinding = 1`，则完全可以进行两次单独的调用。

最后，注意顶点着色器（vertex shader）完全不知道它的输入变量来自哪个绑定。顶点着色器只指定位置，然后每个变量的数据来源由相应的`VkVertexInputAttributeDescription`确定。

![vertex_input](vertex_input.png?raw=true "vertex_input")

<b>描述符集（DescriptorSet）</b> - 由描述符集布局定义的描述符的实际实例。使用类/结构体类比，就像进行`MyDesc DescInstance();`

<b>流水线布局（PipelineLayout）</b> - 如果将整个着色器视为一个大的`void shader(arguments)`函数，那么流水线布局就像描述传递到您的着色器的所有“参数”，例如`void shader(MyDesc desc, MyOtherDesc otherDesc)`。这通常对应于您的着色器代码中的语句，如`layout(std140,set=0, binding = 0) uniform UBufferInfo{Blah MyBlah;}`和`layout(set=0, binding = 2, rgba32f) uniform image2D MyImage;`。

<b>vkCmdBindDescriptorSet</b> - 这是将描述符集传递到着色器（即流水线）的机制。基本上就是像`shader(DescInstance,OtherDescInstance)`这样传递“参数”。
</blockquote>

注意，`vkUpdateDescriptorSets(...)`不是将缓冲区复制到描述符集中，而是让描述符集指向由`VkDescriptorBufferInfo`描述的缓冲区。因此，`vkUpdateDescriptorSets(...)`不需要为描述符集调用多次，因为修改描述符集指向的缓冲区会更新描述符集看到的内容。

![descriptor_sets](descriptor_sets.png?raw=true "descriptor_sets")

## 推送常量（Push Constants）

推送常量（Push constants）是一小部分可以写入命令缓冲区本身的着色器可访问数据。规范只保证128字节的推送常量空间，尽管确切的限制可以从`VkPhysicalDeviceLimits::maxPushConstantsSize`找到。

图表显示如何使用范围和偏移使某些范围从顶点着色器（vertex shader）可用，另一些从片段着色器（fragment shader）可用，但也可以指定只有一个范围，且`stageFlags`设置等于`VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT`；

![push constants](push_constants.png?raw=true "descriptor_sets")

## 流水线障碍（Pipeline Barriers）

此图表显示了流水线障碍的一般用途以及它们如何创建执行依赖和内存依赖。图中的具体示例显示了流水线障碍，它将图像的布局从`VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`转换为`VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`。这是在将图像数据从缓冲区复制到图像后进行的，以准备图像从着色器读取。这个例子摘自[Vulkan教程的纹理映射章节](https://vulkan-tutorial.com/Texture_mapping/Images)。

集合名称和引用直接来自[规范](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#synchronization-dependencies)。

(ctrl+f 搜索PDF版本规范：<i><b>执行和内存依赖(Execution and Memory Dependencies)</b></i>)

![pipeline_barriers](barrier.png?raw=true "pipeline_barriers")
## 流水线阶段和访问类型（Pipeline Stages and Access Types）

使用流水线障碍或子通道依赖进行同步时，需要指定源和目的阶段/访问类型。以下表格显示了每个流水线的所有阶段及其发生的顺序，以及每个阶段支持的访问类型。`VK_ACCESS_MEMORY_READ_BIT` 和 `VK_ACCESS_MEMORY_WRITE_BIT` 适用于所有阶段，但为了清晰，这些表中没有显示。

对应的 `VkPipelineStageFlagBits` 和 `VkAccessFlagBits` 分别出现在阶段和访问类型列中。

(ctrl+f 搜索PDF版本规范：<i><b>支持的访问类型表</b></i> 和 <i><b>图形流水线执行以下阶段</b></i>)

图形流水线（Graphics pipeline）：

| 阶段（Stage）                             | 访问类型（Access Types）                                                                                         |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| Draw Indirect                              | Indirect Command Read                                                                                     |
| Index Input                                | Index Read                                                                                                |
| Vertex Attribute Input                     | Vertex Attribute Read                                                                                     |
| Vertex Shader                              | Uniform Read<br>Shader Read<br>Shader Write<br>Acceleration Structure Read                                |
| Tessellation Control Shader                | Uniform Read<br>Shader Read<br>Shader Write<br>Acceleration Structure Read                                |
| Tessellation Evaluation Shader             | Uniform Read<br>Shader Read<br>Shader Write<br>Acceleration Structure Read                                |
| Geometry Shader                            | Uniform Read<br>Shader Read<br>Shader Write<br>Acceleration Structure Read                                |
| Fragment Shading Rate Attachment           | Fragment Shading Rate Attachment Read                                                                     |
| Early Fragment Tests                       | Depth Stencil Attachment Read<br>Depth Stencil Attachment Write                                           |
| Fragment Shader                            | Uniform Read<br>Shader Read<br>Shader Write<br>Input Attachment Read<br>Acceleration Structure Read       |
| Late Fragment Tests                        | Depth Stencil Attachment Read<br>Depth Stencil Attachment Write                                           |
| Color Attachment Output                    | Color Attachment Read<br>Color Attachment Write                                                           |

计算流水线（Compute pipeline）：

| 阶段（Stage）         | 访问类型（Access Types）                                                              |
| --------------------- | ------------------------------------------------------------------------------------- |
| Draw Indirect         | Indirect Command Read                                                                 |
| Compute Shader        | Uniform Read<br>Shader Read<br>Shader Write<br>Acceleration Structure Read            |

传输流水线（Transfer pipeline）：

| 阶段（Stage） | 访问类型（Access Types）               |
| -------------- | -------------------------------------- |
| Transfer       | Transfer Read<br>Transfer Write        |

主机操作（Host operations）：

| 阶段（Stage） | 访问类型（Access Types）       |
| ------------- | ------------------------------ |
| Host          | Host Read<br>Host Write        |

加速结构操作（Acceleration structure operations）：

| 阶段（Stage）                        | 访问类型（Access Types）                                                                                      |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Acceleration Structure Build         | Draw Indirect<br>Shader Read<br>Transfer Read<br>Transfer Write<br>Acceleration Structure Read<br>Acceleration Structure Write |

光线追踪流水线（Ray tracing pipeline）：

| 阶段（Stage）            | 访问类型（Access Types）                                                              |
| ------------------------- | ------------------------------------------------------------------------------------- |
| Draw Indirect             | Indirect Command Read                                                                 |
| Ray Tracing Shader        | Uniform Read<br>Shader Read<br>Shader Write<br>Acceleration Structure Read            |

## 渲染循环（Render Loop）

在这张图中，时间从上到下推移。这个[渲染循环的C++实现](https://github.com/David-DiGioia/pumpkin/blob/f0d86f5b561abbb124a6086ec724b96915090e87/src/renderer/vulkan_renderer.cpp#L41)可以在我的[(WIP) Pumpkin游戏引擎仓库](https://github.com/David-DiGioia/pumpkin)中找到。

CPU标记为“CPU工作”的部分专门指在渲染该帧时由GPU读取的资源的写入。例如，通过写入它们的UBOs来更新所有渲染对象的变换。

请注意，`vkQueuePresentKHR`何时完成呈现是不明确的，因为API不接受任何同步原语来信号它完成。相反，您只能知道当`vkAcquireNextImageKHR`返回其索引并信号同步原语表明它已准备好时，交换链图像才完成使用。或者，如果您想明确等待呈现完成，可以使用[VK_KHR_present_wait](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VK_KHR_present_wait.html)扩展。

虽然有两个不同的时间线绘制了每个帧在飞行中的`vkQueueSubmit`，但它们都提交到同一个队列。

注意：`vkAcquireNextImageKHR`不会信号信号灯/栅栏，直到图像准备好，图像不会准备好，直到足够之前获得的图像通过`vkQueuePresentKHR`释放。`vkAcquireNextImageKHR`将返回代码`VK_NOT_READY`表示信号灯/栅栏没有立即发出信号，但一旦获得图像，它将稍后发出信号。

如果启用了垂直同步，`vkQueuePresentKHR`是在下一个垂直同步周期阻塞的功能，至少在我的GeForce GTX 1080上是这样。

![render_loop](render_loop.png?raw=true "render_loop")

## 光线追踪（Ray Tracing）

Vulkan中的光线追踪包括构建加速结构（acceleration structures），创建光线追踪流水线和着色器绑定表（shader binding table, SBT），然后用`vkCmdTraceRaysKHR(...)`追踪射线。还有[VK_KHR_ray_query](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_ray_query.html)用于在现有着色器中投射射线，不需要光线追踪流水线，但这里没有讨论。

构建加速结构的大部分工作由驱动程序完成，但应用程序开发者负责在顶级加速结构（TLAS）中放置实例，将它们的基元分组到底级加速结构（BLASes）中，并在该BLAS中将基元分组到几何体中。这样做的方式对性能有重要影响。我在[GPUOpen上写了一篇文章](https://gpuopen.com/learn/improving-rt-perf-with-rra/)，详细讨论了光线追踪性能的最佳实践。

第一个图表展示了构建BLAS所需的Vulkan对象。

![ray_tracing_build_blas](ray_tracing_build_blas.png?raw=true "ray_tracing_build_blas")

注意，几乎没有实现支持`VkPhysicalDeviceAccelerationStructureFeaturesKHR::accelerationStructureHostCommands`，因此你很可能需要在设备上构建加速结构，如图所示。这使得压缩BLAS更加复杂，因为它需要两次队列提交。压缩BLAS的过程如下：

1) 在构建原始加速结构的`VkAccelerationStructureBuildGeometryInfoKHR::flags`中添加`VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_COMPACTION_BIT_KHR`标志。

2) 创建原始加速结构。

3) 创建`VkQueryPool`，使用`VkQueryPoolCreateInfo::queryType`的`VK_QUERY_TYPE_ACCELERATION_STRUCTURE_COMPACTED_SIZE_KHR`。

4) 用`vkCmdWriteAccelerationStructuresPropertiesKHR(...)`查询压缩尺寸。

5) 提交命令缓冲区，然后用`vkGetQueryPoolResults(...)`获取查询结果。

6) 使用查询得到的压缩尺寸创建一个`VkAccelerationStructureKHR`和`VkBuffer`。

7) 开始记录新的命令缓冲区。

8) 使用`VkCopyAccelerationStructureInfoKHR::mode`设置为`VK_COPY_ACCELERATION_STRUCTURE_MODE_COMPACT_KHR`的`vkCmdCopyAcceleration

## 光线追踪（Ray Tracing）

Vulkan中的光线追踪包括构建加速结构（acceleration structures），创建光线追踪流水线和着色器绑定表（shader binding table, SBT），然后用`vkCmdTraceRaysKHR(...)`追踪射线。还有[VK_KHR_ray_query](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_ray_query.html)用于在现有着色器中投射射线，不需要光线追踪流水线，但这里没有讨论。

构建加速结构的大部分工作由驱动程序完成，但应用程序开发者负责在顶级加速结构（TLAS）中放置实例，将它们的基元分组到底级加速结构（BLASes）中，并在该BLAS中将基元分组到几何体中。这样做的方式对性能有重要影响。我在[GPUOpen上写了一篇文章](https://gpuopen.com/learn/improving-rt-perf-with-rra/)，详细讨论了光线追踪性能的最佳实践。

第一个图表展示了构建BLAS所需的Vulkan对象。

![ray_tracing_build_blas](ray_tracing_build_blas.png?raw=true "ray_tracing_build_blas")

注意，几乎没有实现支持`VkPhysicalDeviceAccelerationStructureFeaturesKHR::accelerationStructureHostCommands`，因此你很可能需要在设备上构建加速结构，如图所示。这使得压缩BLAS更加复杂，因为它需要两次队列提交。压缩BLAS的过程如下：

1) 在构建原始加速结构的`VkAccelerationStructureBuildGeometryInfoKHR::flags`中添加`VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_COMPACTION_BIT_KHR`标志。

2) 创建原始加速结构。

3) 创建`VkQueryPool`，使用`VkQueryPoolCreateInfo::queryType`的`VK_QUERY_TYPE_ACCELERATION_STRUCTURE_COMPACTED_SIZE_KHR`。

4) 用`vkCmdWriteAccelerationStructuresPropertiesKHR(...)`查询压缩尺寸。

5) 提交命令缓冲区，然后用`vkGetQueryPoolResults(...)`获取查询结果。

6) 使用查询得到的压缩尺寸创建一个`VkAccelerationStructureKHR`和`VkBuffer`。

7) 开始记录新的命令缓冲区。

8) 使用`VkCopyAccelerationStructureInfoKHR::mode`设置为`VK_COPY_ACCELERATION_STRUCTURE_MODE_COMPACT_KHR`的`vkCmdCopyAccelerationStructureKHR(...)`将原始加速结构复制到压缩的加速结构中。

9) 提交命令缓冲区。

下一个图表显示了构建TLAS所需的Vulkan对象。

![ray_tracing_build_tlas](ray_tracing_build_tlas.png?raw=true "ray_tracing_build_tlas")

最后，光线追踪流水线和着色器绑定表。

![ray_tracing_sbt](ray_tracing_sbt.png?raw=true "ray_tracing_sbt")
