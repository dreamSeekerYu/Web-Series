# iframe

iframe 能够帮助我们嵌入更为丰富的视图内容，譬如 VSCode 这样的 IDE 也是典型的微前端框架，他使用了 Electron 作为底层，并且使用 webview 标签作为视图的容器。而在浏览器中我们往往使用 iframe 来加载不同域的内容。

iFrame 可以创建一个全新的独立的宿主环境，iFrame 的页面和父页面是分开的，作为独立区域而不受父页面的 CSS 或者全局的 JavaScript 影响。iFrame 的不足或缺陷也非常明显，其会进行资源的重复加载，占用额外的内存；其会阻塞主页面的 onload 事件，和主页面共享连接池，而浏览器对相同域的连接有限制，所以会影响页面的并行加载。

iFrame 的改造门槛较低，但是从功能需求的角度看，其无法提供 SEO，并且需要我们自定义应用管理与应用通讯机制。iFrame 的应用管理不仅要关注其加载与生命周期，还需要考虑到浏览器缩放等场景下的界面重适配问题，以提供用户一致的交互体验；这里我们再简要讨论下同源场景中的跨界面通讯解决方案。

> 📖 详细解读参阅 [DOM CheatSheet](https://parg.co/YlB)

- BroadcastChannel

BroadcastChannel 能够用于同源不同页面之间完成通信的功能。它与 window.postMessage 的区别就是，BroadcastChannel 只能用于同源的页面之间进行通信，而 window.postMessage 却可以用于任何的页面之间；BroadcastChannel 可以认为是 window.postMessage 的一个实例，它承担了 window.postMessage 的一个方面的功能。

```js
const channel = new BroadcastChannel("channel-name");

channel.postMessage("some message");
channel.postMessage({ key: "value" });

channel.onmessage = function (e) {
  const message = e.data;
};

channel.close();
```

- SharedWorker API

Shared Worker 类似于 Web Workers，不过其会被来自同源的不同浏览上下文间共享，因此也可以用作消息的中转站。

```js
// main.js
const worker = new SharedWorker("shared-worker.js");

worker.port.postMessage("some message");

worker.port.onmessage = function (e) {
  const message = e.data;
};

// shared-worker.js
const connections = [];

onconnect = function (e) {
  const port = e.ports[0];
  connections.push(port);
};

onmessage = function (e) {
  connections.forEach(function (connection) {
    if (connection !== port) {
      connection.postMessage(e.data);
    }
  });
};
```

- Local Storage

localStorage 是常见的持久化同源存储机制，其会在内容变化时触发事件，也就可以用作同源界面的数据通信。

```js
localStorage.setItem("key", "value");

window.onstorage = function (e) {
  const message = e.newValue; // previous value at e.oldValue
};
```

# 线程独立性问题

在 Electron 中的，webview 天然地和外部的 browserWindow 拥有分开的线程和 js 上下文，甚至连 devtools 都要单独启用。因为它其实是 Chromium 中的 Out-of-Process iframes 的实现（经过了 Electron 方面的封装）。对应的概念就是 Chrome App 中的 Webview 组件。

iframe 作为升级版的 frame，一般来说都会被认为和上层的 parent 容器处在同一个进程中, 因为基于 html 的 spec，他们会拥有父容器的一个孩子 BrowserContext。在这种情况下，iframe 当中的 js 运行时便会阻塞外部的 js 运行，特别是当如果 iframe 中的代码质量不高而导致性能问题时，外层运行的容器会受到相当大的影响。这显然是我们不愿意看到的，因为 webview 中的内容仅仅会作为 IDE 拓展机制的一部分，我们不希望看到我们的外部 UI 和程序被 iframe 阻塞从而导致性能表现不佳。

![iframe 线程](https://s2.ax1x.com/2019/09/11/nwcdzR.png)

幸运的是，Chromium 在 67 版本之后默认开启了 Site Isolation。基于它的描述，如果 iframe 中的域名和当前父域名不同（也就是大家熟悉的跨域情况），那么这个 iframe 中的渲染内容就会被放在两个不同的渲染进程中。而这就给我们解决线程独立性的问题带来了曙光。只需要将 IDE 主应用的页面挂在 a.com 的域名，而同时将 iframe 的的页面挂在另外一个域名下，那么这个 iframe 的进程就和主进程分开了。在这种模型下，iframe 和主进程仅仅能通过 postMessage 和 message 事件进行数据通讯。但是在上面的模型中，仍然有一点需要注意。基于 Site Isolation 的特性，同一个页面中如果有多个，拥有同一个域名的多个 iframe 之间是共享进程的，因此他们仍然会互相卡顿。如果某个业务场景需要一个更为独立的 iframe 进程，它必须和其他 iframe 拥有不同的域名。

# iframe 生命周期

## 持久化

对于需要持久的 iframe 元素，我们始终将它挂载在一个 body 根节点下的固定区域中。同时，为其指定一个观察目标，使用 MutationObserver 和 ResizeObserver(Chrome 61 之后支持)对这个目标进行观察，以便能即使知道这个目标在可视区域中所处的位置。最后，根据计算出的位置，将这个 iframe 盖在目标区域上，从而看起来就好像一直嵌在目标中一样。

![iframe 持久化](https://s2.ax1x.com/2019/09/11/nwcoef.png)
