# 30分钟认识Vulkan API

---

	

> 这篇文章是写给对已有的D3D11和GL比较熟悉并理解多线程、资源暂存、同步等概念但想进一步了解它们是如何以Vulkan实现的读者，因此本文会以Vulkan概念之旅结束。

文章并不追求易懂（为了解里面的晦涩内容，你应该去阅读Vulkan规范或者更深度的教程）。

本文是我第一次见到Vulkan写下的。

－ baldurk

## General

---

文末会介绍一个精简的伪代码来阐述显示一个三角形的大致步骤。

以下是一些简单的注意点，不适用其他部分：

* Vulkan是一个类似GL的C语言图形库
* 这个API是强类型的，不像GL用GLenum就搞定所有参数，Vk的枚举是类型分离的。
* 多数函数调用的参数都是结构体或者是嵌套结构体。
* VkAllocationCallbacks * 是许多vkCreate＊函数的参数，它用来接管内存分配，当然你也可以简单地传NULL。

> 注意：我并没有考虑任何错误异常处理，也不谈论query的实现限制。

## 初始化步骤

---

Vulkan API的初始化必须要创建实例（`VkInstance`）。Vulkan实例之间是相互独立的，所以你可以给不同的实例设置不同的属性（例如，是否激活validation layer和extensions）。


通过VkInstance可以检查GPU设备是否可用。Vulkan不一定是运行在GPU上，但我们把问题简单化。每个GPU都会提供一个句柄 - VkPhysicalDevice。你可以通过它来查询GPU厂商、属性（`vkGetPhysicalDeviceProperties`）、能力（`vkGetPhysicalDeviceFeatures`）等。


通过VkPhysicalDevice可以创建VkDevice。VkDevice是主要的调用句柄，它在逻辑上与GPU相联系。它相当于GL Context或者D3D11 Device。


> 一个VkInstance可以有多个VkPhysicalDevice，一个VkPhysicalDevice可以有多个VkDevice。在Vulkan1.0，跨GPU的调用还未实现（将来会）。
	
	
大概的初始化调用就像是这样：`vkCreateInstance()` → `vkEnumeratePhysicalDevices()` → `vkCreateDevice()`	。对于一个简陋的HelloTriangle程序来说，你只需要简单地将第一个物理设备作为主要的VkDevice，然后打开这个设备的相应属性（错误上报、API调用检查）进行开发就行了。

## Images和Buffers 

---

既然我们有了VkDevice，我们可以开始创建任意类型的资源，比如VkImage和VkBuffer。

相对GL来说，使用Vulkan时，你必须在创建Image之前声明Image的用法。你可以设定bit表示Image的使用类型－Color Attachment、Sampled Image、活着Image load/store


## GPU内存分配

---



## 内存绑定

---




## 命令缓冲和提交

---

执行工作是通过VkCommandBuffer显式地记录和提交来完成。


VkCommandBuffer并不是直接创建的，它是从VkCommandPool中分配出来的。

## Shaders和PSO

---

## 绑定模型

---

## 同步

---

## RenderPass

---

## CommandBuffer和提交

---

## Backbuffers以及显示

---

## 总结

---


## 附录（伪代码示例）

---

```cpp
#include <vulkan/vulkan.h>

// Pseudocode of what an application looks like. I've omitted most creation structures,
// almost all synchronisation and all error checking. This is not a copy-paste guide!
void DoVulkanRendering()
{
  const char *extensionNames[] = { "VK_KHR_surface", "VK_KHR_win32_surface" };

  // future structs will not be detailed, but this one is for illustration.
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

  // The enumeration pattern SHOULD be to call with last parameter NULL to
  // get the count, then call again to get the handles. For brevity, omitted
  VkPhysicalDevice phys[4]; uint32_t physCount = 4;
  vkEnumeratePhysicalDevices(inst, &physCount, phys);

  VkDeviceCreateInfo deviceCreateInfo = {
    // I said I was going to start omitting things!
  };

  VkDevice dev;
  vkCreateDevice(phys[0], &deviceCreateInfo, NULL, &dev);

  // fetch vkCreateWin32SurfaceKHR extension function pointer via vkGetInstanceProcAddr
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

  // Synchronisation is needed here!
  uint32_t currentSwapImage;
  vkAcquireNextImageKHR(dev, swap, UINT64_MAX, presentCompleteSemaphore, NULL, &currentSwapImage);

  // pass appropriate creation info to create view of image
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
    // one descriptor set, with layout descSetLayout
  };

  VkPipelineLayout pipeLayout;
  vkCreatePipelineLayout(dev, &pipeLayoutCreateInfo, NULL, &pipeLayout);

  // 上传SPIR-V shaders
  VkShaderModule vertModule, fragModule;
  vkCreateShaderModule(dev, &vertModuleInfoWithSPIRV, NULL, &vertModule);
  vkCreateShaderModule(dev, &fragModuleInfoWithSPIRV, NULL, &fragModule);

  VkGraphicsPipelineCreateInfo pipeCreateInfo = {
    // there are a LOT of sub-structures under here to fully specify
    // the PSO state. It will reference vertModule, fragModule and pipeLayout
    // as well as renderpass for compatibility
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

  // finally we can render something!
  // ...
  // Almost.

  VkCommandPoolCreateInfo commandPoolCreateInfo = {
    // nothing interesting
  };

  VkCommandPool commandPool;
  vkCreateCommandPool(dev, &commandPoolCreateInfo, NULL, &commandPool);

  VkCommandBufferAllocateInfo commandAllocInfo = {
    // allocate from commandPool
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

