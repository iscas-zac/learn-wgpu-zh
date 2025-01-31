# 使用 Snapdragon Profiler 调试 wgpu 程序

在[与 Android App 集成](../android/)章节我们已经学习了 wgpu 与 Android App 的集成，现在来看看集成后的调试。

## Snapdragon Profiler 工具介绍
[Snapdragon Profiler](https://developer.qualcomm.com/software/snapdragon-profiler) 是高通公司开发的一款可运行在 Windows、Mac 和 Linux 平台上的性能分析和帧调试工具。 它通过 USB 与**安卓设备**连接，允许开发人员分析 CPU、GPU、内存等数据，以便我们发现并修复性能瓶颈。

Snapdragon Profiler 工具的功能特点：
- 实时监测 GPU 性能;
- 查看 CPU 调度和 GPU 阶段数据，了解应用程序将时间花在哪里;
- GPU **帧**捕获;
- 单步调试**帧**绘制;
- 查看和编辑**着色器**并在设备上预览结果;
- 查看和调试像素历史记录;
- 捕获和查看每次绘制调用的 GPU 指标;

上面的官网链接提供了对应平台安装包的免费下载。如果是 Mac 和 Linux 平台, 在安装 Snapdragon Profiler 之前需要先安装 [momo 框架](http://www.mono-project.com/download/)（mono 是 Windows .Net 框架的跨平台开源实现）。
在运行 Snapdragon Profiler 之前需要确保系统上安装了 Android Studio 或者 AndroidSDK，并且已将 **ADB** 路径添加到系统环境变量中。

## 实时模式查看 GPU 统计数据
USB 连接要调试的 Android 手机后打开 Snapdragon Profiler，点击窗口左边栏的 **Start a Session**, 此时右边出现的小弹窗里会列出当前与电脑连接的所有可调试设备，我们选中列表中的设备，勾选上弹窗左下角的 **Auto connect** 再点击右下角的 **Connect**，这样，下回再次调试同一台设备时就能自动连接到 Snapdragon Profiler 了：

<img src="./connect.jpg" />

连接后，有四种调试模式供我们选择：实时、追踪、帧捕获及 CPU 采样，现在选择**实时**（左图），在**实时**窗口的左边栏展示了实时指标列表，我们可以选择相应的指标项来收集 CPU、GPU、内存、网络、电源和散热的实时指标（右图）：

<div style="display: flex;">
    <div>
        <img src="./realtime-left.jpg" alt="实时模式" />
    </div>
    <div style="width: 20px;"></div>
    <div>
        <img src="./realtime.jpg" alt="实时预览" />
    </div>
</div>

上面的右图中，我选择了 **GPU General**、**GPU Stalls** 两个指标类别，窗口右边展示了每个细分指标的实时数据图表，要添加新的指标图表，只需双击类别（以添加类别中的所有指标）或单个指标，或者将类别或指标拖放到右侧的“图表”窗格中。

## 追踪模式检查片上内存装载

**片上内存**（on-chip memory）装载是影响移动应用中 GPU 性能的常见问题之一。在本节中，我们来学习如何使用 Snapdragon Profiler 查找和定位引起片上内存装载的应用程序代码。

<div class="note">

Snapdragon Profiler 里将**片上内存**称之为**图形内存**（GMEM，全称 Graphic Memory），但是这里的图形内存跟显存容易混淆，它俩并不是一回事。故，下边统一使用**片上内存**来指代 GMEM。

</div>

### 什么是片上内存装载？

移动 GPU 的 **Tiling** 架构管线包括一个渲染通道。在渲染过程中，每个 **Tile** 都是先被渲染到**片上内存**中。按照驱动程序的默认行为，先前的**帧缓冲区**数据被从设备内存加载到每个 Tile 的片上内存中，即发生片上内存装载。

<div class="note">

所谓 Tiling，本质上就是管理 GPU 内存的技术。Tiling 利用**片上内存**（on-chip memory）去降低**设备内存**的访问次数，从而降低 GPU 内存带宽的消耗及访问延迟。 
正确理解并利用 Tiling 架构的内存管理特性，可以有效的提高 GPU 程序的性能。

</div>

### 为什么要尽可能地减少或避免片上内存装载？

因为每一次**片上内存**的加载都会减慢 GPU 的处理速度。<br />
如果在 `begin_render_pass` 时通过设置 `Clear()` 来清理片上内存，驱动程序就可以避免在片上内存中装载**帧缓冲区**数据。虽然这涉及到一个额外的图形指令调用及其相关的开销，但它比为每个正在渲染的 **Tile** 将帧缓冲区数据加载回片上内存的开销要低得多。

导致**片上内存**装载的最主要原因是: 对驱动程序的不恰当提示。
应用程序代码使驱动程序认为需要**帧缓冲区**的先前内容。


### 检测片上内存装载
在 Snapdragon Profiler 的**追踪模式**下，我们可以让**渲染阶段**（Rendering Stages） 指标突出显示其自身通道中的**片上内存装载**（GMEM Loads）。

<div class="note">

GPU 应用必须在项目的 AndroidManifest.xml 文件中包含 `INTERNET` 权限以启用图形 API 及 GPU 指标的追踪：

```toml
<uses-permission android:name="android.permission.INTERNET" />
```

另外，Snapdragon Profiler 的追踪模式不允许追踪捕获超过 10 秒。也就是说，从点击 `Start Capture` 开始到点击 `Stop Capture` 结束，时长不得超过 10 秒。

</div>

启用**追踪模式**的操作步骤：
- 连接好 Android 设备后，从 `Start Page` 界面单击左边栏的 `System Trace Analysis`，此时，就创建了一个新的 `Trace` 选项卡。
- 选择刚创建的 `Trace` 选项卡，进入一个类似于**实时**模式的视图，然后在 `Data Sources` 边栏上端的应用列表中选中要追踪的应用（如果列表中找不到，就通过列表右上角的 `Launch` 按钮去启动要追踪的应用）。
- 在 `Data Sources` 边栏下端，选中 `Process` -> `Vulkan` -> `Rendering Stages` 项。

<img src="./trace.png" style="max-width: 425px;"/>

点击 `Start Capture` 开始追踪，在 10 秒内的任意段点击 `Stop Capture`，在等待 N 秒（取决于电脑性能）后就会展示出如下图表：

<img src="./GMEM_load.jpg" />

上图渲染阶段的**设置**对话框显示，这些**片上内存装载**消耗了总渲染时间的 23% 左右。

我们来看看源码帧渲染中的[这条 begin_render_pass() 命令](https://github.com/jinleili/wgpu-in-app/blob/88e53957f7c80dbd8e75273c9ff48ecab958984f/src/examples/cube.rs#L356-L363)，颜色附件的片上操作使用了 Load：
```rust
ops: wgpu::Operations {
    // load: wgpu::LoadOp::Clear(wgpu::Color::BLACK),
    load: wgpu::LoadOp::Load,
    store: true,
},                    
```
但此处实际上没有装载之前的帧缓冲区数据的必要，我们改为使用 `Clear()` 改善性能之后，就回收了之前片上内存装载消耗的性能，下图可以看到 `GMEM Load` 统计项消失了（没有发生片上内存装载时就不会显示）：

<img src="./GMEM_store.jpg" />

## 帧捕获模式
**帧捕获模式**允许捕获 GPU 应用程序的单一帧, 可以详细显示一个场景在 GPU 上的渲染情况。

启用帧捕获模式的操作与追踪模式几乎一样，唯一不同之处就是帧捕获模式在点击 `Take Snapshot` 捕获一帧数据后会自动结束捕获：

<img src="./frame.jpg" />

左侧<span style="color: red;">红框</span>区域是当前帧的着色器代码，它们是由 WGSL 自动转换而来的 SPIR-V 代码（当然，此处的着色器代码还取决于 GPU 应用所使用的图形后端，我使用的是 Vulkan 后端，如果使用 OpenGL 后端，此处就会显示 GLSL 代码）。红框下方的区域可以显示着色器的错误信息。说到这里就不得不提 WebGPU 的 WGSL 着色器语言的优势了：WGSL 在编译阶段时就得到了很好的验证，运行时的验证更是能方便地指出着色器与管线不一致的地方。所以，我们不需要依赖 Snapdragon Profiler 的着色器调试功能。

中间<span style="color: green;">绿框</span>区域是**命令队列**（Queue）提交给当前帧的所有 Vulkan 命令。选中某一条命令，右侧资源面板将展示出此命令涉及的所有资源：图形|计算管线，纹理，着色器等等。

右侧<span style="color: blue;">蓝框</span>区域是资源面板。选中某一项资源，下方的面板将能展示出资源详情。<br />
比如，选择纹理资源后，下方的 `Image Preview` 选项卡会展示可缩放的大图预览，鼠标在纹理图片上滑动可显示对应像素的 RGB 色值，`Inspector` 选项卡会展示纹理的格式及层次细节参数等（左图）; 选择布局描述符资源后，`Inspector` 选项卡会展示出**绑定组布局描述符**（BindGroupLayoutDescriptor）详情（右图）：

<div style="display: flex;">
    <div>
        <img src="./resource-left.jpg" />
    </div>
    <div style="width: 20px;"></div>
    <div>
        <img src="./resource-right.jpg" />
    </div>
</div>