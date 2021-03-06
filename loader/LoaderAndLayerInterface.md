# Vulkan加载器接口架构
此文翻译于 https://github.com/KhronosGroup/Vulkan-LoaderAndValidationLayers/blob/master/loader/LoaderAndLayerInterface.md

## Table of Contents
  * [Overview](#overview)
    * [Who Should Read This Document](#who-should-read-this-document)
    * [The Loader](#the-loader)
    * [Layers](#layers)
    * [Installable Client Drivers](#installable-client-drivers)
    * [Instance Versus Device](#instance-versus-device)
    * [Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains)

  * [Application Interface to the Loader](#application-interface-to-the-loader)
    * [Interfacing with Vulkan Functions](#interfacing-with-vulkan-functions)
    * [Application Layer Usage](#application-layer-usage)
    * [Application Usage of Extensions](#application-usage-of-extensions)

  * [Loader and Layer Interface](#loader-and-layer-interface)
    * [Layer Discovery](#layer-discovery)
    * [Layer Version Negotiation](#layer-version-negotiation)
    * [Layer Call Chains and Distributed Dispatch](#layer-call-chains-and-distributed-dispatch)
    * [Layer Unknown Physical Device Extensions](#layer-unknown-physical-device-extensions)
    * [Layer Intercept Requirements](#layer-intercept-requirements)
    * [Distributed Dispatching Requirements](#distributed-dispatching-requirements)
    * [Layer Conventions and Rules](#layer-conventions-and-rules)
    * [Layer Dispatch Initialization](#layer-dispatch-initialization)
    * [Example Code for CreateInstance](#example-code-for-createinstance)
    * [Example Code for CreateDevice](#example-code-for-createdevice)
    * [Meta-layers](#meta-layers)
    * [Special Considerations](#special-considerations)
    * [Layer Manifest File Format](#layer-manifest-file-format)
    * [Layer Library Versions](#layer-library-versions)

  * [Vulkan Installable Client Driver interface with the loader](#vulkan-installable-client-driver-interface-with-the-loader)
    * [ICD Discovery](#icd-discovery)
    * [ICD Manifest File Format](#icd-manifest-file-format)
    * [ICD Vulkan Entry-Point Discovery](#icd-vulkan-entry-point-discovery)
    * [ICD Unknown Physical Device Extensions](#icd-unknown-physical-device-extensions)
    * [ICD Dispatchable Object Creation](#icd-dispatchable-object-creation)
    * [Handling KHR Surface Objects in WSI Extensions](#handling-khr-surface-objects-in-wsi-extensions)
    * [Loader and ICD Interface Negotiation](#loader-and-icd-interface-negotiation)

  * [Table of Debug Environment Variables](#table-of-debug-environment-variables)
  * [Glossary of Terms](#glossary-of-terms)
 
## Overview

Vulkan is a layered architecture, made up of the following elements:
  * The Vulkan Application
  * [The Vulkan Loader](#the-loader)
  * [Vulkan Layers](#layers)
  * [Installable Client Drivers (ICDs)](#installable-client-drivers)

![High Level View of Loader](./images/high_level_loader.png)

The general concepts in this document are applicable to the loaders available
for Windows, Linux and Android based systems.


#### Who Should Read This Document

这份文档主要的目标读者是Vulkan应用开发者，驱动和layer开发者，然而本文的信息对于想要深入了解Vulkan运行时 lib的人都是有帮助的。


#### The Loader

应用程序处于加载器的一端，是加载器直接的上层。 应用程序中加载器的另外一侧是ICD，它控制了vulkan兼容的硬件。需要谨记的一点是Vulkan兼容的硬件可以基于图形的、基于计算的，或者二者兼有。 在应用程序和ICD之间，加载器可以插入多个可选的 layers，这些layer各自提供了某些功能。

加载器负责和各种layers打交道，并负责支持多GPU及驱动程序。任何Vulkan函数最终都会访问到各个模块：加载器、layers和ICDs。加载器肩负把Vulkan函数转发到合适layers和ICDs的重责。Vulkan对象模型允许加载器想调用链中插入layers，以便layers可以在ICD被调用之前处理Vulkan函数。

此文档旨在提供在这些概念之间必需的接口的概览。


##### Goals of the Loader

加载器设计之初衷如下：
 1. 在用户的计算机系统上支持多个Vulkan兼容的ICD，且互相之间不影响。
 2. 支持Vulkan 层，它是可选的模块，可以被应用程序、用户或者系统设定所启用。
 3. Impact the overall performance of a Vulkan application in the lowest
possible fashion.


#### Layers

Layers是增强Vulkan系统可选的组件。它们可以在已有的Vulkan函数从应用程序到硬件之间进行拦截、求值，甚至修改。
Layers通过libraries来实现，可以通过多种不同的方式被启用（包括应用程序的请求），在CreateInstance的时候被载入。
每一个layer可以选择hook（拦截）任何Vulkan函数，终造成vulkan函数被忽略或者增强。一个layer可以不拦截任何Vulkan函数。它可以选择拦截任何已知的的函数，或者可以选择只拦截一个函数。

layers能带来的特性案例可以包括如下几个：
 * 验证API的使用
 * 增强Vulkan API的跟踪和调试能力
 * 给应用程序的surfaces覆盖追加的内容

因为layers是可选的，你可以选择启用layers来调试你的应用程序，但是在发布产品的时候关闭任何layer。


#### Installable Client Drivers

Vulkan允许多个 Installable Client Drivers (ICDs) 的一个系统中共存，每一个支持一个或者多个物理设备（通过  `VkPhysicalDevice` 对象表示）。
加载器负责查找系统上可用的Vulkan ICDs。当给出一组可用的ICDs，加载器可以遍历所有可用的物理设备，并把这些信息返回给应用程序。


#### Instance Versus Device

在此文档中你可能一直看到一个非常重要的概念。很多函数，拓展和Vulkan中其他的东西都被分为两类：
 * Instance-related Objects
 * Device-related Objects


##### Instance-related Objects

一个Vulkan实例是一个高层次的构造，用来提供Vulkan系统层次的信息或者功能。和instance直接相关的Vulkan对象有：
 * `VkInstance`
 * `VkPhysicalDevice`

 一个instance函数的定义是任何把Instance列表的其中一个作为第一个参数，或者没有参数的函数。一些Vulkan Instance函数有：
 * `vkEnumerateInstanceExtensionProperties`
 * `vkEnumeratePhysicalDevices`
 * `vkCreateInstance`
 * `vkDestroyInstance`

你可以使用`vkGetInstanceProcAddr`查询Vulkan实例函数。`vkGetInstanceProcAddr` 可以被用来查询设备或者实例的入口函数（包括核心入口函数）。
返回的函数指针对实例以及在该实例下创建的对象（包括所有的l `VkDevice` 对象）来说是有效的。

同样，一个实例拓展是拓展了Vulkan语言的一系列实例函数。这些将在后续小节中讨论。


##### Device-related Objects

一个Vulkan设备，从另外一方面讲，是一个用来关联函数和用户系统上特定物理设备的逻辑标识符。
与设备直接关联的Vulkan数据结构有：
 * `VkDevice`
 * `VkQueue`
 * `VkCommandBuffer`
 * Any dispatchable object that is a child of a one of the above.

 一个设备函数是任何把设备对象作为第一个参数的函数。一些设备函数有：
 * `vkQueueSubmit`
 * `vkBeginCommandBuffer`
 * `vkCreateEvent`

你可以使用 `vkGetInstanceProcAddr` 或`vkGetDeviceProcAddr` 来查询Vulkan设备函数。
如果你选择使用`vkGetInstanceProcAddr`，它将在调用链上有附加的层，这将些许降低程序性能。
然而，返回的函数指针可以用在以后创建的任何设备上，只要它关联到同一个Vulkan实例上。
如果你使用 `vkGetDeviceProcAddr`，调用链将针对特定设备被优化，但是它将  **只** 能在查询到该函数的设备上使用。
 还有，不像 `vkGetInstanceProcAddr`，`vkGetDeviceProcAddr` 只能用于核心的Vulkan设备函数，或者设备拓展函数。

最佳解决方案是使用`vkGetInstanceProcAddr`来查询实例拓展函数，使用`vkGetDeviceProcAddr`来查询设备拓展函数。
参考[Best Application Performance Setup](#best-application-performance-setup) 以获取更多信息。


和Instance拓展一样，一个设备拓展是一系列的拓展了Vulkan语言的Vulkan设备函数。
你可以在本文档后面有更多的了解。


#### Dispatch Tables and Call Chains

Vulkan使用对象模型来控制特定动作/操作的生命周期。被操作的对象一般都作为Vulkan调用的第一个参数，而且是一个可分发对象（参看Vulkan规范 2.3节 对象模型）。在底层，可分发对象的handle是一个指向数据结构的指针，数据结构反过来包含了一个由加载器维护的分发表。这个转发表包含了能够获取到这个对象的Vulkan函数的指针。

加载器维护了两种类型的分发表：
 - Instance转发表
  - 在`vkCreateInstance`调用中加载器创建的
 - 设备转发表
  - 在`vkCreateDevice`调用中加载器创建的

在此时，应用程序或者系统可以指定可选的将被包括的layers。加载器将初始化指定的layers，来为每一个Vulkan函数创建一个调用链，转发表的每一条将指向该调用链的第一个元素。故，加载器为每一个被创建的 `VkInstance` 建立了instance调用链，为每一个被创建的`VkDevice` 建立了设备调用链。

当应用程序调用一个Vulkan函数，通常这将先访问加载器的*trampoline* 函数。这些*trampoline* 函数很小、简短，能跳转到给定对象的分发表的某一条。另外，对于在instance调用链的函数，加载器有额外的函数，称为 *terminator*，它在所有即将启用的layers把合适的信息存放到可选的ICDs之后被调用。


##### Instance Call Chain Example

例如，如下的图展示了`vkCreateInstance`的调用链发生了什么。在初始化链之后，加载器将调用第一层的`vkCreateInstance`，它将再次调用加载器，这个函数将调用每一个ICD的`vkCreateInstance` 并保存运行结果。这允许调用链中每一个被启用的层都能基于`VkInstanceCreateInfo`数据结构建立自己所需的信息。

![Instance Call Chain](./images/loader_instance_chain.png)

这也强调出来加载器在使用instance调用链时必须管理好各种复杂性。如下所示，加载器的 *terminator* 必须在可见的多个ICD中收集或发送分类信息。这表示加载器必须获知实例层的各个拓展，以便更好的分类。


##### Device Call Chain Example

设备调用链是在`vkCreateDevice` 调用中创建的，通常较为简单，因为他们只和一个物理设备打交道，ICD也总是调用链的 *terminator*。

![Loader Device Call Chain](./images/loader_device_chain_loader.png)


<br/>
<br/>

## Application Interface to the Loader

在本节，我们将讨论应用程序如何和加载器交互协作，包括：
  * [Interfacing with Vulkan Functions](#interfacing-with-vulkan-functions)
    * [Vulkan Direct Exports](#vulkan-direct-exports)
    * [Directly Linking to the Loader](#directly-linking-to-the-loader)
      * [Dynamic Linking](#dynamic-linking)
      * [Static Linking](#static-linking)
    * [Indirectly Linking to the Loader](#indirectly-linking-to-the-loader)
    * [Best Application Performance Setup](#best-application-performance-setup)
    * [ABI Versioning](#abi-versioning)
  * [Application Layer Usage](#application-layer-usage)
    * [Implicit vs Explicit Layers](#implicit-vs-explicit-layers)
    * [Forcing Layer Source Folders](#forcing-layer-source-folders)
    * [Forcing Layers to be Enabled](#forcing-layers-to-be-enabled)
    * [Overall Layer Ordering](#overall-layer-ordering)
  * [Application Usage of Extensions](#application-usage-of-extensions)
    * [Instance and Device Extensions](#instance-and-device-extensions)
    * [WSI Extensions](#wsi-extensions)
    * [Unknown Extensions](#unknown-extensions)

  
#### Interfacing with Vulkan Functions
你可以通过加载器用多种方式和Vulkan函数交互。


##### Vulkan Direct Exports
在Windows，Linux和Android上的加载器将导出所有的核心Vulkan和所有合适的 Window System Interface (WSI) 拓展。完成了这步会让Vulkan开发变得简单。当应用程序通过这种方式直接链接到加载器library，Vulkan调用只是简单的跳转函数，
跳转到它们所在的对象在转发表中合适的入口。


##### Directly Linking to the Loader

###### Dynamic Linking
加载器通常是以动态库的方式发布的（在Windows上是.dll，在Linux上是.so），安装在系统的动态库路径中。
链接到加载器通常推荐的方式是链接到动态库，这样做允许加载器可以更新版本来修复bug或提升性能。
还有，动态库通常被安装在Windows 系统中，作为驱动安装的一部分，在Linux系统上通常通过包管理系统提供。
这意味着应用程序一般可以获取到当前系统中加载器的一份copy。如果应用程序想要百分百保证加载器是存在的，程序可以自带一份加载器安装包。

###### Static Linking
加载器也可以被静态链接（这通常随着Windows SDK一起发布，名为`VKstatic.1.lib`）。链接静态加载器意味着用户不需要系统已经安装过Vulkan运行时环境，这也保证了你的应用程序将使用一个特定版本的加载器。
然而，这种方式有几个缺陷：

  - 若不重新连接程序则无法更新静态库
  - 这可能导致包含的两个库可能包含不同版本的加载器
    - 这可能导致不同版本加载器潜在的的冲突

因此，推荐用户使用链接到.dll或.so 的方式。


##### Indirectly Linking to the Loader
应用程序并不需要直接链接到加载器库，它们可以对加载器使用合适的平台特定的动态符号查找方式来初始化应用程序自己的转发表。
这允许应用程序在没有找到加载器时可以从容的退出。这也给应用程序提供了调用Vulkan函数最快的机制。
应用程序只要向加载器库查询 `vkGetInstanceProcAddr` 的地址（通过dlsym()这个系统调用）。应用程序可以使用 `vkGetInstanceProcAddr` 来找到所有函数的地址和可用的拓展的地址，
如 `vkCreateInstance`,`vkEnumerateInstanceExtensionProperties` 和`vkEnumerateInstanceLayerProperties`。


##### Best Application Performance Setup

如果你希望得到最佳性能，你应该建立起自己的转发表，这样你所有的实例函数都可以通过 `vkGetInstanceProcAddr` 获取到，所有的设备函数可以使用`vkGetDeviceProcAddr` 获取到。

*Why should you do this?*

这个答案取决于实例函数调用链和设备函数调用链是如何被实现的。记住，一个[Vulkan 实例是一个高级的数据结构，用来提供Vulkan系统级别的信息](#instance-related-objects)。
因此，实例函数需要传播到系统上所有可用的ICD。下图展示了启用三个layers是实例调用链的视图：

![Instance Call Chain](./images/loader_instance_chain.png)

若你使用`vkGetInstanceProcAddr`查询它的话，这也是Vulkan设备函数调用链看起来的样子。
在另外一方面，一个设备函数并不需要担心传播，因为它已经知道自己关联的ICD和和函数所应该最后被调用的物理设备了。
由此，加载器不需要和任何启用的layers与ICD相干。故，如果你使用一个加载器暴露出来的设备函数，如上描述情形的调用链将如下所示：

![Loader Device Call Chain](./images/loader_device_chain_loader.png)

一个更好的解决方案是在应用程序中使用`vkGetDeviceProcAddr` 来调用所有的设备函数。这将通过移出大多数情形下加载器的中无关紧要的部分来进行更多优化。

![Application Device Call Chain](./images/loader_device_chain_app.png)

还有，注意如果没有启用任何layer，你的应用程序函数指针将直接指向ICD。如果调用次数足够多，将带来一些性能提升。


**注意:** 有一些设备函数要求加载器用一个 *trampoline* and *terminator*来拦截它们。虽然个数比较少，但是，它们是典型的需要加载器包装他们数据的函数。
在这些情形下，即使是设备函数调用链看起来也像是实例调用链。一个例子是`vkCreateSwapchainKHR`要求一个 *terminator* 。对于这个函数，在把函数剩下的信息传递给ICD之前，
加载器需要把 KHR_surface 对象转换为一个ICD特定的 KHR_surface。

记住:
 * `vkGetInstanceProcAddr` 可以用来查询设备或者实例入口函数（包括所有核心的入口函数）。
 * `vkGetDeviceProcAddr` 只可以用来查询设备拓展或者核心的设备入口函数。


##### ABI Versioning

Vulkan加载器库有各种分发方式，包括Vulkan SDK、操作系统包管理分发和Independent Hardware Vendor (IHV) 驱动安装包。
这些东西不在本文的关注点内。然而，Vulkan加载器库的名字和版本需要被明确下来，应用程序才能链接到正确版本的Vulkan ABI 库。
Vulkan 版本机制保证ABI 在同一个主版本号内向后兼容。（比如1.0 和1.1）。在Windows上，加载器库文件名字包含了ABI版本，所以会多个版本的Vulkan加载器库会同时存在。
Vulkan加载器库文件名形如 `vulkan-<ABI version>.dll`。例如，对于Windows上Vulkan 1.X版本，库文件名为 vulkan-1.dll。
这个文件通常可以在 windows/system32 目录找到（在64位系统上，32位版本的加载器以同样的命在可以在windows/sysWOW64 目录找到）。

对于Linux，共享库的版本号以后缀决定。故，ABI号并不如Windows上那样编码。
在Linux上应用程序如果想要连接最新版本的Vulkan ABI版本，只需要链接libvulkan.so 即可。也可以由应用程序指定链接的Vulkan ABI版本（如libvulkan.so.1）。


#### Application Layer Usage

应用程序可以使用各种layers或者拓展来获取在核心API 提供的基础上更多的能力。
一个layer并不能引入在Vulkan.h中暴露出来的新的Vulkan核心API 函数。然而，layers可以提供拓展，来引入新的Vulkan命令，可以通过查询拓展接口来获取到。

layers常见的一种使用方式是API验证，它可以在应用程序开发过程中载入layer时被启动，但是在应用程序发布时被关闭。
这减少了验证应用程序使用API的性能损失，在一些其他的图形API中并没有这个特性。

可以使用`vkEnumerateInstanceLayerProperties`来获知应用程序可以用哪些layers。
它会报告加载器发现的所有layers。加载器在系统的各个路径搜索layers。关于此点，请参考下节的[Layer discovery](#layer-discovery) 。

要想启用一个layer，仅需要在调用`vkCreateInstance`时把你想要启用的layer的名字传递到 `VkInstanceCreateInfo`的 `ppEnabledLayerNames`域。
一旦完成了，你想要启用的layers对于所有的使用已创建的`VkInstance`对象及其子对象的Vulkan函数都已经启用了。

**NOTE:** Layer 顺序在几个场合下非常重要，因为一些layers会相互之间操作。当此情形时需要格外小心。参考[Overall Layer Ordering](#overall-layer-ordering) 一节获取更多信息。

如下代码展示了如何启用VK_LAYER_LUNARG_standard_validation layer。

```
   char *instance_validation_layers[] = {
        "VK_LAYER_LUNARG_standard_validation"
    };
    const VkApplicationInfo app = {
        .sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
        .pNext = NULL,
        .pApplicationName = "TEST_APP",
        .applicationVersion = 0,
        .pEngineName = "TEST_ENGINE",
        .engineVersion = 0,
        .apiVersion = VK_API_VERSION_1_0,
    };
    VkInstanceCreateInfo inst_info = {
        .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
        .pNext = NULL,
        .pApplicationInfo = &app,
        .enabledLayerCount = 1,
        .ppEnabledLayerNames = (const char *const *)instance_validation_layers,
        .enabledExtensionCount = 0,
        .ppEnabledExtensionNames = NULL,
    };
    err = vkCreateInstance(&inst_info, NULL, &demo->inst);
```

在 `vkCreateInstance` 和 `vkCreateDevice`时，加载器构造了调用链，包含应用程序指定启用的layers。  `ppEnabledLayerNames` 数组中元素的顺序非常重要。
0号元素是调用链最顶层的layer，最后一个元素距离驱动最近。参看  [Overall Layer Ordering](#overall-layer-ordering) 一节来获取关于layer排序的信息。

**NOTE:** *Device Layers 已经被废弃*
> `vkCreateDevice` 最初可以像`vkCreateInstance`那样选择layers。这导致 "instance
> layers" 概念 和 "device layers"。Khronos决定废弃"device layer" 功能，只考虑"instance layers"。
> 因此， `vkCreateDevice` 将会使用`vkCreateInstance`指定的layers。
> 因此，如下条目也被废弃了：
> * `VkDeviceCreateInfo` fields:
>  * `ppEnabledLayerNames`
>  * `enabledLayerCount`
> * The `vkEnumerateDeviceLayerProperties` function


##### Implicit vs Explicit Layers

显式layers是被应用程序启用的layers（例如，通过vkCreateInstance 函数启用），或者通过环境变量（如前面提到的）。

隐式layers是那些默认存在的layers。例如，特定的应用程序环境（如Steam或者娱乐系统）可能有对于所有应用程序都启用的layers。其他的隐式layers可能有系统给所有的程序都自动启用了，
相比之下显式layers需要显式地启用。

隐式层相比于显式层有一些附加的要求，它们需要受环境变量控制来启用、关闭。这是因为它们不受应用程序影响，也不会造成任何问题。
一个好的准则就是记住要定义启动和关闭layers的环境变量，这样用户就可以决定启用哪些功能了。
在桌面平台（Windows和Linux），这些启用/关闭设定都在层的JSON文件中定义了。

关于系统安装的显式和隐式layers在稍后的  [Layer Discovery Section](#layer-discovery) 讲解。
我们暂时只需要知道显式、隐式的区别取决于操作系统即可，如下表所示：


| Operating System | Implicit Layer Identification |
|----------------|--------------------|
| Windows  | 隐式层和显式层在不同的Windows注册表位置 |
| Linux | 隐式层和显式层在不同的目录中|
| Android | Android  ** 不支持显式层 **  |


##### Forcing Layer Source Folders

开发者也许需要使用特殊的，预先产生的layers，而无需修改系统安装的layers。
你可以通过定义"VK\_LAYER\_PATH"  环境变量来引导加载器到一个特定的文件夹搜寻layers。
这将覆盖查找系统安装的layers的机制。因为重点的layers可能在系统的不同文件夹中，这个环境变量可以包含几个不同的路径，以系统特定路径分隔符来分割。
在Windows上，每一个文件夹路径在列表中都应该用一个分号来分割。在Linux上每个文件夹路径都应该用一个冒号来分割。

如果 "VK\_LAYER\_PATH" 存在， **只有** 这个文件夹会被扫描。列表中每一个目录都应该是包含layer明细文件的全路径。


##### Forcing Layers to be Enabled on Windows and Linux

开发者可能想要那些在使用的给定应用程序并没有被启用的层开始启用。
在Linux和Windows上，环境变量"VK\_INSTANCE\_LAYERS" 可以用来启用那些并没有被应用程序的vkCreateInstance`指定的层。
"VK\_INSTANCE\_LAYERS" 是一个以冒号（Linux）、分号（Windows）间隔的层名字的列表。
前后顺序的关系是：在列表中第一个是最顶层的层（靠近应用程序），列表最后一个是最底层的layer（靠近驱动）。
参考 [Overall Layer Ordering](#overall-layer-ordering) 一节以获取更多细节。

应用程序指定的layers和用户指定的layers（通过环境变量）会被综合，重复的会被加载器删除。通过环境变量指定的layers是最顶层（靠近应用程序），而应用程序指定的layers是最底层的。

使用用环境变量的一个例子是在Windows或Linux上启用的验证layer `VK_LAYER_LUNARG_parameter_validation` ：

```
> $ export VK_INSTANCE_LAYERS=VK_LAYER_LUNARG_parameter_validation
```


##### Overall Layer Ordering

The overall ordering of all layers by the loader based on the above looks
as follows:

![Loader Layer Ordering](./images/loader_layer_order.png)

排序对于多个显式layer也很重要。一些layers可能依赖于加载器在加载之前或者之后的其他的行为。
例如， VK_LAYER_LUNARG_core_validation 要求VK_LAYER_LUNARG_parameter_validation 先被调用。
这是因为VK_LAYER_LUNARG_parameter_validation 将会在VK_LAYER_LUNARG_core_validation进行检查之前
过滤任何无效的 `NULL`  指针。如果没有这么干，你可能会看到 VK_LAYER_LUNARG_core_validation层崩溃，这本是可以避免的。

#### Application Usage of Extensions

拓展是由layer，加载器或者ICD提供的可选的功能。拓展可以修改Vulkan API的行为，需要在Khronos进行指定和注册。
这些拓展可以由Independent Hardware Vendor (IHV) 创建，来暴露出硬件的功能，或者由layer程序制造者来暴露内部特性，或者由加载器来提升性能。
提取信息用途的拓展可以在Vulkan Spec，vulkah.h头文件中找到。

##### Instance and Device Extensions

如在  [Instance Versus Device](#instance-versus-device) 一节中提到的，有两种类型的拓展：
 * Instance Extensions
 * Device Extensions

实例拓展是一种可修改实例级别对象（如`VkInstance`、`VkPhysicalDevice`）上已有行为或者实现新的行为的拓展。
一个设备拓展也是相似的定义，只是针对于任何 `VkDevice` 对象，或者以`VkDevice`(`VkQueue` 和 `VkCommandBuffer` 等)为父对象的可转发对象。

当你使用`vkCreateInstance`启用实例拓展，用`vkCreateDevice`启用设备拓展时，知道我们要启用那种类别的拓展  **非常** 重要。

加载器从layers（显式的和隐式的）和ICD中收集并综合所有的拓展，发生在加载器在把这些信息通过`vkEnumerateXXXExtensionProperties` XXX 是 "Instance" 或 "Device"）
返回给应用程序之前。
 - Instance extensions are discovered via
`vkEnumerateInstanceExtensionProperties`.
 - Device extensions are be discovered via
`vkEnumerateDeviceExtensionProperties`.

查看`vulkan.h`, 你将发现它们很像。比如，`vkEnumerateInstanceExtensionProperties` 原型如下：

```
   VkResult
   vkEnumerateInstanceExtensionProperties(const char *pLayerName,
                                          uint32_t *pPropertyCount,
                                          VkExtensionProperties *pProperties);
```

The "pLayerName" parameter in these functions is used to select either a single
layer or the Vulkan platform implementation. If "pLayerName" is NULL, extensions
from Vulkan implementation components (including loader, implicit layers, and
ICDs) are enumerated. If "pLayerName" is equal to a discovered layer module name
then only extensions from that layer (which may be implicit or explicit) are
enumerated. Duplicate extensions (e.g. an implicit layer and ICD might report
support for the same extension) are eliminated by the loader. For duplicates,
the ICD version is reported and the layer version is culled.

Also, Extensions *must be enabled* (in `vkCreateInstance` or `vkCreateDevice`)
before the functions associated with the extensions can be used.  If you get an
Extension function using either `vkGetInstanceProcAddr` or
`vkGetDeviceProcAddr`, but fail to enable it, you could experience undefined
behavior.  This should actually be flagged if you run with Validation layers
enabled.


##### WSI Extensions

Khronos已经审批的WSI拓展已经可用了，并提供了多种执行环境的Windows System集成。
我们必须要明白一些WSI拓展对所有平台都有效，但有一些值针对某些执行环境或者加载器才有效。桌面端加载器（目前只有Windows和Linux支持）只
启用并直接对当前环境暴露出了这些WSI拓展。WSI拓展的选择，在加载器编译期间就通过标志位指定了。所有版本的桌面端加载器当前都至少保留了如下的WSI拓展：
- VK_KHR_surface
- VK_KHR_swapchain
- VK_KHR_display

此外，如下的目标操作系统的加载器都支持其特定的拓展：

| Windowing System | Extensions available |
|----------------|--------------------|
| Windows  | VK_KHR_win32_surface |
| Linux (Default) |  VK_KHR_xcb_surface and VK_KHR_xlib_surface |
| Linux (Wayland) | VK_KHR_wayland_surface |
| Linux (Mir)  | VK_KHR_mir_surface |

**NOTE:** Wayland 和 Mir 平台在当前并没有完全被支持。  Wayland支持只是出现了，但应当被视为beta版本。Mir支持则完全没有实现。

要明确虽然加载支持拓展的多重入口，但是需要认证才能使用它们：
* 至少有一个物理设备必须支持这些拓展
* 应用程序必须选择这样的一个物理设备
* 应用程序必须要求在床将实例或者逻辑设备时，这些拓展被启用了（这依赖于给定的拓展和实例或设备之间是否共同工作）。
* 实例或者逻辑设备的创建必须要成功

这些条件都符合之后你才能在Vulkan程序中使用WSI拓展。


##### Unknown Extensions

拓展Vulkan的能力是如此的简单，所以创建拓展时加载器也无需知道拓展相关的信息。如果是设备拓展，加载器将把未知的入口函数传递设备调用链，以ICD入口点函数为作为结束。
对于实例拓展而言，也是同样的过程，拓展接受一个物理设备参数为第一个component。然而，对于所有其他的实例拓展，加载器将加载失败。

*But why doesn't the loader support unknown instance extensions?*
<br/>
Let's look again at the Instance call chain:

![Instance call chain](./images/loader_instance_chain.png)

注意，对于一个普通实例函数调用，加载器必须处理好把函数调用传递给可用的ICD。如果加载器不知道实例调用的参数或者返回值，就不可能正确的把信息传递到ICD。
有很多办法可以做到这样，将在后面提到。然而，目前，加载器不支持把物理设备作为第一个参数的实例拓展。

Because the device call-chain does not normally pass through the loader
*terminator*, this is not a problem for device extensions.  Additionally,
since a physical device is associated with one ICD, we can use a generic
*terminator* pointing to one ICD.  This is because both of these extensions
terminate directly in the ICD they are associated with.

*Is this a big problem?*
<br/>
No!  绝大多数拓展功能只影响一个物理设备或者逻辑设备，并不影响实例。故，大多数拓展都应该被加载器所直接支持。

##### Filtering Out Unknown Instance Extension Names
在一些情形下，一个ICD可能支持实例拓展，但加载器并不支持。
对于以上原因，当应用程序调用`vkEnumerateInstanceExtensionProperties`时，加载器将过滤掉这些未知的实例拓展的名字。
另外，如果你继续使用这些拓展，此行为将导致加载器在运行`vkCreateInstance`时抛出一个错误。
这将保护应用程序，避免其使用能致其崩溃的功能。

另外一方面，如果你能够安全的使用拓展，你可以定义环境变量 `VK_LOADER_DISABLE_INST_EXT_FILTER` 来关闭过滤，并设置这个值为一个非0值。
这将高效禁用加载器的过滤实例拓展的名字。

<br/>
<br/>

## Loader and Layer Interface

In this section we'll discuss how the loader interacts with layers, including:
  * [Layer Discovery](#layer-discovery)
    * [Layer Manifest File Usage](#layer-manifest-file-usage)
    * [Android Layer Discovery](#android-layer-discovery)
    * [Windows Layer Discovery](#windows-layer-discovery)
    * [Linux Layer Discovery](#linux-layer-discovery)
  * [Layer Version Negotiation](#layer-version-negotiation)
  * [Layer Call Chains and Distributed Dispatch](#layer-call-chains-and-distributed-dispatch)
  * [Layer Unknown Physical Device Extensions](#layer-unknown-physical-device-extensions)
  * [Layer Intercept Requirements](#layer-intercept-requirements)
  * [Distributed Dispatching Requirements](#distributed-dispatching-requirements)
  * [Layer Conventions and Rules](#layer-conventions-and-rules)
  * [Layer Dispatch Initialization](#layer-dispatch-initialization)
  * [Example Code for CreateInstance](#example-code-for-createinstance)
  * [Example Code for CreateDevice](#example-code-for-createdevice)
  * [Meta-layers](#meta-layers)
  * [Special Considerations](#special-considerations)
    * [Associating Private Data with Vulkan Objects Within a Layer](#associating-private-data-with-vulkan-objects-within-a-layer)
      * [Wrapping](#wrapping)
      * [Hash Maps](#hash-maps)
    * [Creating New Dispatchable Objects](#creating-new-dispatchable-objects)
  * [Layer Manifest File Format](#layer-manifest-file-format)
    * [Layer Manifest File Version History](#layer-manifest-file-version-history)
  * [Layer Library Versions](#layer-library-versions)
    * [Layer Library API Version 2](#layer-library-api-version-2)
    * [Layer Library API Version 1](#layer-library-api-version-1)
    * [Layer Library API Version 0](#layer-library-api-version-0)
  

 
#### Layer Discovery

如在[Application Interface section](#implicit-vs-explicit-layers) 一节中提到的，layers可以分为两类：
 * Implicit Layers
 * Explicit Layers

主要的区别为隐式层是自动开启的（除非被overriden了），显式层必须被开启。记住，隐式层并不是在所有的操作系统中都存在（如Android）。


在任何系统上，加载器按照用户的要求在特定位置查找可用的layers。在系统中寻找可用的layers的过程也被称为“Layer Discovery”。在寻找过程中，
加载器决定采用哪些layers，layers的名字，layers的版本，和layer支持的拓展。这些信息由应用程序通过`vkEnumerateInstanceLayerProperties`可获取到。

对加载器可用的layers群组也被称为layer库。本节定义了发现layer库包含哪些layer的拓展接口。

本节也指定了最小惯例和layer必须遵守的规则，特别是涉及到与加载器和其他layer有交互的layer。

##### Layer Manifest File Usage

在Windows和Linux系统上，JSON格式的配置文件被用来存储layer信息。为了找到系统安装的layers，Vulkan加载器将会读取这些JSON文件来知晓layer与拓展的名字和属性。
配置文件的使用允许加载器避免加载了任何应用程序用不到的layer或拓展。[Layer Manifest File](#layer-manifest-file-format) 的格式在下面会详细讲述。

Android加载器不使用配置文件。相反，加载器使用特殊的函数--自省函数--来查询layer属性。
使用这些函数的目标是为了获取到与读取配置文件相同的信息。这些自省函数不会被桌面端加载器使用，但应该在layer中存在并保持一致性。特定的自省函数通过
 [Layer Manifest File Format](#layer-manifest-file-format) 表被调用。


##### Android Layer Discovery

在Android上，加载器遍历/data/local/debug/vulkan文件夹来搜寻layers。应用程序debug版本可以遍历并启用该位置的layers。


##### Windows Layer Discovery

In order to find system-installed layers, the Vulkan loader will scan the
values in the following Windows registry keys:

```
   HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ExplicitLayers
   HKEY_CURRENT_USER\SOFTWARE\Khronos\Vulkan\ExplicitLayers
   HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers
   HKEY_CURRENT_USER\SOFTWARE\Khronos\Vulkan\ImplicitLayers
```

For each value in these keys which has DWORD data set to 0, the loader opens
the JSON manifest file specified by the name of the value. Each name must be a
full pathname to the manifest file.

Additionally, the loader will scan through registry keys specific to Display
Adapters and all Software Components associated with these adapters for the
locations of JSON manifest files. These keys are located in device keys
created during driver installation and contain configuration information
for base settings, including Vulkan, OpenGL, and Direct3D ICD location.

The Device Adapter and Software Component key paths should be obtained through the PnP
Configuration Manager API. The `000X` key will be a numbered key, where each
device is assigned a different number.

```
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanExplicitLayers
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanImplicitLayers
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanExplicitLayers
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanImplicitLayers
```

In addition, on 64-bit systems there may be another set of registry values, listed
below. These values record the locations of 32-bit layers on 64-bit operating systems,
in the same way as the Windows-on-Windows functionality.

```
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanExplicitLayersWow
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanImplicitLayersWow
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanExplicitLayersWow
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanImplicitLayersWow
```

If any of the above values exist and is of type `REG_SZ`, the loader will open the JSON
manifest file specified by the key value. Each value must be a full absolute
path to a JSON manifest file. A key value may also be of type `REG_MULTI_SZ`, in
which case the value will be interpreted as a list of paths to JSON manifest files.

In general, applications should install layers into the `SOFTWARE\Khrosos\Vulkan`
paths. The PnP registry locations are intended specifically for layers that are
distrubuted as part of a driver installation. An application installer should not
modify the device-specific registries, while a device driver should not modify
the system wide registries.

The Vulkan loader will open each manifest file that is given
to obtain information about the layer, including the name or pathname of a
shared library (".dll") file.  However, if VK\_LAYER\_PATH is defined, then the
loader will instead look at the paths defined by that variable instead of using
the information provided by these registry keys.  See
[Forcing Layer Source Folders](#forcing-layer-source-folders) for more
information on this.


##### Linux Layer Discovery

On Linux, the Vulkan loader will scan the files in the following Linux
directories:

    /usr/local/etc/vulkan/explicit_layer.d
    /usr/local/etc/vulkan/implicit_layer.d
    /usr/local/share/vulkan/explicit_layer.d
    /usr/local/share/vulkan/implicit_layer.d
    /etc/vulkan/explicit_layer.d
    /etc/vulkan/implicit_layer.d
    /usr/share/vulkan/explicit_layer.d
    /usr/share/vulkan/implicit_layer.d
    $HOME/.local/share/vulkan/explicit_layer.d
    $HOME/.local/share/vulkan/implicit_layer.d

Of course, ther are some things you have to know about the above folders:
 1. The "/usr/local/*" directories can be configured to be other directories at
build time.
 2. $HOME is the current home directory of the application's user id; this path
will be ignored for suid programs.
 3. The "/usr/local/etc/vulkan/\*\_layer.d" and
"/usr/local/share/vulkan/\*\_layer.d" directories are for layers that are
installed from locally-built sources.
 4. The "/usr/share/vulkan/\*\_layer.d" directories are for layers that are
installed from Linux-distribution-provided packages.

As on Windows, if VK\_LAYER\_PATH is defined, then the
loader will instead look at the paths defined by that variable instead of using
the information provided by these default paths.  However, these
environment variables are only used for non-suid programs.  See
[Forcing Layer Source Folders](#forcing-layer-source-folders) for more
information on this.


#### Layer Version Negotiation

Now that a layer has been discovered, an application can choose to load it (or
it is loaded by default if it is an Implicit layer).  When the loader attempts
to load the layer, the first thing it does is attempt to negotiate the version
of the loader to layer interface.  In order to negotiate the loader/layer
interface version, the layer must implement the
`vkNegotiateLoaderLayerInterfaceVersion` function.  The following information is
provided for this interface in include/vulkan/vk_layer.h:

```cpp
  typedef enum VkNegotiateLayerStructType {
      LAYER_NEGOTIATE_INTERFACE_STRUCT = 1,
  } VkNegotiateLayerStructType;

  typedef struct VkNegotiateLayerInterface {
      VkNegotiateLayerStructType sType;
      void *pNext;
      uint32_t loaderLayerInterfaceVersion;
      PFN_vkGetInstanceProcAddr pfnGetInstanceProcAddr;
      PFN_vkGetDeviceProcAddr pfnGetDeviceProcAddr;
      PFN_GetPhysicalDeviceProcAddr pfnGetPhysicalDeviceProcAddr;
  } VkNegotiateLayerInterface;

  VkResult vkNegotiateLoaderLayerInterfaceVersion(
                   VkNegotiateLayerInterface *pVersionStruct);
```

You'll notice the `VkNegotiateLayerInterface` structure is similar to other
Vulkan structures.  The "sType" field, in this case takes a new enum defined
just for internal loader/layer interfacing use.  The valid values for "sType"
could grow in the future, but right only havs the one value
"LAYER_NEGOTIATE_INTERFACE_STRUCT".

This function (`vkNegotiateLoaderLayerInterfaceVersion`) should be exported by
the layer so that using "GetProcAddress" on Windows or "dlsym" on Linux, should
return a valid function pointer to it.  Once the loader has grabbed a valid
address to the layers function, the loader will create a variable of type
`VkNegotiateLayerInterface` and initialize it in the following ways:
 1. Set the structure "sType" to "LAYER_NEGOTIATE_INTERFACE_STRUCT"
 2. Set pNext to NULL.
     - This is for future growth
 3. Set "loaderLayerInterfaceVersion" to the current version the loader desires
to set the interface to.
      - The minimum value sent by the loader will be 2 since it is the first
version supporting this function.

The loader will then individually call each layer’s
`vkNegotiateLoaderLayerInterfaceVersion` function with the filled out
“VkNegotiateLayerInterface”. The layer will either accept the loader's version
set in "loaderLayerInterfaceVersion", or modify it to the closest value version
of the interface that the layer can support.  The value should not be higher
than the version requested by the loader.  If the layer can't support at a
minimum the version requested, then the layer should return an error like
"VK_ERROR_INITIALIZATION_FAILED".  If a layer can support some version, then
the layer should do the following:
 1. Adjust the version to the layer's desired version.
 2. The layer should fill in the function pointer values to its internal
functions:
    - "pfnGetInstanceProcAddr" should be set to the layer’s internal
`GetInstanceProcAddr` function.
    - "pfnGetDeviceProcAddr" should be set to the layer’s internal
`GetDeviceProcAddr` function.
    - "pfnGetPhysicalDeviceProcAddr" should be set to the layer’s internal
`GetPhysicalDeviceProcAddr` function.
      - If the layer supports no physical device extensions, it may set the
value to NULL.
      - More on this function later
 3. The layer should return "VK_SUCCESS"

This function **SHOULD NOT CALL DOWN** the layer chain to the next layer.
The loader will work with each layer individually.

If the layer supports the new interface and reports version 2 or greater, then
the loader will use the “fpGetInstanceProcAddr” and “fpGetDeviceProcAddr”
functions from the “VkNegotiateLayerInterface” structure.  Prior to these
changes, the loader would query each of those functions using "GetProcAddress"
on Windows or "dlsym" on Linux.


#### Layer Call Chains and Distributed Dispatch

There are two key architectural features that drive the loader to layer library
interface:
 1. Separate and distinct instance and device call chains
 2. Distributed dispatch.

You can read an overview of dispatch tables and call chains above in the
[Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains) section.

What's important to note here is that a layer can intercept Vulkan
instance functions, device functions or both. For a layer to intercept instance
functions, it must participate in the instance call chain.  For a layer to
intercept device functions, it must participate in the device call chain.

Remember, a layer does not need to intercept all instance or device functions,
instead, it can choose to intercept only a subset of those functions.

Normally, when a layer intercepts a given Vulkan function, it will call down the
instance or device call chain as needed. The loader and all layer libraries that
participate in a call chain cooperate to ensure the correct sequencing of calls
from one entity to the next. This group effort for call chain sequencing is
hereinafter referred to as **distributed dispatch**.

In distributed dispatch each layer is responsible for properly calling the next
entity in the call chain. This means that a dispatch mechanism is required for
all Vulkan functions that a layer intercepts. If a Vulkan function is not
intercepted by a layer, or if a layer chooses to terminate the function by not
calling down the chain, then no dispatch is needed for that particular function.

For example, if the enabled layers intercepted only certain instance functions,
the call chain would look as follows:
![Instance Function Chain](./images/function_instance_chain.png)

Likewise, if the enabled layers intercepted only a few of the device functions,
the call chain could look this way:
![Device Function Chain](./images/function_device_chain.png)

The loader is responsible for dispatching all core and instance extension Vulkan
functions to the first entity in the call chain.


#### Layer Unknown Physical Device Extensions

Originally, if the loader was called with `vkGetInstanceProcAddr`, it would
result in the following behavior:
 1. The loader would check if core function:
    - If it was, it would return the function pointer
 2. The loader would check if known extension function:
    - If it was, it would return the function pointer
 3. If the loader knew nothing about it, it would call down using
`GetInstanceProcAddr`
    - If it returned non-NULL, treat it as an unknown logical device command.
    - This meant setting up a generic trampoline function that takes in a
VkDevice as the first parameter and adjusting the dispatch table to call the
ICD/Layers function after getting the dispatch table from the VkDevice.
 4. If all the above failed, the loader would return NULL to the application.

This caused problems when a layer attempted to expose new physical device
extensions the loader knew nothing about, but an application did.  Because the
loader knew nothing about it, the loader would get to step 3 in the above
process and would treat the function as an unknown logical device command.  The
problem is, this would create a generic VkDevice trampoline function which, on
the first call, would attempt to dereference the VkPhysicalDevice as a VkDevice.
This would lead to a crash or corruption.

In order to identify the extension entry-points specific to physical device
extensions, the following function can be added to a layer:

```cpp
PFN_vkVoidFunction vk_layerGetPhysicalDeviceProcAddr(VkInstance instance,
                                                     const char* pName);
```

This function behaves similar to `vkGetInstanceProcAddr` and
`vkGetDeviceProcAddr` except it should only return values for physical device
extension entry-points.  In this way, it compares "pName" to every physical
device function supported in the layer.

The following rules apply:
  * If it is the name of a physical device function supported by the layer, the
pointer to the layer's corresponding function should be returned.
  * If it is the name of a valid function which is **not** a physical device
function (i.e. an Instance, Device, or other function implemented by the layer),
then the value of NULL should be returned.
    * We don’t call down since we know the command is not a physical device
extension).
  * If the layer has no idea what this function is, it should call down the layer
chain to the next `vk_layerGetPhysicalDeviceProcAddr` call.
    * This can be retrieved in one of two ways:
      * During `vkCreateInstance`, it is passed to a layer in the
chain information passed to a layer in the `VkLayerInstanceCreateInfo`
structure.
        * Use `get_chain_info()` to get the pointer to the
`VkLayerInstanceCreateInfo` structure.  Let's call it chain_info.
        * The address is then under
chain_info->u.pLayerInfo->pfnNextGetPhysicalDeviceProcAddr
        * See
[Example Code for CreateInstance](#example-code-for-createinstance)
      * Using the next layer’s `GetInstanceProcAddr` function to query for
`vk_layerGetPhysicalDeviceProcAddr`.

This support is optional and should not be considered a requirement.  This is
only required if a layer intends to support some functionality not directly
supported by loaders released in the public.  If a layer does implement this
support, it should return the address of its `vk_layerGetPhysicalDeviceProcAddr`
function in the "pfnGetPhysicalDeviceProcAddr" member of the
`VkNegotiateLayerInterface` structure during
[Layer Version Negotiation](#layer-version-negotiation).  Additionally, the
layer should also make sure `vkGetInstanceProcAddr` returns a valid function
pointer to a query of `vk_layerGetPhysicalDeviceProcAddr`.

The new behavior of the loader's `vkGetInstanceProcAddr` with support for the
`vk_layerGetPhysicalDeviceProcAddr` function is as follows:
 1. Check if core function:
    - If it is, return the function pointer
 2. Check if known instance or device extension function:
    - If it is, return the function pointer
 3. Call the layer/ICD `GetPhysicalDeviceProcAddr`
    - If it returns non-NULL, return a trampoline to a generic physical device
function, and setup a generic terminator which will pass it to the proper ICD.
 4. Call down using `GetInstanceProcAddr`
    - If it returns non-NULL, treat it as an unknown logical device command.
This means setting up a generic trampoline function that takes in a VkDevice as
the first parameter and adjusting the dispatch table to call the ICD/Layers
function after getting the dispatch table from the VkDevice. Then, return the
pointer to corresponding trampoline function.
 5. Return NULL

You can see now, that, if the command gets promoted to core later, it will no
longer be setup using `vk_layerGetPhysicalDeviceProcAddr`.  Additionally, if the
loader adds direct support for the extension, it will no longer get to step 3,
because step 2 will return a valid function pointer.  However, the layer should
continue to support the command query via `vk_layerGetPhysicalDeviceProcAddr`,
until at least a Vulkan version bump, because an older loader may still be
attempting to use the commands.


#### Layer Intercept Requirements

  * Layers intercept a Vulkan function by defining a C/C++ function with
signature **identical** to the Vulkan API for that function.
  * A layer **must intercept at least** `vkGetInstanceProcAddr` and
`vkCreateInstance` to participate in the instance call chain.
  * A layer **may also intercept** `vkGetDeviceProcAddr` and `vkCreateDevice`
to participate in the device call chain.
  * For any Vulkan function a layer intercepts which has a non-void return value,
**an appropriate value must be returned** by the layer intercept function.
  * Most functions a layer intercepts **should call down the chain** to the
corresponding Vulkan function in the next entity.
    * The common behavior for a layer is to intercept a call, perform some
behavior, then pass it down to the next entity.
      * If you don't pass the information down, undefined behavior may occur.
      * This is because the function will not be received by layers further down
the chain, or any ICDs.
    * One function that **must never call down the chain** is:
      * `vkNegotiateLoaderLayerInterfaceVersion`
    * Three common functions that **may not call down the chain** are:
      * `vkGetInstanceProcAddr`
      * `vkGetDeviceProcAddr`
      * `vk_layerGetPhysicalDeviceProcAddr`
      * These functions only call down the chain for Vulkan functions that they
do not intercept.
  * Layer intercept functions **may insert extra calls** to Vulkan functions in
addition to the intercept.
    * For example, a layer intercepting `vkQueueSubmit` may want to add a call to
`vkQueueWaitIdle` after calling down the chain for `vkQueueSubmit`.
    * This would result in two calls down the chain: First a call down the
`vkQueueSubmit` chain, followed by a call down the `vkQueueWaitIdle` chain.
    * Any additional calls inserted by a layer must be on the same chain
      * If the function is a device function, only other device functions should
be added.
      * Likewise, if the function is an instance function, only other instance
functions should be added.


#### Distributed Dispatching Requirements

- For each entry-point a layer intercepts, it must keep track of the entry
point residing in the next entity in the chain it will call down into.
  * In other words, the layer must have a list of pointers to functions of the
appropriate type to call into the next entity.
  * This can be implemented in various ways but
for clarity, will be referred to as a dispatch table.
- A layer can use the `VkLayerDispatchTable` structure as a device dispatch
table (see include/vulkan/vk_layer.h).
- A layer can use the `VkLayerInstanceDispatchTable` structure as a instance
dispatch table (see include/vulkan/vk_layer.h).
- A Layer's `vkGetInstanceProcAddr` function uses the next entity's
`vkGetInstanceProcAddr` to call down the chain for unknown (i.e.
non-intercepted) functions.
- A Layer's `vkGetDeviceProcAddr` function uses the next entity's
`vkGetDeviceProcAddr` to call down the chain for unknown (i.e. non-intercepted)
functions.
- A Layer's `vk_layerGetPhysicalDeviceProcAddr` function uses the next entity's
`vk_layerGetPhysicalDeviceProcAddr` to call down the chain for unknown (i.e.
non-intercepted) functions.


#### Layer Conventions and Rules

A layer, when inserted into an otherwise compliant Vulkan implementation, must
still result in a compliant Vulkan implementation.  The intention is for layers
to have a well-defined baseline behavior.  Therefore, it must follow some
conventions and rules defined below.

A layer is always chained with other layers.  It must not make invalid calls
to, or rely on undefined behaviors of, its lower layers.  When it changes the
behavior of a function, it must make sure its upper layers do not make invalid
calls to or rely on undefined behaviors of its lower layers because of the
changed behavior.  For example, when a layer intercepts an object creation
function to wrap the objects created by its lower layers, it must make sure its
lower layers never see the wrapping objects, directly from itself or
indirectly from its upper layers.

When a layer requires host memory, it may ignore the provided allocators.  It
should use memory allocators if the layer is intended to run in a production
environment.  For example, this usually applies to implicit layers that are
always enabled.  That will allow applications to include the layer's memory
usage.

Additional rules include:
  - `vkEnumerateInstanceLayerProperties` **must** enumerate and **only**
enumerate the layer itself.
  - `vkEnumerateInstanceExtensionProperties` **must** handle the case where
`pLayerName` is itself.
    - It **must** return `VK_ERROR_LAYER_NOT_PRESENT` otherwise, including when
`pLayerName` is `NULL`.
  - `vkEnumerateDeviceLayerProperties` **is deprecated and may be omitted**.
    - Using this will result in undefined behavior.
  - `vkEnumerateDeviceExtensionProperties` **must** handle the case where
`pLayerName` is itself.
    - In other cases, it should normally chain to other layers.
  - `vkCreateInstance` **must not** generate an error for unrecognized layer
names and extension names.
    - It may assume the layer names and extension names have been validated.
  - `vkGetInstanceProcAddr` intercepts a Vulkan function by returning a local
entry-point
    - Otherwise it returns the value obtained by calling down the instance call
chain.
  - `vkGetDeviceProcAddr` intercepts a Vulkan function by returning a local
entry-point
    - Otherwise it returns the value obtained by calling down the device call
chain.
    - These additional functions must be intercepted if the layer implements
device-level call chaining:
      - `vkGetDeviceProcAddr`
      - `vkCreateDevice`(only required for any device-level chaining)
         - **NOTE:** older layer libraries may expect that `vkGetInstanceProcAddr`
ignore `instance` when `pName` is `vkCreateDevice`.
  - The specification **requires** `NULL` to be returned from
`vkGetInstanceProcAddr` and `vkGetDeviceProcAddr` for disabled functions.
    - A layer may return `NULL` itself or rely on the following layers to do so.


#### Layer Dispatch Initialization

- A layer initializes its instance dispatch table within its `vkCreateInstance`
function.
- A layer initializes its device dispatch table within its `vkCreateDevice`
function.
- The loader passes a linked list of initialization structures to layers via
the "pNext" field in the `VkInstanceCreateInfo` and `VkDeviceCreateInfo`
structures for `vkCreateInstance` and `VkCreateDevice` respectively.
- The head node in this linked list is of type `VkLayerInstanceCreateInfo` for
instance and VkLayerDeviceCreateInfo for device. See file
`include/vulkan/vk_layer.h` for details.
- A VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO is used by the loader for the
"sType" field in `VkLayerInstanceCreateInfo`.
- A VK_STRUCTURE_TYPE_LOADER_DEVICE_CREATE_INFO is used by the loader for the
"sType" field in `VkLayerDeviceCreateInfo`.
- The "function" field indicates how the union field "u" should be interpreted
within `VkLayer*CreateInfo`. The loader will set the "function" field to
VK_LAYER_LINK_INFO. This indicates "u" field should be `VkLayerInstanceLink` or
`VkLayerDeviceLink`.
- The `VkLayerInstanceLink` and `VkLayerDeviceLink` structures are the list
nodes.
- The `VkLayerInstanceLink` contains the next entity's `vkGetInstanceProcAddr`
used by a layer.
- The `VkLayerDeviceLink` contains the next entity's `vkGetInstanceProcAddr` and
`vkGetDeviceProcAddr` used by a layer.
- Given the above structures set up by the loader, layer must initialize their
dispatch table as follows:
  - Find the `VkLayerInstanceCreateInfo`/`VkLayerDeviceCreateInfo` structure in
the `VkInstanceCreateInfo`/`VkDeviceCreateInfo` structure.
  - Get the next entity's vkGet*ProcAddr from the "pLayerInfo" field.
  - For CreateInstance get the next entity's `vkCreateInstance` by calling the
"pfnNextGetInstanceProcAddr":
     pfnNextGetInstanceProcAddr(NULL, "vkCreateInstance").
  - For CreateDevice get the next entity's `vkCreateDevice` by calling the
"pfnNextGetInstanceProcAddr":
     pfnNextGetInstanceProcAddr(NULL, "vkCreateDevice").
  - Advanced the linked list to the next node: pLayerInfo = pLayerInfo->pNext.
  - Call down the chain either `vkCreateDevice` or `vkCreateInstance`
  - Initialize your layer dispatch table by calling the next entity's
Get*ProcAddr function once for each Vulkan function needed in your dispatch
table

#### Example Code for CreateInstance

```cpp
VkResult vkCreateInstance(
        const VkInstanceCreateInfo *pCreateInfo,
        const VkAllocationCallbacks *pAllocator,
        VkInstance *pInstance)
{
   VkLayerInstanceCreateInfo *chain_info =
        get_chain_info(pCreateInfo, VK_LAYER_LINK_INFO);

    assert(chain_info->u.pLayerInfo);
    PFN_vkGetInstanceProcAddr fpGetInstanceProcAddr =
        chain_info->u.pLayerInfo->pfnNextGetInstanceProcAddr;
    PFN_vkCreateInstance fpCreateInstance =
        (PFN_vkCreateInstance)fpGetInstanceProcAddr(NULL, "vkCreateInstance");
    if (fpCreateInstance == NULL) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    // Advance the link info for the next element of the chain
    chain_info->u.pLayerInfo = chain_info->u.pLayerInfo->pNext;

    // Continue call down the chain
    VkResult result = fpCreateInstance(pCreateInfo, pAllocator, pInstance);
    if (result != VK_SUCCESS)
        return result;

    // Init layer's dispatch table using GetInstanceProcAddr of
    // next layer in the chain.
    instance_dispatch_table = new VkLayerInstanceDispatchTable;
    layer_init_instance_dispatch_table(
        *pInstance, my_data->instance_dispatch_table, fpGetInstanceProcAddr);

    // Other layer initialization
    ...

    return VK_SUCCESS;
}
```

#### Example Code for CreateDevice

```cpp
VkResult 
vkCreateDevice(
        VkPhysicalDevice gpu,
        const VkDeviceCreateInfo *pCreateInfo,
        const VkAllocationCallbacks *pAllocator,
        VkDevice *pDevice)
{
    VkLayerDeviceCreateInfo *chain_info =
        get_chain_info(pCreateInfo, VK_LAYER_LINK_INFO);

    PFN_vkGetInstanceProcAddr fpGetInstanceProcAddr =
        chain_info->u.pLayerInfo->pfnNextGetInstanceProcAddr;
    PFN_vkGetDeviceProcAddr fpGetDeviceProcAddr =
        chain_info->u.pLayerInfo->pfnNextGetDeviceProcAddr;
    PFN_vkCreateDevice fpCreateDevice =
        (PFN_vkCreateDevice)fpGetInstanceProcAddr(NULL, "vkCreateDevice");
    if (fpCreateDevice == NULL) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    // Advance the link info for the next element on the chain
    chain_info->u.pLayerInfo = chain_info->u.pLayerInfo->pNext;

    VkResult result = fpCreateDevice(gpu, pCreateInfo, pAllocator, pDevice);
    if (result != VK_SUCCESS) {
        return result;
    }

    // initialize layer's dispatch table
    device_dispatch_table = new VkLayerDispatchTable;
    layer_init_device_dispatch_table(
        *pDevice, device_dispatch_table, fpGetDeviceProcAddr);

    // Other layer initialization
    ...

    return VK_SUCCESS;
}
```


#### Meta-layers

Meta-layers are a special kind of layer which is only available through the
desktop loader.  While normal layers are associated with one particular library,
a meta-layer is actually a collection layer which contains an ordered list of
other layers (called component layers).

The most common example of a meta-layer is the
`VK_LAYER_LUNARG_standard_validation` layer which groups all the most common
individual validation layers into a single layer for ease-of-use.

The benefits of a meta-layer are:
 1. You can activate more than one layer using a single layer name by simply
grouping multiple layers in a meta-layer.
 2. You can define the order the loader will activate individual layers within
the meta-layer.
 3. You can easily share your special layer configuration with others.
 4. The loader will automatically collate all instance and device extensions in
a meta-layer's component layers, and report them as the meta-layer's properties
to the application when queried.
 
Restrictions to defining and using a meta-layer are:
 1. A Meta-layer Manifest file **must** be a properly formated that contains one
or more component layers.
 3. All component layers **must be** present on a system for the meta-layer to
be used.
 4. All component layers **must be** at the same Vulkan API major and minor
version for the meta-layer to be used.
 
The ordering of a meta-layer's component layers in the instance or device
call-chain is simple:
  * The first layer listed will be the layer closest to the application.
  * The last layer listed will be the layer closest to the drivers.

Inside the meta-layer Manifest file, each component layer is listed by its
layer name.  This is the "name" tag's value associated with each component layer's
Manifest file under the "layer" or "layers" tag.  This is also the name that
would normally be used when activating a layer during `vkCreateInstance`.

Any duplicate layer names in either the component layer list, or globally among
all enabled layers, will simply be ignored.  Only the first instance of any
layer name will be used.

For example, if you have a layer enabled using the environment variable
`VK_INSTANCE_LAYERS` and have that same layer listed in a meta-layer, then the
environment variable enabled layer will be used and the component layer will
be dropped.  Likewise, if a person were to enable a meta-layer and then
separately enable one of the component layers afterwards, the second
instantiation of the layer name would be ignored.

The
Manifest file formatting necessary to define a meta-layer can be found in the
[Layer Manifest File Format](#layer-manifest-file-format) section.

#### Special Considerations


##### Associating Private Data with Vulkan Objects Within a Layer

A layer may want to associate it's own private data with one or more Vulkan
objects.  Two common methods to do this are hash maps and object wrapping. 


###### Wrapping

The loader supports layers wrapping any Vulkan object, including dispatchable
objects.  For functions that return object handles, each layer does not touch
the value passed down the call chain.  This is because lower items may need to
use the original value.  However, when the value is returned from a
lower-level layer (possibly the ICD), the layer saves the handle  and returns
its own handle to the layer above it (possibly the application).  When a layer
receives a Vulkan function using something that it previously returned a handle
for, the layer is required to unwrap the handle and pass along the saved handle
to the layer below it.  This means that the layer **must intercept every Vulkan
function which uses the object in question**, and wrap or unwrap the object, as
appropriate.  This includes adding support for all extensions with functions
using any object the layer wraps.

Layers above the object wrapping layer will see the wrapped object. Layers
which wrap dispatchable objects must ensure that the first field in the wrapping
structure is a pointer to a dispatch table as defined in `vk_layer.h`.
Specifically, an instance wrapped dispatchable object could be as follows:
```
struct my_wrapped_instance_obj_ {
    VkLayerInstanceDispatchTable *disp;
    // whatever data layer wants to add to this object
};
```
A device wrapped dispatchable object could be as follows:
```
struct my_wrapped_instance_obj_ {
    VkLayerDispatchTable *disp;
    // whatever data layer wants to add to this object
};
```

Layers that wrap dispatchable objects must follow the guidelines for creating
new dispatchable objects (below).

<u>Cautions About Wrapping</u>

Layers are generally discouraged from wrapping objects, because of the
potential for incompatibilities with new extensions.  For example, let's say
that a layer wraps `VkImage` objects, and properly wraps and unwraps `VkImage`
object handles for all core functions.  If a new extension is created which has
functions that take `VkImage` objects as parameters, and if the layer does not
support those new functions, an application that uses both the layer and the new
extension will have undefined behavior when those new functions are called (e.g.
the application may crash).  This is because the lower-level layers and ICD
won't receive the handle that they generated.  Instead, they will receive a
handle that is only known by the layer that is wrapping the object.

Because of the potential for incompatibilities with unsupported extensions,
layers that wrap objects must check which extensions are being used by the
application, and take appropriate action if the layer is used with unsupported
extensions (e.g. disable layer functionality, stop wrapping objects, issue a
message to the user).

The reason that the validation layers wrap objects, is to track the proper use
and destruction of each object.  They issue a validation error if used with
unsupported extensions, alerting the user to the potential for undefined
behavior.


###### Hash Maps

Alternatively, a layer may want to use a hash map to associate data with a
given object. The key to the map could be the object. Alternatively, for
dispatchable objects at a given level (eg device or instance) the layer may
want data associated with the `VkDevice` or `VkInstance` objects. Since
there are multiple dispatchable objects for a given `VkInstance` or `VkDevice`,
the `VkDevice` or `VkInstance` object is not a great map key. Instead the layer
should use the dispatch table pointer within the `VkDevice` or `VkInstance`
since that will be unique for a given `VkInstance` or `VkDevice`.


##### Creating New Dispatchable Objects

Layers which create dispatchable objects must take special care. Remember that
loader *trampoline* code normally fills in the dispatch table pointer in the
newly created object. Thus, the layer must fill in the dispatch table pointer if
the loader *trampoline* will not do so.  Common cases where a layer (or ICD) may
create a dispatchable object without loader *trampoline* code is as follows:
- layers that wrap dispatchable objects
- layers which add extensions that create dispatchable objects
- layers which insert extra Vulkan functions in the stream of functions they
intercept from the application
- ICDs which add extensions that create dispatchable objects

The desktop loader provides a callback that can be used for initializing
a dispatchable object.  The callback is passed as an extension structure via the
pNext field in the create info structure when creating an instance
(`VkInstanceCreateInfo`) or device (`VkDeviceCreateInfo`).  The callback
prototype is defined as follows for instance and device callbacks respectively
(see `vk_layer.h`):

```cpp
VKAPI_ATTR VkResult VKAPI_CALL vkSetInstanceLoaderData(VkInstance instance,
                                                       void *object);
VKAPI_ATTR VkResult VKAPI_CALL vkSetDeviceLoaderData(VkDevice device,
                                                     void *object);
```

To obtain these callbacks the layer must search through the list of structures
pointed to by the "pNext" field in the `VkInstanceCreateInfo` and
`VkDeviceCreateInfo` parameters to find any callback structures inserted by the
loader. The salient details are as follows:
- For `VkInstanceCreateInfo` the callback structure pointed to by "pNext" is
`VkLayerInstanceCreateInfo` as defined in `include/vulkan/vk_layer.h`.
- A "sType" field in of VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO within
`VkInstanceCreateInfo` parameter indicates a loader structure.
- Within `VkLayerInstanceCreateInfo`, the "function" field indicates how the
union field "u" should be interpreted.
- A "function" equal to VK_LOADER_DATA_CALLBACK indicates the "u" field will
contain the callback in "pfnSetInstanceLoaderData".
- For `VkDeviceCreateInfo` the callback structure pointed to by "pNext" is
`VkLayerDeviceCreateInfo` as defined in `include/vulkan/vk_layer.h`.
- A "sType" field in of VK_STRUCTURE_TYPE_LOADER_DEVICE_CREATE_INFO within
`VkDeviceCreateInfo` parameter indicates a loader structure.
- Within `VkLayerDeviceCreateInfo`, the "function" field indicates how the union
field "u" should be interpreted.
- A "function" equal to VK_LOADER_DATA_CALLBACK indicates the "u" field will
contain the callback in "pfnSetDeviceLoaderData".

Alternatively, if an older loader is being used that doesn't provide these
callbacks, the layer may manually initialize the newly created dispatchable
object.  To fill in the dispatch table pointer in newly created dispatchable
object, the layer should copy the dispatch pointer, which is always the first
entry in the structure, from an existing parent object of the same level
(instance versus device).

For example, if there is a newly created `VkCommandBuffer` object, then the
dispatch pointer from the `VkDevice` object, which is the parent of the
`VkCommandBuffer` object, should be copied into the newly created object.


#### Layer Manifest File Format

On Windows and Linux (desktop), the loader uses manifest files to discover
layer libraries and layers.  The desktop loader doesn't directly query the
layer library except during chaining.  This is to reduce the likelihood of
loading a malicious layer into memory.  Instead, details are read from the
Manifest file, which are then provided for applications to determine what
layers should actually be loaded.

The following section discusses the details of the Layer Manifest JSON file
format.  The JSON file itself does not have any requirements for naming.  The
only requirement is that the extension suffix of the file ends with ".json".

Here is an example layer JSON Manifest file with a single layer:

```
{
   "file_format_version" : "1.0.0",
   "layer": {
       "name": "VK_LAYER_LUNARG_overlay",
       "type": "INSTANCE",
       "library_path": "vkOverlayLayer.dll"
       "api_version" : "1.0.5",
       "implementation_version" : "2",
       "description" : "LunarG HUD layer",
       "functions": {
           "vkNegotiateLoaderLayerInterfaceVersion":
               "OverlayLayer_NegotiateLoaderLayerInterfaceVersion"
       },
       "instance_extensions": [
           {
               "name": "VK_EXT_debug_report",
               "spec_version": "1"
           },
           {
               "name": "VK_VENDOR_ext_x",
               "spec_version": "3"
            }
       ],
       "device_extensions": [
           {
               "name": "VK_EXT_debug_marker",
               "spec_version": "1",
               "entrypoints": ["vkCmdDbgMarkerBegin", "vkCmdDbgMarkerEnd"]
           }
       ],
       "enable_environment": {
           "ENABLE_LAYER_OVERLAY_1": "1"
       }
       "disable_environment": {
           "DISABLE_LAYER_OVERLAY_1": ""
       }
   }
}
```

Here's a snippet with the changes required to support multiple layers per
manifest file:
```
{
   "file_format_version" : "1.0.1",
   "layers": [
      {
           "name": "VK_LAYER_layer_name1",
           "type": "INSTANCE",
           ...
      },
      {
           "name": "VK_LAYER_layer_name2",
           "type": "INSTANCE",
           ...
      }
   ]
}
```

Here's an example of a meta-layer manifest file:
```
{
   "file_format_version" : "1.1.1",
   "layer": {
       "name": "VK_LAYER_LUNARG_standard_validation",
       "type": "GLOBAL",
       "api_version" : "1.0.40",
       "implementation_version" : "1",
       "description" : "LunarG Standard Validation Meta-layer",
       "component_layers": [
           "VK_LAYER_GOOGLE_threading",
           "VK_LAYER_LUNARG_parameter_validation",
           "VK_LAYER_LUNARG_object_tracker",
           "VK_LAYER_LUNARG_core_validation",
           "VK_LAYER_GOOGLE_unique_objects"
       ]
   }
}
```
| JSON Node | Description and Notes | Introspection Query |
|:----------------:|--------------------|:----------------:
| "file\_format\_version" | Manifest format major.minor.patch version number. | N/A |
| | Supported versions are: 1.0.0, 1.0.1, and 1.1.0. | |
| "layer" | The identifier used to group a single layer's information together. | vkEnumerateInstanceLayerProperties |
| "layers" | The identifier used to group multiple layers' information together.  This requires a minimum Manifest file format version of 1.0.1.| vkEnumerateInstanceLayerProperties |
| "name" | The string used to uniquely identify this layer to applications. | vkEnumerateInstanceLayerProperties |
| "type" | This field indicates the type of layer.  The values can be: GLOBAL, or INSTANCE | vkEnumerate*LayerProperties |
|  | **NOTES:** Prior to deprecation, the "type" node was used to indicate which layer chain(s) to activate the layer upon: instance, device, or both. Distinct instance and device layers are deprecated; there are now just layers. Allowable values for type (both before and after deprecation) are "INSTANCE", "GLOBAL" and, "DEVICE." "DEVICE" layers are skipped over by the loader as if they were not found. |  |
| "library\_path" | The "library\_path" specifies either a filename, a relative pathname, or a full pathname to a layer shared library file.  If "library\_path" specifies a relative pathname, it is relative to the path of the JSON manifest file (e.g. for cases when an application provides a layer that is in the same folder hierarchy as the rest of the application files).  If "library\_path" specifies a filename, the library must live in the system's shared object search path. There are no rules about the name of the layer shared library files other than it should end with the appropriate suffix (".DLL" on Windows, and ".so" on Linux).  **This field must not be present if "component_layers" is defined**  | N/A |
| "api\_version" | The major.minor.patch version number of the Vulkan API that the shared library file for the library was built against. For example: 1.0.33. | vkEnumerateInstanceLayerProperties |
| "implementation_version" | The version of the layer implemented.  If the layer itself has any major changes, this number should change so the loader and/or application can identify it properly. | vkEnumerateInstanceLayerProperties |
| "description" | A high-level description of the layer and it's intended use. | vkEnumerateInstanceLayerProperties |
| "functions" | **OPTIONAL:** This section can be used to identify a different function name for the loader to use in place of standard layer interface functions. The "functions" node is required if the layer is using an alternative name for `vkNegotiateLoaderLayerInterfaceVersion`. | vkGet*ProcAddr |
| "instance\_extensions" | **OPTIONAL:** Contains the list of instance extension names supported by this layer. One "instance\_extensions" node with an array of one or more elements is required if any instance extensions are supported by a layer, otherwise the node is optional. Each element of the array must have the nodes "name" and "spec_version" which correspond to `VkExtensionProperties` "extensionName" and "specVersion" respectively. | vkEnumerateInstanceExtensionProperties |
| "device\_extensions" | **OPTIONAL:** Contains the list of device extension names supported by this layer. One "device_\extensions" node with an array of one or more elements is required if any device extensions are supported by a layer, otherwise the node is optional. Each element of the array must have the nodes "name" and "spec_version" which correspond to `VkExtensionProperties` "extensionName" and "specVersion" respectively. Additionally, each element of the array of device extensions must have the node "entrypoints" if the device extension adds Vulkan API functions, otherwise this node is not required. The "entrypoint" node is an array of the names of all entrypoints added by the supported extension. | vkEnumerateDeviceExtensionProperties |
| "enable\_environment" | **Implicit Layers Only** - **OPTIONAL:** Indicates an environment variable used to enable the Implicit Layer (w/ value of 1).  This environment variable (which should vary with each "version" of the layer) must be set to the given value or else the implicit layer is not loaded. This is for application environments (e.g. Steam) which want to enable a layer(s) only for applications that they launch, and allows for applications run outside of an application environment to not get that implicit layer(s).| N/A |
| "disable\_environment" | **Implicit Layers Only** - **REQUIRED:**Indicates an environment variable used to disable the Implicit Layer (w/ value of 1). In rare cases of an application not working with an implicit layer, the application can set this environment variable (before calling Vulkan functions) in order to "blacklist" the layer. This environment variable (which should vary with each "version" of the layer) must be set (not particularly to any value). If both the "enable_environment" and "disable_environment" variables are set, the implicit layer is disabled. | N/A |
| "component_layers" | **Meta-layers Only** - Indicates the component layer names that are part of a meta-layer.  The names listed must be the "name" identified in each of the component layer's Mainfest file "name" tag (this is the same as the name of the layer that is passed to the `vkCreateInstance` command).  All component layers must be present on the system and found by the loader in order for this meta-layer to be available and activated. **This field must not be present if "library\_path" is defined** | N/A |

##### Layer Manifest File Version History

The current highest supported Layer Manifest file format supported is 1.1.0.
Information about each version is detailed in the following sub-sections:

###### Layer Manifest File Version 1.1.0

Layer Manifest File Version 1.1.0 is tied to changes exposed by the Loader/Layer
interface version 2.
  1. Renaming "vkGetInstanceProcAddr" in the "functions" section is
     deprecated since the loader no longer needs to query the layer about
     "vkGetInstanceProcAddr" directly.  It is now returned during the layer
     negotiation, so this field will be ignored.
  2. Renaming "vkGetDeviceProcAddr" in the "functions" section is
     deprecated since the loader no longer needs to query the layer about
     "vkGetDeviceProcAddr" directly.  It too is now returned during the layer
     negotiation, so this field will be ignored.
  3. Renaming the "vkNegotiateLoaderLayerInterfaceVersion" function is
     being added to the "functions" section, since this is now the only
     function the loader needs to query using OS-specific calls.
      - NOTE: This is an optional field and, as the two previous fields, only
needed if the layer requires changing the name of the function for some reason.

You do not need to update your layer manifest file if you don't change the
names of any of the listed functions.

###### Layer Manifest File Version 1.0.1

The ability to define multiple layers using the "layers" array was added.  This
JSON array field can be used when defining a single layer or multiple layers.
The "layer" field is still present and valid for a single layer definition.

###### Layer Manifest File Version 1.0.0

The initial version of the layer manifest file specified the basic format and
fields of a layer JSON file.  The fields of the 1.0.0 file format include:
 * "file\_format\_version"
 * "layer"
 * "name"
 * "type"
 * "library\_path"
 * "api\_version"
 * "implementation\_version"
 * "description"
 * "functions"
 * "instance\_extensions"
 * "device\_extensions"
 * "enable\_environment"
 * "disable\_environment"

It was also during this time that the value of "DEVICE" was deprecated from
the "type" field.


#### Layer Library Versions

The current Layer Library interface is at version 2.  The following sections
detail the differences between the various versions.

##### Layer Library API Version 2

Introduced the concept of
[loader and layer interface](#layer-version-negotiation) using the new
`vkNegotiateLoaderLayerInterfaceVersion` function. Additionally, it introduced
the concept of
[Layer Unknown Physical Device Extensions](#layer-unknown-physical-device-
extensions)
and the associated `vk_layerGetPhysicalDeviceProcAddr` function.  Finally, it
changed the manifest file defition to 1.1.0.

##### Layer Library API Version 1

A layer library supporting interface version 1 had the following behavior:
 1. `GetInstanceProcAddr` and `GetDeviceProcAddr` were directly exported
 2. The layer manifest file was able to override the names of the
`GetInstanceProcAddr` and `GetDeviceProcAddr`functions.

##### Layer Library API Version 0

A layer library supporting interface version 0 must define and export these
introspection functions, unrelated to any Vulkan function despite the names,
signatures, and other similarities:

- `vkEnumerateInstanceLayerProperties` enumerates all layers in a layer
library.
  - This function never fails.
  - When a layer library contains only one layer, this function may be an alias
   to the layer's `vkEnumerateInstanceLayerProperties`.
- `vkEnumerateInstanceExtensionProperties` enumerates instance extensions of
   layers in a layer library.
  - "pLayerName" is always a valid layer name.
  - This function never fails.
  - When a layer library contains only one layer, this function may be an alias
   to the layer's `vkEnumerateInstanceExtensionProperties`.
- `vkEnumerateDeviceLayerProperties` enumerates a subset (can be full,
   proper, or empty subset) of layers in a layer library.
  - "physicalDevice" is always `VK_NULL_HANDLE`.
  - This function never fails.
  - If a layer is not enumerated by this function, it will not participate in
   device function interception.
- `vkEnumerateDeviceExtensionProperties` enumerates device extensions of
   layers in a layer library.
  - "physicalDevice" is always `VK_NULL_HANDLE`.
  - "pLayerName" is always a valid layer name.
  - This function never fails.

It must also define and export these functions once for each layer in the
library:

- `<layerName>GetInstanceProcAddr(instance, pName)` behaves identically to a
layer's vkGetInstanceProcAddr except it is exported.

   When a layer library contains only one layer, this function may
   alternatively be named `vkGetInstanceProcAddr`.

- `<layerName>GetDeviceProcAddr`  behaves identically to a layer's
vkGetDeviceProcAddr except it is exported.

   When a layer library contains only one layer, this function may
   alternatively be named `vkGetDeviceProcAddr`.

All layers contained within a library must support `vk_layer.h`.  They do not
need to implement functions that they do not intercept.  They are recommended
not to export any functions.


<br/>
<br/>

## Vulkan Installable Client Driver Interface With the Loader

This section discusses the various requirements for the loader and a Vulkan
ICD to properly hand-shake.

  * [ICD Discovery](#icd-discovery)
    * [Overriding the Default ICD Usage](#overriding-the-default-icd-usage)
    * [ICD Manifest File Usage](#icd-manifest-file-usage)
    * [ICD Discovery on Windows](#icd-discovery-on-windows)
    * [ICD Discovery on Linux](#icd-discovery-on-linux)
    * [Using Pre-Production ICDs on Windows and Linux](#using-pre-production-icds-on-windows-and-linux)
    * [ICD Discovery on Android](#icd-discovery-on-android)
  * [ICD Manifest File Format](#icd-manifest-file-format)
    * [ICD Manifest File Versions](#icd-manifest-file-versions)
      * [ICD Manifest File Version 1.0.0](#icd-manifest-file-version-1.0.0)
  * [ICD Vulkan Entry-Point Discovery](#icd-vulkan-entry-point-discovery)
  * [ICD Unknown Physical Device Extensions](#icd-unknown-physical-device-extensions)
  * [ICD Dispatchable Object Creation](#icd-dispatchable-object-creation)
  * [Handling KHR Surface Objects in WSI Extensions](#handling-khr-surface-objects-in-wsi-extensions)
  * [Loader and ICD Interface Negotiation](#loader-and-icd-interface-negotiation)
    * [Windows and Linux ICD Negotiation](#windows-and-linux-icd-negotiation)
      * [Version Negotiation Between Loader and ICDs](#version-negotiation-between-loader-and-icds)
        * [Interfacing With Legacy ICDs or Loader](#interfacing-with-legacy-icds-or-loader)
      * [Loader Version 5 Interface Requirements](#loader-version-5-interface-requirements)
      * [Loader Version 4 Interface Requirements](#loader-version-4-interface-requirements)
      * [Loader Version 3 Interface Requirements](#loader-version-3-interface-requirements)
      * [Loader Version 2 Interface Requirements](#loader-version-2-interface-requirements)
      * [Loader Versions 0 and 1 Interface Requirements](#loader-versions-0-and-1-interface-requirements)
    * [Android ICD Negotiation](#android-icd-negotiation)


### ICD Discovery

Vulkan 允许多个驱动对应着一个或多个设备（由`VkPhysicalDevice`对象表示）一起被使用。加载器负责寻找系统中可用的Vulkan　ICD。
给定了一个列表的ICDs，加载遍历所有的物理设备并给应用程序返回可用的信息。这个由加载器发现系统上可用的 Installable Client Drivers (ICDs) 过程依赖于系统。
Windows、Linux和Android ICD搜寻的差异细节如下。

#### Overriding the Default ICD Usage

There may be times that a developer wishes to force the loader to use a specific ICD.
This could be for many reasons including : using a beta driver, or forcing the loader
to skip a problematic ICD.  In order to support this, the loader can be forced to
look at specific ICDs with the `VK_ICD_FILENAMES` environment variable.  In order
to use the setting, simply set it to a properly delimited list of ICD Manifest
files that you wish to use.  In this case, please provide the global path to these
files to reduce issues.

For example:

##### On Windows

```
set VK_ICD_FILENAMES=/windows/system32/nv-vk64.json
```

This is an example which is using the `VK_ICD_FILENAMES` override on Windows to point
to the Nvidia Vulkan driver's ICD Manifest file.

##### On Linux

```
export VK_ICD_FILENAMES=/home/user/dev/mesa/share/vulkan/icd.d/intel_icd.x86_64.json
```

This is an example which is using the `VK_ICD_FILENAMES` override on Linux to point
to the Intel Mesa driver's ICD Manifest file.


#### ICD Manifest File Usage

和layers相同，在Windows和Linux系统上，JSON格式的明细文件被用来存储ICD信息。为了找到系统安装的驱动，Vulkan加载器将读取JSON文件来指定每一个驱动的名字和属性。
你需要注意的一点是ICD明细文件比对应的layer明细文件要简单的多。

参考 [Current ICD Manifest File Format](#icd-manifest-file-format) 小节获取更多细节信息。


#### ICD Discovery on Windows
为了寻找已安装的ICD，加载器扫描显示适配器对应的注册表key和这些适配器相关 的软件组件对应的JSON明细文件。这些注册表key都在device keys中，在驱动程序安装时
创建，包含了基础的配置信息，如OpenGL和Direct3D ICD位置。

设备适配器和软件组件的key路径应通过PnP配置管理器API来获取。`000X` key是一个可编号的key，每一个设备都被赋予一个不同的号码。

```
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanDriverName
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{SoftwareComponent GUID}\000X\VulkanDriverName
```

此外，在64-bit系统上，也许有另外一套注册值，如下所示。这些值记录了64-bit系统上32-bit layer的路径，和Windows-on-Windows功能类似。

```
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanDriverNameWow
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{SoftwareComponent GUID}\000X\VulkanDriverNameWow
```

If any of the above values exist and is of type `REG_SZ`, the loader will open the JSON
manifest file specified by the key value. Each value must be a full absolute
path to a JSON manifest file. The values may also be of type `REG_MULTI_SZ`, in
which case the value will be interpreted as a list of paths to JSON manifest files.

Additionally, the Vulkan loader will scan the values in the following Windows registry key:

```
   HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\Drivers
```
对于64-bit系统上32-bit应用程序，加载器扫描32-bit注册表条目位置：

```
   HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Khronos\Vulkan\Drivers
```

Every ICD in these locations should be given as a DWORD, with value 0, where
the name of the value is the full path to a JSON manifest file. The Vulkan loader
will attempt to open each manifest file to obtain the information about an ICD's
shared library (".dll") file.

For example, let us assume the registry contains the following data:

```
[HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\Drivers\]

"C:\vendor a\vk_vendora.json"=dword:00000000
"C:\windows\system32\vendorb_vk.json"=dword:00000001
"C:\windows\system32\vendorc_icd.json"=dword:00000000
```

In this case, the loader will step through each entry, and check the value.  If
the value is 0, then the loader will attempt to load the file.  In this case,
the loader will open the first and last listings, but not the middle.  This
is because the value of 1 for vendorb_vk.json disables the driver.

The Vulkan loader will open each enabled manifest file found to obtain the name
or pathname of an ICD shared library (".DLL") file.

ICDs should use the registry locations from the PnP Configuration Manager wherever
practical. That location clearly ties the ICD to a given device. The
`SOFTWARE\Khronos\Vulkan\Drivers` location is the older method for locating ICDs,
and is retained for backwards compatibility.

See the [ICD Manifest File Format](#icd-manifest-file-format) section for more
details.


#### ICD Discovery on Linux

为了找到安装的ICDs，Vulkan加载器扫描如下Linux目录：

```
    /usr/local/etc/vulkan/icd.d
    /usr/local/share/vulkan/icd.d
    /etc/vulkan/icd.d
    /usr/share/vulkan/icd.d
    $HOME/.local/share/vulkan/icd.d
```

The "/usr/local/*" directories can be configured to be other directories at
build time.

目录的典型用途如下表所示：

| Location  |  Details |
|-------------------|------------------------|
| $HOME/.local/share/vulkan/icd.d | $HOME is the current home directory of the application's user id; this path will be ignored for suid programs |
| "/usr/local/etc/vulkan/icd.d" | Directory for locally built ICDs |
| "/usr/local/share/vulkan/icd.d" | Directory for locally built ICDs |
| "/etc/vulkan/icd.d" | Location of ICDs installed from non-Linux-distribution-provided packages |
| "/usr/share/vulkan/icd.d" | Location of ICDs installed from Linux-distribution-provided packages |

The Vulkan loader will open each manifest file found to obtain the name or
pathname of an ICD shared library (".so") file.

See the [ICD Manifest File Format](#icd-manifest-file-format) section for more
details.

##### Additional Settings For ICD Debugging

If you are seeing issues which may be related to the ICD.  A possible option to debug is to enable the
`LD_BIND_NOW` environment variable.  This forces every dynamic library's symbols to be fully resolved on load.  If
there is a problem with an ICD missing symbols on your system, this will expose it and cause the Vulkan loader
to fail on loading the ICD.  It is recommended that you enable `LD_BIND_NOW` along with `VK_LOADER_DEBUG=warn`
to expose any issues.

#### Using Pre-Production ICDs on Windows and Linux

独立设备提供商（IHV）预发布版本ICDs。在一些情形下，一个预发布ICD也可以是一个可安装的包。在其他情形下，一个预发布ICD也许只是一个指向开发版本的
连接库。在后者中，我们想要允许开发者可以指向该ICD而无需修改系统安装的ICD。

这个需求可以使用"VK\_ICD\_FILENAMES" 环境变量来实现，它将覆盖寻找系统已安装ICD机制。亦即，只有在"VK\_ICD\_FILENAMES" 中的ICDs才会被使用。

 "VK\_ICD\_FILENAMES" 环境变量是一个包含了ICD明细文件的列表，包含了ICD JSON明细文件的全路径。
这个列表在Linux上有冒号分隔，在Windows上用分号分隔。 

通常， "VK\_ICD\_FILENAMES" 只会包含一个指向开发者编译的ICD。分隔符（冒号或者分号）只会在列表中有多个ICD时使用。

**NOTE:** On Linux, this environment variable will be ignored for suid programs.


#### ICD Discovery on Android

Android 加载器在系统库目录。位置不可被更改。加载器将通过以“vulkan”为ID用 hw\_get\_module来获取驱动/ICD。 
** 因为Android系统的安全策略，在正常使用中这些东西都不可被修改。**


### ICD Manifest File Format

如下小节讨论了ICD JSON 明细文件的格式。 JSON文件自身没有名字的限制。仅有的要求是文件的后缀必须是 ".json"。

如下是一个ICD JSON明细文件的例子：

```
{
   "file_format_version": "1.0.0",
   "ICD": {
      "library_path": "path to ICD library",
      "api_version": "1.0.5"
   }
}
```

| Field Name | Field Value |
|----------------|--------------------|
| "file\_format\_version" | The JSON format major.minor.patch version number of this file.  Currently supported version is 1.0.0. |
| "ICD" | The identifier used to group all ICD information together. |
| "library_path" | The "library\_path" specifies either a filename, a relative pathname, or a full pathname to a layer shared library file.  If "library\_path" specifies a relative pathname, it is relative to the path of the JSON manifest file.  If "library\_path" specifies a filename, the library must live in the system's shared object search path. There are no rules about the name of the ICD shared library files other than it should end with the appropriate suffix (".DLL" on Windows, and ".so" on Linux). | N/A |
| "api_version" | The major.minor.patch version number of the Vulkan API that the shared library files for the ICD was built against. For example: 1.0.33. |

**NOTE:** If the same ICD shared library supports multiple, incompatible
versions of text manifest file format versions, it must have separate
JSON files for each (all of which may point to the same shared library).

##### ICD Manifest File Versions

There has only been one version of the ICD manifest files supported.  This is
version 1.0.0.

###### ICD Manifest File Version 1.0.0

The initial version of the ICD Manifest file specified the basic format and
fields of a layer JSON file.  The fields of the 1.0.0 file format include:
 * "file\_format\_version"
 * "ICD"
 * "library\_path"
 * "api\_version"

 
###  ICD Vulkan Entry-Point Discovery

ICD导出的Vulkan符号不能和加载器导出的Vulkan符号冲突。有几个原因。因此，所有的ICD必须必须导出如下的函数，用来发现ICD函数指针。
此函数指针并不是Vulkan API的一部分，只是加载器和ICD版本1或更高之间的私有接口。

```cpp
VKAPI_ATTR PFN_vkVoidFunction VKAPI_CALL vk_icdGetInstanceProcAddr(
                                               VkInstance instance,
                                               const char* pName);
```
这个函数和 `vkGetInstanceProcAddr`有非常相似的语义。`vk_icdGetInstanceProcAddr` 返回个有效的函数指针，
指向了全局和实例级Vulkan函数，也可指向 `vkGetDeviceProcAddr`。
Global-level functions are those which contain no dispatchable object as the
first parameter, such as `vkCreateInstance` and
`vkEnumerateInstanceExtensionProperties`. The ICD must support querying global-
level entry-points by calling `vk_icdGetInstanceProcAddr` with a NULL
`VkInstance` parameter. Instance-level functions are those that have either
`VkInstance`, or `VkPhysicalDevice` as the first parameter dispatchable object.
Both core entry-points and any instance extension entry-points the ICD supports
should be available via `vk_icdGetInstanceProcAddr`. Future Vulkan instance
extensions may define and use new instance-level dispatchable objects other
than `VkInstance` and `VkPhysicalDevice`, in which case extension entry-points
using these newly defined dispatchable objects must be queryable via
`vk_icdGetInstanceProcAddr`.

其他的Vulkan函数指针必须：
 * 不能通过ICD库直接导出
 * 或导出的名字不同使用官方Vulkan函数名字
 
这个要求并不针对ICD库包含的其他功能（如OpenGL）和应用程序在载入Vulkan之前就载入的功能。

如果使用了Vulkan函数名字，对系统动态加载库加载的符号一定要小心。在Linux上，如果使用了Vulkan函数名字，ICD库必须通过-Bsymbolic来链接。


### ICD Unknown Physical Device Extensions


起始时，若加载器是通过调用 `vkGetInstanceProcAddr`才使用到的，它将导致如下行为：
 1. 加载器将检查核心函数：
    - 如果是核心函数，加载器将返回函数指针
 2. 加载器将检查已知的拓展函数：
    - 如果是拓展函数，加载器将返回函数指针
 3. 若加载器不知道它的信息，它将按照如下方式来使用`GetInstanceProcAddr`
    - 如果他返回non-NULL，把它动作未知的逻辑设备命令
    - 这意味着创建了一个通用的trampoline函数，接受VKDevice做为第一个参数，在从VKDevice中获取到转发表后，调整转发表来调用ICD/Layers 函数。
 4. 若以上都失败了，加载器将向应用程序返回  NULL 。

This caused problems when an ICD attempted to expose new physical device
extensions the loader knew nothing about, but an application did.  Because the
loader knew nothing about it, the loader would get to step 3 in the above
process and would treat the function as an unknown logical device command.  The
problem is, this would create a generic VkDevice trampoline function which, on
the first call, would attempt to dereference the VkPhysicalDevice as a VkDevice.
This would lead to a crash or corruption.

In order to identify the extension entry-points specific to physical device
extensions, the following function can be added to an ICD:

```cpp
PFN_vkVoidFunction vk_icdGetPhysicalDeviceProcAddr(VkInstance instance,
                                                   const char* pName);
```

This function behaves similar to `vkGetInstanceProcAddr` and
`vkGetDeviceProcAddr` except it should only return values for physical device
extension entry-points.  In this way, it compares "pName" to every physical
device function supported in the ICD.

The following rules apply:
* If it is the name of a physical device function supported by the ICD, the
pointer to the ICD's corresponding function should be returned.
* If it is the name of a valid function which is **not** a physical device
function (i.e. an Instance, Device, or other function implemented by the ICD),
then the value of NULL should be returned.
* If the ICD has no idea what this function is, it should return NULL.

This support is optional and should not be considered a requirement.  This is
only required if an ICD intends to support some functionality not directly
supported by a significant population of loaders in the public.  If an ICD
does implement this support, it should return the address of its
`vk_icdGetPhysicalDeviceProcAddr` function through the `vkGetInstanceProcAddr`
function.

The new behavior of the loader's vkGetInstanceProcAddr with support for the
`vk_icdGetPhysicalDeviceProcAddr` function is as follows:
 1. Check if core function:
    - If it is, return the function pointer
 2. Check if known instance or device extension function:
    - If it is, return the function pointer
 3. Call the layer/ICD `GetPhysicalDeviceProcAddr`
    - If it returns non-NULL, return a trampoline to a generic physical device
function, and setup a generic terminator which will pass it to the proper ICD.
 4. Call down using `GetInstanceProcAddr`
    - If it returns non-NULL, treat it as an unknown logical device command.
This means setting up a generic trampoline function that takes in a VkDevice as
the first parameter and adjusting the dispatch table to call the ICD/Layers
function after getting the dispatch table from the VkDevice. Then, return the
pointer to corresponding trampoline function.
 5. Return NULL

You can see now, that, if the command gets promoted to core later, it will no
longer be setup using `vk_icdGetPhysicalDeviceProcAddr`.  Additionally, if the
loader adds direct support for the extension, it will no longer get to step 3,
because step 2 will return a valid function pointer.  However, the ICD should
continue to support the command query via `vk_icdGetPhysicalDeviceProcAddr`,
until at least a Vulkan version bump, because an older loader may still be
attempting to use the commands.


### ICD Dispatchable Object Creation

如前面提到过的，加载器要求转发表在Vulkan可转发对象内部可访问到，如：`VkInstance`, `VkPhysicalDevice`,
`VkDevice`, `VkQueue`, and `VkCommandBuffer`。此条限制对ICD创建的可转发对象要求如下：

- 所有被ICD创建的可转发对象可以被强制类型转换为 void \*\*
- 加载器将替换替换第一个成员为加载器的转发表的指针。这对ICD驱动来说意味三点：
  1. ICD必须返回一个不可见的可转发对象的handle
  2. 这个指针必须指向一个常规的C结构体，第一个成员是一个指针。
   * **NOTE:** For any C\++ ICD's that implement VK objects directly as C\++
classes.
     * The C\++ compiler may put a vtable at offset zero if your class is non-
POD due to the use of a virtual function.
     * In this case use a regular C structure (see below).
  3. 加载器检查 被创建的不可转发对象的魔法数字(ICD\_LOADER\_MAGIC) ，参考 ( `include/vulkan/vk_icd.h`):

```cpp
#include "vk_icd.h"

union _VK_LOADER_DATA {
    uintptr loadermagic;
    void *loaderData;
} VK_LOADER_DATA;

vkObj alloc_icd_obj()
{
    vkObj *newObj = alloc_obj();
    ...
    // Initialize pointer to loader's dispatch table with ICD_LOADER_MAGIC

    set_loader_magic_value(newObj);
    ...
    return newObj;
}
```
 

### Handling KHR Surface Objects in WSI Extensions

Normally, ICDs handle object creation and destruction for various Vulkan
objects. The WSI surface extensions for Linux and Windows
("VK\_KHR\_win32\_surface", "VK\_KHR\_xcb\_surface", "VK\_KHR\_xlib\_surface",
"VK\_KHR\_mir\_surface", "VK\_KHR\_wayland\_surface", and "VK\_KHR\_surface")
are handled differently.  For these extensions, the `VkSurfaceKHR` object
creation and destruction may be handled by either the loader, or an ICD.

If the loader handles the management of the `VkSurfaceKHR` objects:
 1. The loader will handle the calls to `vkCreateXXXSurfaceKHR` and
`vkDestroySurfaceKHR`
    functions without involving the ICDs.
    * Where XXX stands for the Windowing System name:
      * Mir
      * Wayland
      * Xcb
      * Xlib
      * Windows
      * Android
 2. The loader creates a `VkIcdSurfaceXXX` object for the corresponding
`vkCreateXXXSurfaceKHR` call.
    * The `VkIcdSurfaceXXX` structures are defined in `include/vulkan/vk_icd.h`.
 3. ICDs can cast any `VkSurfaceKHR` object to a pointer to the appropriate
    `VkIcdSurfaceXXX` structure.
 4. The first field of all the `VkIcdSurfaceXXX` structures is a
`VkIcdSurfaceBase` enumerant that indicates whether the
    surface object is Win32, Xcb, Xlib, Mir, or Wayland.

The ICD may choose to handle `VkSurfaceKHR` object creation instead.  If an ICD
desires to handle creating and destroying it must do the following:
 1. Support version 3 or newer of the loader/ICD interface.
 2. Export and handle all functions that take in a `VkSurfaceKHR` object,
including:
     * `vkCreateXXXSurfaceKHR`
     * `vkGetPhysicalDeviceSurfaceSupportKHR`
     * `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
     * `vkGetPhysicalDeviceSurfaceFormatsKHR`
     * `vkGetPhysicalDeviceSurfacePresentModesKHR`
     * `vkCreateSwapchainKHR`
     * `vkDestroySurfaceKHR`

Because the `VkSurfaceKHR` object is an instance-level object, one object can be
associated with multiple ICDs.  Therefore, when the loader receives the
`vkCreateXXXSurfaceKHR` call, it still creates an internal `VkSurfaceIcdXXX`
object.  This object acts as a container for each ICD's version of the
`VkSurfaceKHR` object.  If an ICD does not support the creation of its own
`VkSurfaceKHR` object, the loader's container stores a NULL for that ICD.  On
the otherhand, if the ICD does support `VkSurfaceKHR` creation, the loader will
make the appropriate `vkCreateXXXSurfaceKHR` call to the ICD, and store the
returned pointer in it's container object.  The loader then returns the
`VkSurfaceIcdXXX` as a `VkSurfaceKHR` object back up the call chain.  Finally,
when the loader receives the `vkDestroySurfaceKHR` call, it subsequently calls
`vkDestroySurfaceKHR` for each ICD who's internal `VkSurfaceKHR` object is not
NULL.  Then the loader destroys the container object before returning.


### Loader and ICD Interface Negotiation

Generally, for functions issued by an application, the loader can be
viewed as a pass through. That is, the loader generally doesn't modify the
functions or their parameters, but simply calls the ICDs entry-point for that
function. There are specific additional interface requirements an ICD needs to
comply with that are not part of any requirements from the Vulkan specification.
These addtional requirements are versioned to allow flexibility in the future.


#### Windows and Linux ICD Negotiation


##### Version Negotiation Between Loader and ICDs

All ICDs (supporting interface version 2 or higher) must export the following
function that is used for determination of the interface version that will be
used.  This entry-point is not a part of the Vulkan API itself, only a private
interface between the loader and ICDs.

```cpp
   VKAPI_ATTR VkResult VKAPI_CALL
       vk_icdNegotiateLoaderICDInterfaceVersion(
           uint32_t* pSupportedVersion);
```

This function allows the loader and ICD to agree on an interface version to use.
The "pSupportedVersion" parameter is both an input and output parameter.
"pSupportedVersion" is filled in by the loader with the desired latest interface
version supported by the loader (typically the latest). The ICD receives this
and returns back the version it desires in the same field.  Because it is
setting up the interface version between the loader and ICD, this should be
the first call made by a loader to the ICD (even prior to any calls to
`vk_icdGetInstanceProcAddr`).

If the ICD receiving the call no longer supports the interface version provided
by the loader (due to deprecation), then it should report
VK_ERROR_INCOMPATIBLE_DRIVER error.  Otherwise it sets the value pointed by
"pSupportedVersion" to the latest interface version supported by both the ICD
and the loader and returns VK_SUCCESS.

The ICD should report VK_SUCCESS in case the loader provided interface version
is newer than that supported by the ICD, as it's the loader's responsibility to
determine whether it can support the older interface version supported by the
ICD.  The ICD should also report VK_SUCCESS in the case its interface version
is greater than the loader's, but return the loader's version. Thus, upon
return of VK_SUCCESS the "pSupportedVersion" will contain the desired interface
version to be used by the ICD.

If the loader receives an interface version from the ICD that the loader no
longer supports (due to deprecation), or it receives a
VK_ERROR_INCOMPATIBLE_DRIVER error instead of VK_SUCCESS, then the loader will
treat the ICD as incompatible and will not load it for use.  In this case, the
application will not see the ICDs `vkPhysicalDevice` during enumeration.

###### Interfacing With Legacy ICDs or Loader

If a loader sees that an ICD does not export the
`vk_icdNegotiateLoaderICDInterfaceVersion` function, then the loader assumes the
corresponding ICD only supports either interface version 0 or 1.

From the other side of the interface, if an ICD sees a call to
`vk_icdGetInstanceProcAddr` before a call to
`vk_icdNegotiateLoaderICDInterfaceVersion`, then it knows that loader making the calls
is a legacy loader supporting version 0 or 1.  If the loader calls
`vk_icdGetInstanceProcAddr` first, it supports at least version 1.  Otherwise,
the loader only supports version 0.


##### Loader Version 5 Interface Requirements

Version 5 of the loader/ICD interface has no changes to the actual interface.
If the loader requests interface version 5 or greater, it is simply
an indication to ICDs that the loader is now evaluating if the API Version info
passed into vkCreateInstance is a valid version for the loader.  If it is not,
the loader will catch this during vkCreateInstance and fail with a
VK_ERROR_INCOMPATIBLE_DRIVER error.

On the other hand, if version 5 or newer is not requested by the loader, then it
indicates to the ICD that the loader is ignorant of the API version being
requested.  Because of this, it falls on the ICD to validate that the API
Version is not greater than major = 1 and minor = 0.  If it is, then the ICD
should automatically fail with a VK_ERROR_INCOMPATIBLE_DRIVER error since the
loader is a 1.0 loader, and is unaware of the version.

Here is a table of the expected behaviors:

| Loader Supports I/f Version  |  ICD Supports I/f Version  |    Result        |
| :---: |:---:|------------------------|
|           <= 4               |           <= 4             | ICD must fail with `VK_ERROR_INCOMPATIBLE_DRIVER` for all vkCreateInstance calls with apiVersion set to > Vulkan 1.0 because both the loader and ICD support interface version <= 4. Otherwise, the ICD should behave as normal. |
|           <= 4               |           >= 5             | ICD must fail with `VK_ERROR_INCOMPATIBLE_DRIVER` for all vkCreateInstance calls with apiVersion set to > Vulkan 1.0 because the loader is still at interface version <= 4. Otherwise, the ICD should behave as normal.  |
|           >= 5               |           <= 4             | Loader will fail with `VK_ERROR_INCOMPATIBLE_DRIVER` if it can't handle the apiVersion.  ICD may pass for all apiVersions, but since it's interface is <= 4, it is best if it assumes it needs to do the work of rejecting anything > Vulkan 1.0 and fail with `VK_ERROR_INCOMPATIBLE_DRIVER`. Otherwise, the ICD should behave as normal.  |
|           >= 5               |           >= 5             | Loader will fail with `VK_ERROR_INCOMPATIBLE_DRIVER` if it can't handle the apiVersion, and ICDs should fail with `VK_ERROR_INCOMPATIBLE_DRIVER` **only if** they can not support the specified apiVersion. Otherwise, the ICD should behave as normal.  |

##### Loader Version 4 Interface Requirements

The major change to version 4 of the loader/ICD interface is the support of
[Unknown Physical Device Extensions](#icd-unknown-physical-device-
extensions] using the `vk_icdGetPhysicalDeviceProcAddr` function.  This
function is purely optional.  However, if an ICD supports a Physical Device
extension, it must provide a `vk_icdGetPhysicalDeviceProcAddr` function.
Otherwise, the loader will continue to treat any unknown functions as VkDevice
functions and cause invalid behavior.


##### Loader Version 3 Interface Requirements

The primary change that occurred in version 3 of the loader/ICD interface was to
allow an ICD to handle creation/destruction of their own KHR_surfaces.  Up until
this point, the loader created a surface object that was used by all ICDs.
However, some ICDs may want to provide their own surface handles.  If an ICD
chooses to enable this support, it must export support for version 3 of the
loader/ICD interface, as well as any Vulkan function that uses a KHR_surface
handle, such as:
- `vkCreateXXXSurfaceKHR` (where XXX is the platform specific identifier [i.e.
`vkCreateWin32SurfaceKHR` for Windows])
- `vkDestroySurfaceKHR`
- `vkCreateSwapchainKHR`
- `vkGetPhysicalDeviceSurfaceSupportKHR`
- `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
- `vkGetPhysicalDeviceSurfaceFormatsKHR`
- `vkGetPhysicalDeviceSurfacePresentModesKHR`

An ICD can still choose to not take advantage of this functionality by simply
not exposing the above the `vkCreateXXXSurfaceKHR` and `vkDestroySurfaceKHR`
functions.


##### Loader Version 2 Interface Requirements

Version 2 interface has requirements in three areas:
 1. ICD Vulkan entry-point discovery,
 2. `KHR_surface` related requirements in the WSI extensions,
 3. Vulkan dispatchable object creation requirements.

##### Loader Versions 0 and 1 Interface Requirements

Version 0 and 1 interfaces do not support version negotiation via
`vk_icdNegotiateLoaderICDInterfaceVersion`.  ICDs can distinguish version 0 and
version 1 interfaces as follows: if the loader calls `vk_icdGetInstanceProcAddr`
first it supports version 1; otherwise the loader only supports version 0.

Version 0 interface does not support `vk_icdGetInstanceProcAddr`.  Version 0
interface requirements for obtaining ICD Vulkan entry-points are as follows:

- The function `vkGetInstanceProcAddr` **must be exported** in the ICD library
and returns valid function pointers for all the Vulkan API entry-points.
- `vkCreateInstance` **must be exported** by the ICD library.
- `vkEnumerateInstanceExtensionProperties` **must be exported** by the ICD
library.

Additional Notes:

- The loader will filter out extensions requested in `vkCreateInstance` and
`vkCreateDevice` before calling into the ICD; Filtering will be of extensions
advertised by entities (e.g. layers) different from the ICD in question.
- The loader will not call the ICD for `vkEnumerate\*LayerProperties`() as layer
properties are obtained from the layer libraries and layer JSON files.
- If an ICD library author wants to implement a layer, it can do so by having
the appropriate layer JSON manifest file refer to the ICD library file.
- The loader will not call the ICD for
  `vkEnumerate\*ExtensionProperties` if "pLayerName" is not equal to `NULL`.
- ICDs creating new dispatchable objects via device extensions need to
initialize the created dispatchable object.  The loader has generic *trampoline*
code for unknown device extensions.  This generic *trampoline* code doesn't
initialize the dispatch table within the newly created object.  See the
[Creating New Dispatchable Objects](#creating-new-dispatchable-objects) section
for more information on how to initialize created dispatchable objects for
extensions non known by the loader.


#### Android ICD Negotiation

The Android loader uses the same protocol for initializing the dispatch
table as described above. The only difference is that the Android
loader queries layer and extension information directly from the
respective libraries and does not use the json manifest files used
by the Windows and Linux loaders.

## Table of Debug Environment Variables

The following are all the Debug Environment Variables available for use with the
Loader.  These are referenced throughout the text, but collected here for ease
of discovery.

| Environment Variable              | Behavior |  Example Format  |
|:---:|---------------------|----------------------|
| VK_ICD_FILENAMES                  | Force the loader to use the specific ICD JSON files.  The value should contain a list of delimited full path listings to ICD JSON Manifest files.  **NOTE:** If you fail to use the global path to a JSON file, you may encounter issues.  |  `export VK_ICD_FILENAMES=<folder_a>\intel.json:<folder_b>\amd.json`<br/><br/>`set VK_ICD_FILENAMES=<folder_a>\nvidia.json;<folder_b>\mesa.json` |
| VK_INSTANCE_LAYERS                | Force the loader to add the given layers to the list of Enabled layers normally passed into `vkCreateInstance`.  These layers are added first, and the loader will remove any duplicate layers that appear in both this list as well as that passed into `ppEnabledLayerNames`. | `export VK_INSTANCE_LAYERS=<layer_a>:<layer_b>`<br/><br/>`set VK_INSTANCE_LAYERS=<layer_a>;<layer_b>` |
| VK_LAYER_PATH                     | Override the loader's standard Layer library search folders and use the provided delimited folders to search for layer Manifest files. | `export VK_LAYER_PATH=<path_a>:<path_b>`<br/><br/>`set VK_LAYER_PATH=<path_a>;<pathb>` |
| VK_LOADER_DISABLE_INST_EXT_FILTER | Disable the filtering out of instance extensions that the loader doesn't know about.  This will allow applications to enable instance extensions exposed by ICDs but that the loader has no support for.  **NOTE:** This may cause the loader or applciation to crash. |  `export VK_LOADER_DISABLE_INST_EXT_FILTER=1`<br/><br/>`set VK_LOADER_DISABLE_INST_EXT_FILTER=1` |
| VK_LOADER_DEBUG                   | Enable loader debug messages.  Options are:<br/>- error (only errors)<br/>- warn (warnings and errors)<br/>- info (info, warning, and errors)<br/> - debug (debug + all before) <br/> -all (report out all messages) | `export VK_LOADER_DEBUG=all`<br/><br/>`set VK_LOADER_DEBUG=warn` |
 
## Glossary of Terms

| Field Name | Field Value |
|:---:|--------------------|
| Android Loader | The loader designed to work primarily for the Android OS.  This is generated from a different code-base than the desktop loader.  But, in all important aspects, should be functionally equivalent. |
| Desktop Loader | The loader designed to work on both Windows and Linux.  This is generated from a different [code-base](#https://github.com/KhronosGroup/Vulkan-LoaderAndValidationLayers) than the Android loader.  But in all important aspects, should be functionally equivalent. |
| Core Function | A function that is already part of the Vulkan core specification and not an extension.  For example, vkCreateDevice(). |
| Device Call Chain | The call chain of functions followed for device functions.  This call chain for a device function is usually as follows: first the application calls into a loader trampoline, then the loader trampoline calls enabled layers, the final layer calls into the ICD specific to the device.  See the [Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains) section for more information |
| Device Function | A Device function is any Vulkan function which takes a `VkDevice`, `VkQueue`, `VkCommandBuffer`, or any child of these, as its first parameter.  Some Vulkan Device functions are: `vkQueueSubmit`, `vkBeginCommandBuffer`, `vkCreateEvent`.  See the [Instance Versus Device](#instance-versus-device) section for more information. |
| Discovery | The process of the loader searching for ICD and Layer files to setup the internal list of Vulkan objects available.  On Windows/Linux, the discovery process typically focuses on searching for Manifest files.  While on Android, the process focuses on searching for library files. |
| Dispatch Table | An array of function pointers (including core and possibly extension functions) used to step to the next entity in a call chain.  The entity could be the loader, a layer or an ICD.  See [Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains) for more information.  |
| Extension | A concept of Vulkan used to expand the core Vulkan functionality.  Extensions may be IHV-specific, platform-specific, or more broadly available.  You should always query if an extension exists, and enable it during `vkCreateInstance` (if it is an instance extension) or during `vkCreateDevice` (if it is a device extension). |
| ICD | Acronym for Installable Client Driver.  These are drivers that are provided by IHVs to interact with the hardware they provide.  See [Installable Client Drivers](#installable-client-drivers) section for more information.
| IHV | Acronym for an Independent Hardware Vendor.  Typically the company that built the underlying hardware technology you are trying to use.  A typical examples for a Graphics IHV are: AMD, ARM, Imagination, Intel, Nvidia, Qualcomm, etc. |
| Instance Call Chain | The call chain of functions followed for instance functions.  This call chain for an instance function is usually as follows: first the application calls into a loader trampoline, then the loader trampoline calls enabled layers, the final layer calls a loader terminator, and the loader terminator calls all available ICDs.  See the [Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains) section for more information |
| Instance Function | An Instance function is any Vulkan function which takes as its first parameter either a `VkInstance` or a `VkPhysicalDevice` or nothing at all.  Some Vulkan Instance functions are: `vkEnumerateInstanceExtensionProperties`, `vkEnumeratePhysicalDevices`, `vkCreateInstance`, `vkDestroyInstance`.  See the [Instance Versus Device](#instance-versus-device) section for more information. |
| Layer | Layers are optional components that augment the Vulkan system.  They can intercept, evaluate, and modify existing Vulkan functions on their way from the application down to the hardware.  See the [Layers](#layers) section for more information. |
| Loader | The middle-ware program which acts as the mediator between Vulkan applications, Vulkan layers and Vulkan drivers.  See [The Loader](#the loader) section for more information. |
| Manifest Files | Data files in JSON format used by the desktop loader.  These files contain specific information for either a [Layer](#layer-manifest-file-format) or an [ICD](#icd-manifest-file-format).
| Terminator Function | The last function in the instance call chain above the ICDs and owned by the loader.  This function is required in the instance call chain because all instance functionality must be communicated to all ICDs capable of receiving the call.  See [Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains) for more information. |
| Trampoline Function | The first function in an instance or device call chain owned by the loader which handles the setup and proper call chain walk using the appropriate dispatch table.  On device functions (in the device call chain) this function can actually be skipped.  See [Dispatch Tables and Call Chains](#dispatch-tables-and-call-chains) for more information. |
| WSI Extension | Acronym for Windowing System Integration.  A Vulkan extension targeting a particular Windowing and designed to interface between the Windowing system and Vulkan. See [WSI Extensions](#wsi-extensions) for more information. |
