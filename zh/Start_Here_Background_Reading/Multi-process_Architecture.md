# 多进程架构

这个文档描述了 Chromium 的高层架构

## 问题

构建一个从不会挂起或崩溃的渲染引擎几乎是不可能的。构建一个完全安全的渲染引擎也是几乎不可能的。

在某种程度上，web浏览器当前状态就像一个与过去的多任务操作系统合作的单独的用户。正如在一个这样的操作系统中的错误程序会让整个系统挂掉，所以一个错误的web页面也可以让一个现代浏览器挂掉。仅仅需要一个浏览器或插件的 bug，就饿能让整个浏览器和所有正在运行的标签页停止运行。

现代操作系统更加健壮，因为他们把应用程序分成了彼此隔离的独立进程。一个程序中的 crash 通常不会影响其他程序或整个操作系统，每个用户对用户数据的访问也是有限制的。

## 架构概览

我们为浏览器的标签页使用独立的进程，以此保护整个应用程序免受渲染引擎中的 bug 和故障的伤害。我们也会限制每个渲染引擎进程的相互访问，以及他们与系统其他部分的访问。某些程度上，这为web浏览提供了内存保护，为操作系统提供了访问控制。

我们把运行UI 和管理 Tab/Plugin 的主进程称为 "browser process" 或 "browser"。同样地，标签页相关的进程被称作 "render processes" 或 "renderers"。renderers 使用开源渲染引擎的 [Blink](https://www.chromium.org/blink) 来解释和渲染 HTML 的布局。

![img](../arch.png)

### 管理渲染进程

每个 render process 有一个全局的 `RenderProcess` 对象，用来管理与父 browser process 之间的通信和维护全局的状态。browser 为每个 render process 维护一个对应的 `RenderProcessHost`，用来管理 browser 状态，并与 renderer 通信。browser and renderer 使用 [Chromium's IPC system](../General_Architecture/Inter-process_Communication.md) 进行通信。

### 管理 view

每个 render process 有一个以上的 `RenderView` 对象，由 `RenderProcess` 管理（它与标签页的内容相关）。对应的 `RenderProcessHost` 维护一个与 renderer 中每个 view 相关的 `RenderViewHost`。每个 view 被赋予一个 view ID,以区分同一个 renderer 中的不同 view。这些 ID 在每个 renderer 内是唯一的，但在 browser 中不是，所以区分一个 view 需要一个 `RenderProcessHost` 和一个 view ID。

browser 与一个包含内容的特定标签页之间的通信是通过这些 `RenderViewHost` 对象来完成的，`RenderViewHost` 知道如何通过它们的 `RenderProcessHost` 向 `RenderProcess` 和 `RenderView` 发送消息。

译者注：

## 组件与接口

在 render process 中：

- `RenderProcess` 处理与 browser 对应的 `RenderProcessHost` 的 `IPC` 通信。每个 render process 就有唯一的一个 `RenderProcess` 对象。这就是所有 browser、renderer 之间的通信发生的方式。

- `RenderView` 对象与它在 browser process 中对应的 `RenderViewHost`（通过 `RenderProcess`）和我们的 webkit 嵌入层通信。这个对象代表了一个网页在标签页或一个弹出窗口的内容。

在 browser process 中:

- `Browser` 对象代表了 a top-level browser window
- `RenderProcessHost` 对象代表了浏览器中单个 browser 与 renderer  之间的 `IPC` 连接（the browser side of a single browser ↔ renderer IPC connection）。在 browser process 中，对于每个 render process 来说，在 browser process 中仅有一个 `RenderProcessHost` 与之对应。
- `RenderViewHost` 对象封装了与远端 `RenderView` 的通信，`RenderWidgetHost` 处理输入并在 browser 中为 `RenderWidget` 进行绘制。

想要得到更多关于这种嵌入是如何工作的详细信息，可以查看 [How Chromium displays web pages design document](How_Chromium_displays_web_pages_design_document)。

## 共享 render process

通常，每个新的 window 或标签页是在一个新进程里打开的。browser 会生成一个新的进程，然后指导它去创建一个 `RenderView`。

有时候，有这样一种必要或欲望在标签页或窗口间共享 render process 。一个 web 应用程序会在期望同步交流时，打开一个新的 window，比如，在 JavaScript 里使用 `window.open`。这种情况下，当我们创建一个新的 window 或标签页时，我们需要重用打开这个 window 的进程。我们也有一些策略来把新的标签页分配给已有的进程（如果进程总数太大，或者用户已经有一个打开到该域的进程）。这些策略在 [Process Models](../General_Architecture/Process_Models.md) 里也有阐述。


## 检测 crashed 或者异常的 renderers

browser process 的每个 `IPC` 连接会监听进程句柄。如果这些句柄是 signaled（有信号的），那么 render process 已经挂了，标签页会得到一个通知。从这时开始，我们会展示一个 "sad tab" 画面来通知用户 renderer 已经挂掉了。这个页面可以按刷新按钮或者通过打开一个新的导航来重新加载。这时，我们会注意到没有对应的进程，然后创建一个新的。

## renderer 中的沙箱

给定的 renderer 是运行在独立的进程中的，所以我们有机会通过沙箱（[sandboxing](https://www.chromium.org/developers/design-documents/sandbox)）限制它对系统资源的访问。例如，我们可以确保 renderer 唯一的网络权限是通过它的 parent browser process 实现。相似的，我们可以通过使用 host 操作系统内置的权限来限制它对文件系统的访问权限。

除了限制 renderer 对文件系统和网络的访问权限，我们也可以限制它对用户的显示器以及相关的东西的一些权限。我们在独立的 Windows "[Desktop](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682573(v=vs.85).aspx)"（对用户不可见）中运行每个 render process。这避免了让 renderer 在新的标签页或捕捉按键之间妥协。

## 归还内存

让 renderers 运行在独立的进程中，赋予隐藏的标签页更低的优先级会更加直接。通常，Windows 平台上的最小化的进程会把它们的内存自动放到一个“可用内存”池里。在低内存的情况下，Windows 会在交换这部分内存到更高优先级内存前，把它们交换到磁盘，以保证用户可见的程序更易响应。我们可以对隐藏的标签页使用相同的策略。当 render process 没有顶层标签页时，我们可以释放进程的“工作集”空间，作为一个给系统的信号，让它如果必要的话，优先把这些内存交换到磁盘。因为我们发现，当用户在两个标签页间切换时，减少工作集大小也会减少标签页切换性能，所以我们是逐渐释放这部分内存的。这意味着如果用户切换回最近使用的标签页，这个标签页的内存比最近较少访问的标签页更可能被换入。有着足够内存的用户运行他们所有的程序时根本不会注意到这个进程：事实上 Windows 只会在需要的时候重新声明这块数据，所以在有充分内存时，不会有性能瓶颈。

这能帮助我们在低内存的情况下得到最佳的内存轨迹。几乎不被使用的后台标签页相关的内存可以被完全交换掉，前台标签页的数据可以被完全加载进内存。相反的，一个 single-process browser 会在它的内存里随机分配所有标签页的数据，并且不可能如此清晰地隔离已使用的和未使用的数据，导致了内存和性能上的浪费。

## 插件和扩展

Firefox 风格的 NPAPI 插件运行在他们自己的进程里，与 renderer 隔离。这会在 [Plugin Architecture](../General_Architecture/Plugin_Architecture.md) 中描述。

[Site Isolation](https://www.chromium.org/developers/design-documents/site-isolation) 项目的旨在提供更多的 renderers 之间的隔离，该项目的早期交付包括在隔离的进程中运行 Chrome 的 HTML / JavaScript 内容扩展。

## 如何添加新特性(不用扩充 RenderView / RenderViewHost / WebContents)

### 问题

过去，新的特性（比如，自动填充选取样例）可以通过把新特性的代码导入到 `RenderView` 类（在 renderer process 里）和 RenderViewHost 类（在 browser process 里）。如果一个新的特性是在 browser process 的 IO 线程里处理的，那么它的 `IPC` 消息由 `BrowserMessageFilter` 调度。`RenderViewHost` 仅仅会通过调用 `WebContent`（通过 `RenderViewHostDelegate` 接口） 分发 `IPC` 消息，这会调用另一块代码。所有的 browser 与 renderer 之间的 `IPC` 消息会被声明在一个巨大的 `render_messages_internal.h` 里，为每个新特性修改所有的这些文件意味着这些类会变得臃肿。


### 解决方案

我们增加了 `helper` 类和对上面的每个线程 `IPC` 消息的过滤的机制。这使得编写自包含的特性更加容易。

#### Renderer 端

如果你想要过滤和发送 `IPC` 消息，就需要实现 `RenderViewObserver` 接口( `content/public/renderer/render_view_observer.h` )。`RenderViewObserver` 基类持有一个 `RenderView` 类，管理对象的生命周期，使其绑定到 `RenderView`（它是可重写的）。这个类就可以过滤和发送 `IPC` 消息，此外还可以获得许多特性需要的关于页面加载与关闭的通知。作为一个例子，可以查看 `ExtensionHelper` ( `extensions/extension_helper.h` )。

如果你的特性有一部分代码是在 `WebKit` 内的，避免通过 `WebViewClient` 接口回调，这样我们就不会使得 `WebViewClient` 变得庞大。考虑创建新的 `WebKit` 接口给 `WebKit` 代码调用，让 renderer 端的类去实现它。作为一个例子，查看 `WebAutoFillClient` (`third_party/WebKit/public/WebAutoFillClient.h`)。

### Browser UI thread

`WebContentsObserver` ( `content/public/browser/web_contents_observer.h` ) 接口允许 UI 线程的对象过滤 `IPC` 消息，以及给出关于页面导航的通知。作为一个例子：请看 `TabHelper` (`chrome/browser/extensions/tab_helper.h`)。

### 其他的 browser threads

为了过滤和发送 `IPC` 消息给其他的 browser threads，比如 IO / FILE 等等，实现 `BrowserMessageFilter` 接口(`content/public/browser/browser_message_filter.h`)。`RenderProcessHostImpl` 对象在它的 `CreateMessageFilters` 函数里创造和增加过滤器。

通常，如果一个特性有许多 `IPC` 消息，这些消息应该移动到一个独立的文件（例如，不要加到 `render_messages_internal.h` 里）。这对过滤消息的线程（除了 IO 线程）也有帮助。作为一个例子，请看 `content/common/database_messages.h` 。这允许他们的过滤器 `DatabaseMessageFilter` 方便的发送文件到 file 线程，而不用在很多位置指定它们的 ID。

```C++
void DatabaseMessageFilter::OverrideThreadForMessage(
    const IPC::Message& message,
    BrowserThread::ID* thread) {
  if (IPC_MESSAGE_CLASS(message) == DatabaseMsgStart)
    *thread = BrowserThread::FILE;
}
```
