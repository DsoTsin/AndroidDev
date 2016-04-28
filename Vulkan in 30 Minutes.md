# 30分钟认识Vulkan API

---

	

> 这篇文章是写给对已有的D3D11和GL比较熟悉并理解多线程、资源暂存、同步等概念但想进一步了解它们是如何以Vulkan实现的读者，因此本文会以Vulkan概念之旅结束。

文章并不追求易懂（为了解里面的晦涩内容，你应该去阅读Vulkan规范或者更深度的教程）。

本文是我第一次见到Vulkan写下的。

－ baldurk

## General

---

> At the end of the post I've included a heavily abbreviated pseudocode program showing the rough steps to a hello world triangle, to match up to the explanations.

文末会介绍一个精简的伪代码来阐述显示一个三角形的大致步骤。

>A few simple things that don't fit any of the other sections:
>
>  	* Vulkan is a C API, i.e. free function entry points. This is the same as GL.
>  	* The API is quite heavily typed - unlike GL. Each enum is separate, handles that are returned are opaque 64-bit handles so they are typed on 64-bit (not typed on 32-bit, although you can make them typed if you use C++).
>	* A lot of functions (most, even) take extensible structures as parameters instead of basic types.
>	* VkAllocationCallbacks * is passed into creation/destruction functions that lets you pass custom malloc/free functions for CPU memory. For more details read the spec, in simple applications you can just pass NULL and let the implementation do its own CPU-side allocation.


以下是一些简单的注意点，不适用其他部分：

* Vulkan是一个类似GL的C语言图形库
* 这个API是强类型的，不像GL用GLenum就搞定所有参数，Vk的枚举是类型分离的。
* 多数函数调用的参数都是结构体或者是嵌套结构体。
* VkAllocationCallbacks * 是许多vkCreate＊函数的参数，它用来接管内存分配，当然你也可以简单地传NULL。

> 注意：我并没有考虑任何错误异常处理，也不谈论query的实现限制。

## 初始化步骤

---

>You initialise Vulkan by creating an instance (VkInstance). The instance is an entirely isolated silo of Vulkan - instances do not know about each other in any way. At this point you specify some simple information including which layers and extensions you want to activate - there are query functions that let you enumerate what layers and extensions are available.

Vulkan API的初始化必须要创建实例（`VkInstance`）。Vulkan实例之间是相互独立的，所以你可以给不同的实例设置不同的属性（例如，是否激活validation layer和extensions）。

>With a VkInstance, you can now examine the GPUs available. A given Vulkan implementation might not be running on a GPU, but let's keep things simple. Each GPU gives you a handle - VkPhysicalDevice. You can query the GPUs names, properties, capabilities, etc. For example see vkGetPhysicalDeviceProperties and vkGetPhysicalDeviceFeatures.

通过VkInstance可以检查GPU设备是否可用。Vulkan不一定是运行在GPU上，但我们把问题简单化。每个GPU都会提供一个句柄 - VkPhysicalDevice。你可以通过它来查询GPU厂商、属性（`vkGetPhysicalDeviceProperties`）、能力（`vkGetPhysicalDeviceFeatures`）等。

>With a VkPhysicalDevice, you can create a VkDevice. The VkDevice is your main handle and it represents a logical connection - i.e. 'I am running Vulkan on this GPU'. VkDevice is used for pretty much everything else. This is the equivalent of a GL context or D3D11 device.

通过VkPhysicalDevice可以创建VkDevice。VkDevice是主要的调用句柄，它在逻辑上与GPU相联系。它相当于GL Context或者D3D11 Device。


>N.B. Each of these is a 1:many relationship. A VkInstance can have many VkPhysicalDevices, a VkPhysicalDevice can have many VkDevices. In Vulkan 1.0, there is no cross-GPU activity, but you can bet this will come in the future though.

> 一个VkInstance可以有多个VkPhysicalDevice，一个VkPhysicalDevice可以有多个VkDevice。在Vulkan1.0，跨GPU的调用还未实现（将来会）。

>I'm hand waving some book-keeping details, Vulkan in general is quite lengthy in setup due to its explicit nature and this is a summary not an implementation guide. The overall picture is that your initialisation mostly looks like vkCreateInstance() → vkEnumeratePhysicalDevices() → vkCreateDevice(). For a quick and dirty hello world triangle program, you can do just that and pick the first physical device, then come back to it once you want error reporting & validation, enabling optional device features, etc.

	
大概的初始化调用就像是这样：`vkCreateInstance()` → `vkEnumeratePhysicalDevices()` → `vkCreateDevice()`	。对于一个简陋的HelloTriangle程序来说，你只需要简单地将第一个物理设备作为主要的VkDevice，然后打开这个设备的相应属性（错误上报、API调用检查）进行开发就行了。

## Images和Buffers 

---

>Now that we have a VkDevice we can start creating pretty much every other resource type (a few have further dependencies on other objects), for example VkImage and VkBuffer.

既然我们有了VkDevice，我们可以开始创建任意类型的资源，比如VkImage和VkBuffer。

>For GL people, one kind of new concept is that you must declare at creation time how an image will be used. You provide a bit field, with each bit indicating a certain type of usage - color attachment, or sampled image in shader, or image load/store, etc.

相对GL来说，使用Vulkan时，你必须在创建Image之前声明Image的用法。你可以设定bit表示Image的使用类型－Color Attachment、Sampled Image、或者Image load/store。

>You also specify the tiling for the image - LINEAR or OPTIMAL. This specifies the tiling/swizzling layout for the image data in memory. OPTIMAL tiled images are opaquely tiled, LINEAR are laid out just as you expect. This affects whether the image data is directly readable/writable, as well as format support - drivers report image support in terms of 'what image types are supported in OPTIMAL tiling, and what image types are supported in LINEAR'. Be prepared for very limited LINEAR support.

你也可以指定Image的Tiling模式－Linear或者Optimal。这设置了Image在内存中的布局。这影响Image数据是否可读可写。

>Buffers are similar and more straightforward, you give them a size and a usage and that's about it.

Buffer类似并更加直接，你指定了大小和用处。

>Images aren't used directly, so you will have to create a VkImageView - this is familiar to D3D11 people. Unlike GL texture views, image views are mandatory but are the same idea - a description of what array slices or mip levels are visible to wherever the image view is used, and optionally a different (but compatible) format (like aliasing a UNORM texture as UINT).
Buffers are usually used directly as they're just a block of memory, but if you want to use them as a texel buffer in a shader, you need to provide a VkBufferView.

Image并不是直接使用的，所以你将要创建VkImageView－这和D3D11类似。不像GLTextureView，Vulkan的ImageView是强制性的，但同样的想法-用于描述将数组切片或MIPLevel暴露给ImageView使用的地方，以及可选的不同（但兼容）格式（如将UNORM格式的纹理作为UINT使用）。

Buffer通常直接使用，因为它仅仅是一块内存，但如果你想将他们在Shader中作为TextureBuffer使用，你需要提供一个VkBufferView。

## GPU内存分配

---

>Those buffers and images can't be used immediately after creation as no memory has been allocated for them. This step is up to you.

刚创建的Buffer和Image并不能立即使用，因为我们并没有为他们分配内存。

>Available memory is exposed to applications by the vkGetPhysicalDeviceMemoryProperties(). It reports one or more memory heaps of given sizes, and one or more memory types with given properties. Each memory type comes from one heap - so a typical example for a discrete GPU on a PC would be two heaps - one for system RAM, and one for GPU RAM, and multiple memory types from each.

我们可以通过调用`vkGetPhysicalDeviceMemoryProperties`查询应用可使用的内存。它会返回请求大小的一个或多个内存堆，或者请求属性的一种或多种内存类型。每种内存类型来自于一个内存堆 － 因此，一个典例就是PC上的一个独立显卡将会有两个堆 － 一个是系统内存，另一个是GPU内存，并且他们各自拥有多种内存类型。

>The memory types have different properties. Some will be CPU visible or not, coherent between GPU and CPU access, cached or uncached, etc. You can find out all of these properties by querying from the physical device. This allows you to choose the memory type you want. E.g. staging resources will need to be in host visible memory, but your images you render to will want to be in device local memory for optimal use. However there is an additional restriction on memory selection that we'll get to in the next section.

内存类型有不同属性。一些内存可以被CPU访问或者不行、GPU和CPU访问一致、有缓存或者无缓存等等。

>To allocate memory you call vkAllocateMemory() which requires your VkDevice handle and a description structure. The structure dictates which type of memory to allocate from which heap and how much to allocate, and returns a VkDeviceMemory handle.

通过调用`vkAllocateMemory()`可以分配内存，但它需要VkDevice句柄和描述结构体。

>Host visible memory can be mapped for update - vkMapMemory()/vkUnmapMemory() are familiar functions. All maps are by definition persistent, and as long as you synchronise it's legal to have memory mapped while in use by the GPU.

HostVisibleMemory是可以通过Map方式来完成数据的更新的（`vkMapMemory()`/`vkUnmapMemory()`）。

>GL people will be familiar with the concept, but to explain for D3D11 people - the pointers returned by vkMapMemory() can be held and even written to by the CPU while the GPU is using them. These 'persistent' maps are perfectly valid as long as you obey the rules and make sure to synchronise access so that the CPU isn't writing to parts of the memory allocation that the GPU is using (see later).

GL使用者应该熟悉这个概念，但解释给D3D11的用户，vkMapMemory返回的指针可以被hold住被CPU写入当GPU正在使用它们。这些持久化的映射是完全正确的只要你遵守规则并且确定同步了内存访问。

>This is a little outside the scope of this guide but I'm going to mention it any chance I get - for the purposes of debugging, persistent maps of non-coherent memory with explicit region flushes will be much more efficient/fast than coherent memory. The reason being that for coherent memory the debugger must jump through hoops to detect and track changes, but the explicit flushes of non-coherent memory provide nice markup of modifications.
In RenderDoc to help out with this, if you flush a memory region then the tool assumes you will flush for every write, and turns off the expensive hoop-jumping to track coherent memory. That way even if the only memory available is coherent, then you can get efficient debugging.

这有点出了这个教程，但是我一旦有机会就会提及它。

## 内存绑定

---

>Each VkBuffer or VkImage, depending on its properties like usage flags and tiling mode (remember that one?) will report their memory requirements to you via vkGetBufferMemoryRequirements or vkGetImageMemoryRequirements.

通过调用`vkGetBufferMemoryRequirements/vkGetImageMemoryRequirements`，你可以知道`VkBuffer/VkImage`的内存需求和类型。

>The reported size requirement will account for padding for alignment between mips, hidden meta-data, and anything else needed for the total allocation. The requirements also include a bitmask of the memory types that are compatible with this particular resource. The obvious restrictions kick in here: that OPTIMAL tiling color attachment image will report that only DEVICE_LOCAL memory types are compatible, and it will be invalid to try to bind some HOST_VISIBLE memory.

获取到的内存大小是包括了Mips，隐藏的元数据以及其他需要的对象之间内存对齐、或者铺垫。要求的东西也包含兼容该资源内存类型的BitmapMask。这里有个明显的限制就是Optimal的ColorAttach Image将只能使用DeviceLocal的内存，如果尝试绑定HostVisible内存将是不正确的。

>The memory type requirements generally won't vary if you have the same kind of image or buffer. For example if you know that optimally tiled images can go in memory type 3, you can allocate all of them from the same place. You will only have to check the size and alignment requirements per-image. Read the spec for the exact guarantee here!

如果你有相同类型的Image或者Buffer，内存类型的要求通常不会太不一样。例如，如果你知道最优平铺Image可以使用内存类型3，你可以从相同的地方分配它们。你只需要对每个Image检查大小和对齐要求。为了准确地保证去读规范吧。

>Note the memory allocation is by no means 1:1. You can allocate a large amount of memory and as long as you obey the above restrictions you can place several images or buffers in it at different offsets. The requirements include an alignment if you are placing the resource at a non-zero offset. In fact you will definitely want to do this in any real application, as there are limits on the total number of allocations allowed.
There is an additional alignment requirement bufferImageGranularity - a minimum separation required between memory used for a VkImage and memory used for a VkBuffer in the same VkDeviceMemory. Read the spec for more details, but this mostly boils down to an effective page size, and requirement that each page is only used for one type of resource.

注意，内存分配不是1:1的。

>Once you have the right memory type and size and alignment, you can bind it with vkBindBufferMemory or vkBindImageMemory. This binding is immutable, and must happen before you start using the buffer or image.

一旦你有了正确的内存类型、大小和对齐，你可以通过vkBindBufferMemory或者vkBindImageMemory来绑定内存。这个绑定是固定的，并且发生在你开始使用Buffer或者Image之前。

## 命令缓冲和提交

---

> Work is explicitly recorded to and submitted from a VkCommandBuffer.

执行工作是通过VkCommandBuffer显式地记录和提交来完成。

> A VkCommandBuffer isn't created directly, it is allocated from a VkCommandPool. This allows for better threading behaviour since command buffers and command pools must be externally synchronised (see later). You can have a pool per thread and vkAllocateCommandBuffers()/vkFreeCommandBuffers() command buffers from it without heavy locking.


VkCommandBuffer并不是直接创建的，它是从VkCommandPool中分配出来的。因为命令缓冲和命令缓冲池需要另外同步，所以这样达到更好地多线程行为。你可以在每个线程创建命令缓冲池，然后无锁地分配(`vkAllocateCommandBuffers()/vkFreeCommandBuffers()`)命令缓冲。

> Once you have a VkCommandBuffer you begin recording, issue all your GPU commands into it *hand waving goes here* and end recording.

创建完VkCommandBuffer后就可以开始录制GPU命令。

Command buffers are submitted to a VkQueue. The notion of queues are how work becomes serialised to be passed to the GPU. A VkPhysicalDevice (remember way back? The GPU handle) can report a number of queue families with different capabilities. e.g. a graphics queue family and a compute-only queue family. When you create your device you ask for a certain number of queues from each family, and then you can enumerate them from the device after creation with vkGetDeviceQueue().

> I'm going to focus on having just a single do-everything VkQueue as the simple case, since multiple queues must be synchronised against each other as they can run out of order or in parallel to each other. Be aware that some implementations might require you to use a separate queue for swapchain presentation - I think chances are that most won't, but you have to account for this. Again, read the spec for details!

我将聚焦于用一个可以做任何事情的VkQueue作为一个简单例子，因为多队列必须相互同步来达到并行执行不影响顺序。有些Vulkan的实现可能要求你用单独的队列来完成swapchain的展示。


> You can vkQueueSubmit() several command buffers at once to the queue and they will be executed in turn. Nominally this defines the order of execution but remember that Vulkan has very specific ordering guarantees - mostly about what work can overlap rather than wholesale rearrangement - so take care to read the spec to make sure you synchronise everything correctly.

你可以调用`vkQueueSubmit()`一次性提交若干个命令缓冲，然后他们就会顺序的被执行。表面上这样会确定命令执行的顺序，但是要记得Vulkan有很明确的顺序保证 － 关于那些工作可以被覆盖，或者全部重新排序，所以要仔细阅读规范以确定正确的同步了所有对象。

## Shaders和PSO

---

>The reasoning behind moving to monolithic PSOs is well trodden by now so I won't go over it.

整体转移到PSO的原因已经很好的踏出去了，所以我这里就不再重复了。

>A Vulkan VkPipeline bakes in a lot of state, but allows specific parts of the fixed function pipeline to be set dynamically: Things like viewport, stencil masks and refs, blend constants, etc. A full list as ever is in the spec. When you call vkCreateGraphicsPipelines(), you choose which states will be dynamic, and the others are taken from values specified in the PSO creation info.

Vulkan的管线烘焙了许多状态在里面，但是允许固定管线的特定部分动态改变，像视口、蒙板、混合等等。规范里有整个管线包含的状态。当你调用vkCreateGraphicsPipelines的时候，你需要选择哪部分变成动态，其他的全来自PSO创建信息。

>You can optionally specify a VkPipelineCache at creation time. This allows you to compile a whole bunch of pipelines and then call vkGetPipelineCacheData() to save the blob of data to disk. Next time you can prepopulate the cache to save on PSO creation time. The expected caveats apply - there is versioning to be aware of so you can't load out of date or incorrect caches.

你也可以选择性地在创建的时候连Cache一起创建。这将允许你编译整个管线并通过vkGetPipelineCacheData来保存数据到磁盘。下次直接用cache创建管线以缩短创建时间。

>Shaders are specified as SPIR-V. This has already been discussed much better elsewhere, so I will just say that you create a VkShaderModule from a SPIR-V module, which could contain several entry points, and at pipeline creation time you chose one particular entry point.

着色器指定为SPIR－V格式。这已经在其他的地方充分地讨论了，所以我只说从SPIR－V创建一个VkShaderModule（包含多个入口，在创建期间指定好函数入口）。

>The easiest way to get some SPIR-V for testing is with the reference compiler glslang, but other front-ends are available, as well as LLVM → SPIR-V support.

测试SPIR－V最简单的方法就是使用参考编译器glslang，其他编译前端也OK，也有LLVM－>SPIR-V的支持。

## 绑定模型

---

> To establish a point of reference, let's roughly outline D3D11's binding model. GL's is quite similar.

为了有个参考点，我们粗略的过下D3D11的绑定模型。GL是非常相似的。

> Each shader stage has its own namespace, so pixel shader texture binding 0 is not vertex shader texture binding 0.
Each resource type is namespaced apart, so constant buffer binding 0 is definitely not the same as texture binding 0.
Resources are individually bound and unbound to slots (or at best in contiguous batches).
In Vulkan, the base binding unit is a descriptor. A descriptor is an opaque representation that stores 'one bind'. This could be an image, a sampler, a uniform/constant buffer, etc. It could also be arrayed - so you can have an array of images that can be different sizes etc, as long as they are all 2D floating point images.

每个着色阶段都有它自己的名字空间，因此像素着色器的纹理绑定单元0不是顶点着色器的纹理绑定单元0。每种资源类型都有独立的名字空间，因此常量缓冲（ConstantBuffer）绑定的单元0一定不同于纹理（Texture）绑定的单元0。资源是独立地绑定到资源槽（Slot）或者从Slot解绑（最好是按批次处理）。Vulkan中基础的绑定单元是描述符（Descriptor）。


> Descriptors aren't bound individually, they are bound in blocks in a VkDescriptorSet which each have a particular VkDescriptorSetLayout. The VkDescriptorSetLayout describes the types of the individual bindings in each VkDescriptorSet.

描述符并不是独立绑定的，它是绑定在VkDescriptorSet（每个Set又有VkDescriptorSetLayout）的Block中。VkDescriptorSetLayout描述了Set中每个绑定的类型。

> The easiest way I find to think about this is consider VkDescriptorSetLayout as being like a C struct type - it describes some members, each member having an opaque type (constant buffer, load/store image, etc). The VkDescriptorSet is a specific instance of that type - and each member in the VkDescriptorSet is a binding you can update with whichever resource you want it to contain.

我能想到的描述`VkDescriptorSetLayout`最简单的方法就把它当成一个C结构体，它描述了一些成员变量，并且每个成员都有明确的类型（constant buffer，load/store image，等等）。一个`VkDescriptorSet`是一个特定的实例，Set里每个成员是一个可更新的资源绑定。

> This is roughly how you create the objects too. You pass a list of the types, array sizes and bindings to Vulkan to create a VkDescriptorSetLayout, then you can allocate VkDescriptorSets with that layout from a VkDescriptorPool. The pool acts the same way as VkCommandPool, to let you allocate descriptors on different threads more efficiently by having a pool per thread.

这是粗略的创建方法。你通过传递一串类型、数组和绑定来创建`VkDescriptorSetLayout`。

```c
VkDescriptorSetLayoutBinding bindings[] = {
	// binding 0 is a UBO, array size 1, visible to all stages
	{ 0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, 1, VK_SHADER_STAGE_ALL_GRAPHICS, NULL },
	// binding 1 is a sampler, array size 1, visible to all stages
	{ 1, VK_DESCRIPTOR_TYPE_SAMPLER,        1, VK_SHADER_STAGE_ALL_GRAPHICS, NULL },
	// binding 5 is an image, array size 10, visible only to fragment shader
	{ 5, VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE, 10, VK_SHADER_STAGE_FRAGMENT_BIT, NULL },
};
```

Example C++ outlining creation of a descriptor set layout
>Once you have a descriptor set, you can update it directly to put specific values in the bindings, and also copy between different descriptor sets.

一旦有了描述符集合，你可以直接更新绑定单元的值，也可以在不同的描述符集合之间拷贝。

>When creating a pipeline, you specify N VkDescriptorSetLayouts for use in a VkPipelineLayout. Then when binding, you have to bind matching VkDescriptorSets of those layouts. The sets can update and be bound at different frequencies, which allows grouping all resources by frequency of update.

创建管线的时候，你需要在管线布局中指定N个描述符集合布局。绑定的时时候，绑定需要匹配描述符集合的布局。这些集合可以以不同的频率更新和绑定，允许通过更新频率进行资源的分组。

>To extend the above analogy, this defines the pipeline as something like a function, and it can take some number of structs as arguments. When creating the pipeline you declare the types (VkDescriptorSetLayouts) of each argument, and when binding the pipeline you pass specific instances of those types (VkDescriptorSets).

当我们创建管线时，我们就声明参数的所有类型（VkDescriptorSetLayouts），并且当我们开始绑定时，你可以传实例过去。

The other side of the equation is fairly simple - instead of having shader or type namespaced bindings in your shader code, each resource in the shader simply says which descriptor set and binding it pulls from. This matches the descriptor set layout you created.

```glsl
#version 430

layout(set = 0, binding = 0) uniform MyUniformBufferType {
// ...
} MyUniformBufferInstance;

// note in the C++ sample above, this is just a sampler - not a combined image+sampler
// as is typical in GL.
layout(set = 0, binding = 1) sampler MySampler;

layout(set = 0, binding = 5) uniform image2D MyImages[10];

```
>Example GLSL showing bindings

GLSL绑定的例子

## 同步

---

>I'm going to hand wave a lot in this section because the specific things you need to synchronise get complicated and long-winded fast, and I'm just going to focus on what synchronisation is available and leave the details of what you need to synchronise to reading of specs or more in-depth documents.

我将在本节阐述许多因为一些需要同步的东西比较复杂和冗长，所以我将具体说明什么样的同步是可行的，关于细节，你需要阅读规范或者更深度的文档。

>This is probably the hardest part of Vulkan to get right, especially since missing synchronisation might not necessarily break anything when you run it!

这大概是Vulkan里最难搞正确的部分，尤其是，错误的同步可能在程序运行的时候不会中断渲染。

>Several types of objects must be 'externally synchronised'. In fact I've used that phrase before in this post. The meaning is basically that if you try to use the same VkQueue on two different threads, there's no internal locking so it will crash - it's up to you to 'externally synchronise' access to that VkQueue.

有些对象必须“在外面被同步”。这意味着如果你想在两个不同线程使用相同队列，这并没有互斥锁的保证所以会发生崩溃－这取决于你是否要在外面同步。

>For the exact requirements of what objects must be externally synchronised when you should check the spec, but as a rule you can use VkDevice for creation functions freely - it is locked for allocation sake - but things like recording and submitting commands must be synchronised.

那些需要在外面被同步的对象在规范里都有写。但作为规则，你可以使用VkDevice自由地创建－它会被锁上用作分配－但是像录制和提交命令则必须被同步。

>N.B. There is no explicit or implicit ref counting of any object - you can't destroy anything until you are sure it is never going to be used again by either the CPU or the GPU.
Vulkan has VkEvent, VkSemaphore and VkFence which can be used for efficient CPU-GPU and GPU-GPU synchronisation. They work as you expect so you can look up the precise use etc yourself, but there are no surprises here. Be careful that you do use synchronisation though, as there are few ordering guarantees in the spec itself.

>Pipeline barriers are a new concept, that are used in general terms for ensuring ordering of GPU-side operations where necessary, for example ensuring that results from one operation are complete before another operation starts, or that all work of one type finishes on a resource before it's used for work of another type.

>There are three types of barrier - VkMemoryBarrier, VkBufferMemoryBarrier and VkImageMemoryBarrier. A VkMemoryBarrier applies to memory globally, and the other two apply to specific resources (and subsections of those resources).

>The barrier takes a bit field of different memory access types to specify what operations on each side of the barrier should be synchronised against the other. A simple example of this would be "this VkImageMemoryBarrier has srcAccessMask = ACCESS_COLOR_ATTACHMENT_WRITE and dstAccessMask = ACCESS_SHADER_READ", which indicates that all color writes should finish before any shader reads begin - without this barrier in place, you could read stale data.

### Image layouts

>Image barriers have one additional property - images exist in states called image layouts. VkImageMemoryBarrier can specify a transition from one layout to another. The layout must match how the image is used at any time. There is a GENERAL layout which is legal to use for anything but might not be optimal, and there are optimal layouts for color attachment, depth attachment, shader sampling, etc.

ImageBarrier有一个附加的属性ImageLayout（image的使用状态）。VkImageMemoryBarrier可以指定从一个ImageLayout到另一个ImageLayout的转换过程。Layout任何时候都要匹配Image的使用方法。GENERAL的Layout可以用于任何Image，但可能并不是最优，有许多其他优化过的Layout用于ColorAttachment，DepthAttachment，ShaderSampler等等。

>Images begin in either the UNDEFINED or PREINITIALIZED state (you can choose). The latter is useful for populating an image with data before use, as the UNDEFINED layout has undefined contents - a transition from UNDEFINED to GENERAL may lose the contents, but PREINITIALIZED to GENERAL won't. Neither initial layout is valid for use by the GPU, so at minimum after creation an image needs to be transitioned into some appropriate state.

Image开始的状态可以是UNDEFINED或者PREINITIALIZED。后者可以用于创建有数据的Image对象，因为UNDEFINED Layout是没有内容的，当Image从UNDEFINED到GENERAL转换的时候会丢失数据内容，但是PREINITIALIZED到GENERAL不会。无论初始的Layout是否正确，Image创建之后需要转换到某个适当的状态。

>Usually you have to specify the previous and new layouts accurately, but it is always valid to transition from UNDEFINED to another layout. This basically means 'I don't care what the image was like before, throw it away and use it like this'.

通常你需要准确地指定Layout，但是从UNDEFINED到其他的Layout总会是正确的转换。这就像“我不关心他以前是什么样的，现在这样用就行了”。

## RenderPass

---

>A VkRenderpass is Vulkan's way of more explicitly denoting how your rendering happens, rather than letting you render into then sample images at will. More information about how the frame is structured will aid everyone, but primarily this is to aid tile based renderers so that they have a direct notion of where rendering on a given target happens and what dependencies there are between passes, to avoid leaving tile memory as much as possible.

相比于你按意愿地渲染到目标Image，VkRenderPass是Vulkan更为显式地表示渲染过程。

>N.B. Because I primarily work on desktops (and for brevity & simplicity) I'm not mentioning a couple of optional things you can do that aren't commonly suited to desktop GPUs like input and transient attachments. As always, read the spec :).
The first building block is a VkFramebuffer, which is a set of VkImageViews. This is not necessarily the same as the classic idea of a framebuffer as the particular images you are rendering to at any given point, as it can contain potentially more images than you ever render to at once.

要注意的是，因为我的工作主要集中在桌面平台（为了简化），我将不会提到非桌面平台共有的一些可选的东西，比如暂存的Attachment（一如既往地阅读spec）。
第一个组成部分就是Framebuffer，它其实就是一堆VkImageView。它并不和传统的Framebuffer一样作为特殊的Image用于渲染到特定目标，它可以在一次渲染过程中包含更多Image。

>A VkRenderPass consists of a series of subpasses. In your simple triangle case and possibly in many other cases, this will just be one subpass. For now, let's just consider that case. The subpass selects some of the framebuffer attachments as color attachments and maybe one as a depth-stencil attachment. If you have multiple subpasses, this is where you might have different subsets used in each subpass - sometimes as output and sometimes as input.

一个VkRenderPass由多个子pass组成。在这个简单的三角形测例以及其他许多可能场景，一般只有一个子pass。现在来看，我们只考虑这种情况。子pass选择一些attachment作为颜色目标，另外一些作为深度和模版目标。如果你有多个子pass，每个子pass将有不同的集合，一些用于输入，一些用于输出。

>Drawing commands can only happen inside a VkRenderPass, and some commands such as copies clears can only happen outside a VkRenderPass. Some commands such as state binding can happen inside or outside at will. Consult the spec to see which commands are which.

绘制的指令调用只能在一个VkRenderPass中使用，如果是copy相关指令则只能在外面使用。状态绑定的指令则可以按意愿在里面或在外面调用。规范中有详细的介绍。

>Subpasses do not inherit state at all, so each time you start a VkRenderPass or move to a new subpass you have to bind/set all of the state. Subpasses also specify an action both for loading and storing each attachment. This allows you to say 'the depth should be cleared to 1.0, but the color can be initialised to garbage for all I care - I'm going to fully overwrite the screen in this pass'. Again, this can provide useful optimisation information that the driver no longer has to guess.

子pass并不继承状态，所以每次开始一个renderpass或者切换到一个新的子pass，你都需要重新绑定状态。

>The last consideration is compatibility between these different objects. When you create a VkRenderPass (and all of its subpasses) you don't reference anything else, but you do specify both the format and use of all attachments. Then when you create a VkFramebuffer you must choose a VkRenderPass that it will be used with. This doesn't have to be the exact instance that you will later use, but it does have to be compatible - the same number and format of attachments. Similarly when creating a VkPipeline you have to specify the VkRenderPass and subpass that it will be used with, again not having to be identical but required to be compatible.

最后需要考虑的是这些不同对象间的兼容性。当你创建一个没有任何reference的VkRenderPass，但是制定了格式和所有attachment的用法，然后创建VkFramebuffer时，你必须选择一个VkRenderPass。这不要求一定是那个Renderpass实例，但一定要兼容－attachment的数量和格式要一样。类似的，创建VkPipeline时用的RenderPass和Subpass，同样不要求一样的实例，但是要兼容。

>There are more complexities to consider if you have multiple subpasses within your render pass, as you have to declare barriers and dependencies between them, and annotate which attachments must be used for what. Again, if you're looking into that read the spec.

如果你的renderpass有多个子pass，你需要考虑更复杂的情况了，你不得不在子pass之间声明屏障和依赖关系，并说明attachment的用途。

## Backbuffers以及显示

---

>I'm only going to talk about this fairly briefly because not only is it platform-specific but it's fairly straightforward.

我即将简单讲述Backbuffer显示的过程，因为这不仅仅是平台特定的但是更加直接。

>Note that Vulkan exposes native window system integration via extensions, so you will have to request them explicitly when you create your VkInstance and VkDevice.
To start with, you create a VkSurfaceKHR from whatever native windowing information is needed.

注意Vulkan可以通过扩展暴露本地Windows系统，所以当你创建VkInstance和VkDevice时你将不得不和它打交道。该开始，你将创建VkSurface。

>Once you have a surface you can create a VkSwapchainKHR for that surface. You'll need to query for things like what formats are supported on that surface, how many backbuffers you can have in the chain, etc.

一旦你创建了surface，你就可以在surface上创建VkSwapchainKHR。你将要查询surface支持的格式，最多有多少个nackbuffer，等等。

>You can then obtain the actual images in the VkSwapchainKHR via vkGetSwapchainImagesKHR(). These are normal VkImage handles, but you don't control their creation or memory binding - that's all done for you. You will have to create an VkImageView each though.

你可以然后通过vkGetSwapchainImagesKHR()获取VkSwapChainKHR里的实际Image。这些是普通的VkImage句柄，但是你不能控制它们的创建和内存绑定－这些步骤都已经完成了。你将要为每个Image创建VkImageView。

>When you want to render to one of the images in the swapchain, you can call vkAcquireNextImageKHR() that will return to you the index of the next image in the chain. You can render to it and then call vkQueuePresentKHR() with the same index to have it presented to the display.

当你想渲染Swapchain中的一个Image时，你可以调用vkAcquireNextImageKHR()，它将返回下一张Image的索引，你可以在它上面渲染然后调用vkQueuePresentKHR()显示出来。

>There are many more subtleties and details if you want to get really optimal use out of the swapchain, but for the dead-simple hello world case, the above suffices.

如果你想最佳地使用SwapChain，还有许多其他的细节，但这个简单的hello world示例已经差不多了。

## 总结

---


## 附录（伪代码示例）

---

```cpp
#include <vulkan/vulkan.h>

// vulkan应用的伪代码看起来就是这样。这里忽略了大部分的创建结构体和所有的同步和错误检查。
// 这不是一个复制粘贴教程！
void DoVulkanRendering()
{
  const char *extensionNames[] = { "VK_KHR_surface", "VK_KHR_win32_surface" };

  // 后面的结构体不会详细说明，这个只是用来说明。
  // Application info is optional (you can specify application/engine name and version)
  // Note we activate the WSI instance extensions, provided by the ICD to
  // allow us to create a surface (win32 is an example, there's also xcb/xlib/etc)
  VkInstanceCreateInfo instanceCreateInfo = {
    VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO, // VkStructureType sType;
    NULL,                                   // const void* pNext;

    0,                                      // VkInstanceCreateFlags flags;

    NULL,                                   // const VkApplicationInfo* pApplicationInfo;

    0,                                      // uint32_t enabledLayerNameCount;
    NULL,                                   // const char* const* ppEnabledLayerNames;

    2,                                      // uint32_t enabledExtensionNameCount;
    extensionNames,                         // const char* const* ppEnabledExtensionNames;
  };

  VkInstance inst;
  vkCreateInstance(&instanceCreateInfo, NULL, &inst);

  // 枚举调用的最后参数设为NULL来获取物理设备数量。
  // 重新再调用，获取设备句柄。
  VkPhysicalDevice phys[4]; uint32_t physCount = 4;
  vkEnumeratePhysicalDevices(inst, &physCount, phys);

  VkDeviceCreateInfo deviceCreateInfo = {
    // 略过
  };

  VkDevice dev;
  vkCreateDevice(phys[0], &deviceCreateInfo, NULL, &dev);

  // 通过 vkGetInstanceProcAddr 获取 vkCreateWin32SurfaceKHR 扩展函数指针
	VkWin32SurfaceCreateInfoKHR surfaceCreateInfo = {
		// HINSTANCE, HWND, etc
	};
  VkSurfaceKHR surf;
  vkCreateWin32SurfaceKHR(inst, &surfaceCreateInfo, NULL, &surf);

  VkSwapchainCreateInfoKHR swapCreateInfo = {
    // surf goes in here
  };
  VkSwapchainKHR swap;
  vkCreateSwapchainKHR(dev, &swapCreateInfo, NULL, &swap);

  // Again this should be properly enumerated
  VkImage images[4]; uint32_t swapCount;
  vkGetSwapchainImagesKHR(dev, swap, &swapCount, images);

  // 这里需要同步
  uint32_t currentSwapImage;
  vkAcquireNextImageKHR(dev, swap, UINT64_MAX, presentCompleteSemaphore, NULL, &currentSwapImage);

  // 传递creationInfo来创建ImageView
  VkImageView backbufferView;
  vkCreateImageView(dev, &backbufferViewCreateInfo, NULL, &backbufferView);

  VkQueue queue;
  vkGetDeviceQueue(dev, 0, 0, &queue);

  VkRenderPassCreateInfo renderpassCreateInfo = {
    // here you will specify the total list of attachments
    // (which in this case is just one, that's e.g. R8G8B8A8_UNORM)
    // as well as describe a single subpass, using that attachment
    // for color and with no depth-stencil attachment
  };

  VkRenderPass renderpass;
  vkCreateRenderPass(dev, &renderpassCreateInfo, NULL, &renderpass);

  VkFramebufferCreateInfo framebufferCreateInfo = {
    // include backbufferView here to render to, and renderpass to be
    // compatible with.
  };

  VkFramebuffer framebuffer;
  vkCreateFramebuffer(dev, &framebufferCreateInfo, NULL, &framebuffer);

  VkDescriptorSetLayoutCreateInfo descSetLayoutCreateInfo = {
    // whatever we want to match our shader. e.g. Binding 0 = UBO for a simple
    // case with just a vertex shader UBO with transform data.
  };

  VkDescriptorSetLayout descSetLayout;
  vkCreateDescriptorSetLayout(dev, &descSetLayoutCreateInfo, NULL, &descSetLayout);

  VkPipelineCreateInfo pipeLayoutCreateInfo = {
    // 一个DescSetLayout维护一个descriptorSet
  };

  VkPipelineLayout pipeLayout;
  vkCreatePipelineLayout(dev, &pipeLayoutCreateInfo, NULL, &pipeLayout);

  // 上传SPIR-V shaders
  VkShaderModule vertModule, fragModule;
  vkCreateShaderModule(dev, &vertModuleInfoWithSPIRV, NULL, &vertModule);
  vkCreateShaderModule(dev, &fragModuleInfoWithSPIRV, NULL, &fragModule);

  VkGraphicsPipelineCreateInfo pipeCreateInfo = {
    // 这里有许多结构体需要完整的填充。
    // 它将指向着色器、管线布局、还有RenderPass。
  };

  VkPipeline pipeline;
  vkCreateGraphicsPipelines(dev, NULL, 1, &pipeCreateInfo, NULL, &pipeline);

  VkDescriptorPoolCreateInfo descPoolCreateInfo = {
    // the creation info states how many descriptor sets are in this pool
  };

  VkDescriptorPool descPool;
  vkCreateDescriptorPool(dev, &descPoolCreateInfo, NULL, &descPool);

  VkDescriptorSetAllocateInfo descAllocInfo = {
    // from pool descPool, with layout descSetLayout
  };

  VkDescriptorSet descSet;
  vkAllocateDescriptorSets(dev, &descAllocInfo, &descSet);

  VkBufferCreateInfo bufferCreateInfo = {
    // buffer for uniform usage, of appropriate size
  };

  VkMemoryAllocateInfo memAllocInfo = {
    // skipping querying for memory requirements. Let's assume the buffer
    // can be placed in host visible memory.
  };
  VkBuffer buffer;
  VkDeviceMemory memory;
  vkCreateBuffer(dev, &bufferCreateInfo, NULL, &buffer);
  vkAllocateMemory(dev, &memAllocInfo, NULL, &memory);
  vkBindBufferMemory(dev, buffer, memory, 0);

  void *data = NULL;
  vkMapMemory(dev, memory, 0, VK_WHOLE_SIZE, 0, &data);
  // fill data pointer with lovely transform goodness
  vkUnmapMemory(dev, memory);

  VkWriteDescriptorSet descriptorWrite = {
    // write the details of our UBO buffer into binding 0
  };

  vkUpdateDescriptorSets(dev, 1, &descriptorWrite, 0, NULL);

  // 最后我们可以渲染了!
  // ...
  // 差不多就这样.

  VkCommandPoolCreateInfo commandPoolCreateInfo = {
    // hehe
  };

  VkCommandPool commandPool;
  vkCreateCommandPool(dev, &commandPoolCreateInfo, NULL, &commandPool);

  VkCommandBufferAllocateInfo commandAllocInfo = {
    // 从commandPool分配
  };
  VkCommandBuffer cmd;
  vkAllocateCommandBuffers(dev, &commandAllocInfo, &cmd);

  // 现在可以渲染了

  vkBeginCommandBuffer(cmd, &cmdBeginInfo);
  vkCmdBeginRenderPass(cmd, &renderpassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
  // 绑定管线
  vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
  // 绑定描述符
  vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
                          descSetLayout, 1, &descSet, 0, NULL);
  // 设置渲染视口
  vkCmdSetViewport(cmd, 1, &viewport);
  // 绘制三角形
  vkCmdDraw(cmd, 3, 1, 0, 0);
  vkCmdEndRenderPass(cmd);
  vkEndCommandBuffer(cmd);

  VkSubmitInfo submitInfo = {
    // this contains a reference to the above cmd to submit
  };

  vkQueueSubmit(queue, 1, &submitInfo, NULL);

  // 现在我们可以显示了
  VkPresentInfoKHR presentInfo = {
    // swap and currentSwapImage are used here
  };
  vkQueuePresentKHR(queue, &presentInfo);

  // 等待所有操作完成并销毁对象
}

```

## 参考

[1]:https://renderdoc.org/vulkan-in-30-minutes.html
[2]:https://developer.nvidia.com/engaging-voyage-vulkan
[3]:https://developer.nvidia.com/vulkan-memory-management
[4]:https://developer.nvidia.com/vulkan-shader-resource-binding

