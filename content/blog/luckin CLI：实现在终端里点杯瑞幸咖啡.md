+++
title = "luckin CLI：实现在终端里点杯瑞幸咖啡"
date = "2026-06-18T11:28:59+08:00"
tags = ["CLI"]
+++

上周末加班，今天调休，相当于提前放端午假了哈哈。

瑞幸咖啡太能整活了，听说出了瑞幸咖啡 CLI 服务，可以在终端调用 CLI 服务，让下单更加便捷。刚好前几天女朋友跟我说瑞幸出了一个新品，“弗朗明戈-跳跳冰茶”，说里面有跳跳躺，喝起来很神奇。我们今天就来尝尝看，顺便看看这个 CLI 是整活还是真能用！

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T05-07-49-696Z.png" width=200/>


首先，访问[瑞幸咖啡AI开放平台](https://open.lkcoffee.com/cli)，可以看到网站做的还是很简洁科技风的。

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-31-46-639Z.png" width=500/>

我们按照官网教程，安装 CLI 工具：

```bash
curl -fsSL https://open.lkcoffee.com/install | bash
```

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-36-16-259Z.png" width=500/>

我们可以看到已经成功安装了 luckin 0.0.1 CLI 工具。接下来我们在终端输入 luckin 回车：

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-38-18-773Z.png" width=500/>

需要先登录授权一下，这里可以看到是跳转到浏览器官网进行授权登录，然后创建一个 Token。

<figure>
<img src="https://cdn.kmoon.fun/2026/2026-06-18T03-39-51-212Z.png" width=300/>
<img src="https://cdn.kmoon.fun/2026/2026-06-18T03-40-18-960Z.png" width=300/>
</figure>

然后我们返回终端，可以看到已经登录成功了，并且可以看到支持哪些斜杠命令：

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-41-02-898Z.png" width=500/>

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-47-39-742Z.png" width=500/>

我们先查看一下附近门店，使用 /store 命令，这里突然离谱了起来，这个命令居然需要我自己输入经纬度？？？

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-49-24-902Z.png" width=500/>

算了，我们还是在 Claude Code 里面使用它的 my-coffee Skill 吧。希望后续可以打通定位服务，自己定位用户所在位置。

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T03-54-20-618Z.png" width=500/>

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T04-38-06-550Z.png" width=500/>

我们将官网创建的 Token 手动复制过来，并告诉它在附近帮我点一杯新品冰茶：

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T04-41-08-607Z.png" width=500/>

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T04-54-50-192Z.png" width=500/>

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T04-56-28-201Z.png" width=500/>

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T04-57-40-940Z.png" width=500/>

这里需要我们扫码支付一下，支付完成后会帮我们查询订单状态和取餐码。这里我们可以发现目前只支持自取，暂时不支持外卖配送。不过我相信不久的将来，当美团 CLI、淘宝闪购 CLI 出现的时候，是完全可以支持外卖配送的。

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T04-59-52-774Z.png" width=500/>

好啦，现在我们可以准备去门店取餐了。

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T05-03-13-746Z.png" width=500/>

打开手机的小程序，我们可以看到订单制作完成，等待取餐中：

<img src= "https://cdn.kmoon.fun/2026/2026-06-18T05-06-18-362Z.png" width=300/>

也是成功喝到新品了，一直滋滋滋，跟小时候吃的跳跳糖一样。

<img src="https://cdn.kmoon.fun/2026/2026-06-18T05-24-51-962Z.png" alt="" width=300/>

## 写在最后

这次折腾这么久，最后也确实成功完成了在终端里下单一杯咖啡。瑞幸这次不是简单整活，而是提供了完整的一套 MCP + CLI + Skills，同时也提供了工具文档，这体现了瑞幸选择拥抱 AI 这条路上迈出了第一步。