+++
title = "zhlgd-cli: 尝试做一个 CLI"
date = "2026-06-07T17:01:02+08:00"
description = "记录我从 0 开始为智慧理工大尝试做一个 Node.js CLI 的过程，包括项目初始化、commander、登录鉴权，以及 msg / news 两个子命令。"
tags = ["CLI"]
+++

一直觉得在命令行里敲命令然后回车，有一种莫名的爽感，特别是当黑框里返回自己想要的东西的时候。

AI 这两年发展很快，CLI（Command Line Interface）也被重新推到了舞台中央。平时我们常用的 GUI，对人类当然很友好，但对 Agent 来说未必是最高效的工作方式。能直接发命令、拿结果、继续串联下一步，这种交互反而更适合 AI 干活。

目前互联网上最火的 CLI 之一，应该就是字节跳动开源的 [飞书 CLI（lark-cli）](https://github.com/larksuite/cli/blob/main/README.zh.md)。它能让人类和 Agent 都在终端里操作飞书，比如收发消息、创建文档、多维表格、查询日历和会议等等。

CLI 对 Agent 来说，本质上也是一种工具调用，而且很多时候还是更顺手的一种工具。所以，为了紧跟 AI 发展，我决定尝试自己写一个 CLI，看看这东西到底是怎么落地的。

## 先想需求

不管干什么，总得结合实际需求，毕竟没实际用途，做着也没劲。仔细想了想，我日常能接触到的场景也就来自两方面：学校、公司。公司这边肯定没法下手，压根不需要我；字节这边每天都会有层出不穷的工具，各种花色任你选择，说实话真有点“军备竞赛”那味道了。

所以我把思路转移到学校这边。学校这边能做的感觉挺多，毕竟学校的网站、各种平台都比较老旧，有些交互也确实不够顺手。突然想到智慧理工大，武理一个大家日常都会用到的大 OA 平台，而且最近还刚改版，那就干脆拿它来练手，试试把它 CLI 化。

## 开发流程

第一步肯定是先了解一下 CLI 的开发流程。我现在除了知道应该要用 Node.js 开发，其他几乎是零基础。那就老老实实先上网搜。

![搜索 Node.js CLI](https://cdn.kmoon.fun/2026/2026-06-08T12-32-53-351Z.png)

找到了一篇 2019 年的知乎文章：[用Node.js开发一个Command Line Interface (CLI)](https://zhuanlan.zhihu.com/p/38730825)。本来我看到这个时间，第一反应是打算划走，但换个思路一想，这么久远的文章还能被我搜到，说不定反而是一篇优质内容，而且标题也非常直白，所以我还是决定仔细看一看。

看完之后，大概就知道整个流程是什么样了。和我预想得差不多，确实是用 Node.js 开发，难怪平时网上安装 CLI 的时候都会让我先确保本地有 Node 环境，而且很多工具都是直接用 `npx` 起。

开发流程大概可以分成下面几步：

1. 创建 npm 模块
2. 增加 bin 入口文件
3. 全局安装
4. 发布

这么一看，整个路径其实还挺清晰的。先把最小可用版本做出来，再往上加子命令就行。

## 环境配置

先简单配置一下 [Node.js](https://nodejs.org/zh-cn) 环境。我平时一般用 [nvm](https://www.nvmnode.com/) 管理 Node 版本，毕竟我之前真的因为版本差异排查过很久 bug，这种坑踩过一次就够了。

Node 环境配好之后先检查一下：

![本地 Node 环境检查结果](https://cdn.kmoon.fun/2026/2026-06-08T11-45-40-376Z.png)

创建 npm 模块，直接执行：`npm init`

![使用 npm init 初始化项目](https://cdn.kmoon.fun/2026/2026-06-08T12-36-11-738Z.png)

这样就初始化好了整个项目：

```json
{
  "name": "zhlgd-cli",
  "version": "1.0.0",
  "description": "武汉理工大学智慧理工大命令行工具",
  "homepage": "https://github.com/kmoonn/zhlgd-cli#readme",
  "bugs": {
    "url": "https://github.com/kmoonn/zhlgd-cli/issues"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/kmoonn/zhlgd-cli.git"
  },
  "license": "MIT",
  "author": "Kmoon",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}
```

> - `type` 这里选择 `module`，因为新版 ES 模块化用的是 `import/export`，现在做 CLI 基本都会用到。
> - `commonjs` 则是老版 Node.js 模块化，写法是 `require()`。

接着在 `package.json` 里增加 `bin` 字段：

```json
{
  "bin": {
    "zhlgd": "./bin/index.js"
  }
}
```

这里也顺便记一下：

- `zhlgd-cli` 是项目名 / 包名
- `zhlgd` 才是最终在终端里执行的命令

在项目根目录新建一个 `bin` 文件夹，并在里面新建一个 `index.js` 文件。

文件结构长这样：

```
zhlgd-cli/
├── bin/
│   └── index.js
└── package.json
```

编写 `bin/index.js` 代码。

第一行必须写：`#!/usr/bin/env node`，意思是告诉系统用 Node 来运行这个脚本。

```js
#!/usr/bin/env node

// 你的命令行逻辑开始
console.log('✅ 智慧理工大命令行工具启动成功！');
console.log('👉 命令：zhlgd');
```

然后全局安装，把命令链接到本地环境。

在项目根目录执行：

```bash
npm link
```

和下面图片一样，基本就说明成功了。

![npm link 成功后的终端输出](https://cdn.kmoon.fun/2026/2026-06-08T11-45-00-454Z.png)

这个阶段主要是本地开发调试方便。我后面把它正式发到了 npm 上，所以现在除了 `npm link` 这种本地联调方式之外，也可以直接下载安装：

```bash
npm i zhlgd-cli
```

能走到这一步，我自己其实还挺开心的。因为它说明这个项目已经不只是我本地电脑上的一个小玩具，而是真的变成了一个别人也能装下来体验的 CLI。

然后测试一下我们的命令。新开一个终端输入 `zhlgd`，就可以直接看到输出：

![首次运行 zhlgd 命令的输出](https://cdn.kmoon.fun/2026/2026-06-08T11-44-22-324Z.png)

到这一步，其实最小可用 CLI 就已经搭起来了。后面要做的，就是继续往里面塞真正有用的子命令。

## zhlgd help

接下来先完成第一个最简单的命令，也是几乎每个 CLI 都会有的命令：`help`。

这里认识一个新东西，官方很常见的 CLI 开发框架：[commander](https://github.com/tj/commander.js)。

先安装一下依赖：

```bash
npm install commander
```

写着写着我发现，`help` 其实根本不用自己手搓，框架已经给你做好了。

下面是使用 commander 改造后的入口文件 `index.js`：

```js
#!/usr/bin/env node

// 导入 commander 库，用于处理命令行
import { Command } from 'commander';
const program = new Command();

// 配置 CLI 基础信息
program
  .name('zhlgd')
  .version('1.0.0', '-v, --version')
  .description('一个基于 Node.js 开发的智慧理工大 CLI 工具');

// 解析命令行输入的参数
program.parse(process.argv);
```

这一步虽然简单，但我感觉挺关键。因为从这里开始，这个项目才不只是一个“能打印一句话的脚本”，而是开始往真正的 CLI 工具靠了。

运行一下看看效果：

![zhlgd help 命令输出](https://cdn.kmoon.fun/2026/2026-06-08T12-31-14-420Z.png)

## zhlgd login

下面开始啃一个硬骨头：**登录**。毕竟只有先登录，后面的事情才做得下去。

智慧理工大的登录是单点登录，大概流程如下：

1. 用户输入账号、密码 → 点击登录
2. 前端校验：账号/密码不能为空
3. 记住密码：写入 Cookie（加密存储）
4. 向后端请求 **RSA 公钥**
5. 使用公钥 **加密账号、密码**
6. 把加密后的账号密码塞进表单隐藏域
7. 自动提交登录表单 → 后端验证 → 登录成功 / 失败

每次登录时，前端都会先请求后端获取 RSA 加密公钥：

```js
// 获取 key
$.ajax({
  url: 'rsa?skipWechat=true',
  dataType: 'json',
  type: 'POST',
  success: function (data) {
    var encrypt = new JSEncrypt();
    encrypt.setPublicKey(data.publicKey);
    $('#ul').val(encrypt.encrypt(u));
    $('#pl').val(encrypt.encrypt(p));
    $('#loginForm')[0].submit();
  },
});
```

网页里这一步看起来很自然，因为浏览器已经帮你把表单提交、跳转、Cookie 处理这些事情都包掉了。但放到 CLI 里，这些步骤就得自己一层层补出来。

也就是说，在 CLI 里，登录这件事至少要做下面几步：

1. 请求 RSA 公钥
2. 用公钥加密账号和密码
3. 提交登录请求，拿到 ST ticket
4. 根据 ticket 继续跟重定向，换取完整 Cookie
5. 把 Cookie 保存到本地，后续命令统一复用

对用户名和密码加密之后，再提交登录请求获取 ST ticket，接着顺着重定向把完整 Cookie 换回来，并保存到本地，后续都拿这份 Cookie 作为鉴权使用：

```js
// 5. 用 ST 换取完整 Cookie
const st = extractTicket(loginResp.headers.location);
await followRedirects(`${SERVICE}?ticket=${st}`, cookieJar, commonHeaders);

// 6. 保存
saveCookies(cookieJar);
```

这里我自己的体感是，**登录这一关是整个 CLI 最麻烦的部分**。因为一旦 Cookie 这套机制没打通，后面所有需要鉴权的接口都跑不起来；但只要这一关打通了，后续子命令反而会顺很多，本质上就变成“带上鉴权去请求接口，再把结果整理一下输出”。

下面演示一下登录过程。这里为了安全，密码输入做了隐藏处理：

![zhlgd login 命令的隐藏密码输入效果](https://cdn.kmoon.fun/2026/2026-06-14T02-16-43-242Z.png)

到这里，`login` 命令就算是跑通了。现在这个项目也已经正式发到了 npm 上，直接执行下面这条命令就可以安装：

```bash
npm i zhlgd-cli
```

![npm](https://cdn.kmoon.fun/2026/2026-06-14T03-20-20-659Z.png)

所以如果你只是想直接上手体验，而不是自己从头本地联调，走 npm 安装会更方便。具体实现细节如果感兴趣，也可以直接看源码：[zhlgd-cli GitHub 仓库](https://github.com/kmoonn/zhlgd-cli)。

## zhlgd msg & news

登录搞定之后，我先做了两个最基础、也最容易验证成果的功能：

- `zhlgd msg`：查看我的消息
- `zhlgd news`：查看校园新闻

之所以先做这两个，是因为它们很适合拿来验证整条链路是不是已经通了：

- 登录是否真的拿到了可用 Cookie
- Cookie 是否已经被正确保存
- 后续子命令是否能稳定复用这套鉴权能力

实现上其实没有想象中那么玄学，核心就是把两个接口分别转成 JavaScript 请求：

```js
const data = await apiRequest(
  `${BASE_URL}/tp_up_new/up/newhome/getFpMsgList`,
  { MSG_TYPE: '3' },
);

const data = await apiRequest(
  `${BASE_URL}/tp_up_new/up/newhome/getNewsNotice`,
  { channel: '13962' },
);
```

真正关键的地方，还是请求时要把登录成功后保存下来的 Cookie 带上：

```js
/**
 * 带鉴权的 API 请求，未登录时自动提示
 */
export async function apiRequest(url, body) {
  if (!isLoggedIn()) {
    console.log('⚠️  未登录，请先执行 zhlgd login');
    process.exit(1);
  }
  const cookies = loadCookies();
  const resp = await axios.post(url, body, {
    headers: {
      ...API_HEADERS,
      cookie: cookieHeader(cookies),
      'User-Agent': UA,
    },
  });
  return resp.data;
}
```

所以说，前面把 `login` 打通之后，后面很多功能的开发思路就都统一了：

1. 找到对应接口
2. 搞清楚请求参数
3. 带上登录态发请求
4. 把返回结果整理成终端里更适合看的样子

下面测试一下实际效果：

![zhlgd msg 命令输出效果](https://cdn.kmoon.fun/2026/2026-06-14T02-22-55-918Z.png)

![zhlgd news 命令输出效果](https://cdn.kmoon.fun/2026/2026-06-14T02-23-11-367Z.png)

看到这里，我自己其实还挺有成就感的。因为它已经不再只是一个能打印几行字的 demo，而是真的具备了“登录 -> 带鉴权请求 -> 输出结果”这一整套能力。

## 写在最后

这次折腾 `zhlgd-cli`，对我来说最有意思的地方，不是把 `help` 命令跑起来，而是第一次比较完整地把“网页里浏览器偷偷帮你做的事情”，一点点搬到了 CLI 里。

目前这个 CLI 至少已经把最难的登录流程打通了，也做出了 `msg` 和 `news` 这两个能实际用起来的子命令。后面如果继续往下做，其实就可以顺着这条路继续扩，比如把更多查询类功能搬进终端，甚至再往前走一步，给 Agent 直接调用。

总之，这次尝试让我更直观地感受到一件事：CLI 这种东西，看起来很“黑框”，但一旦场景选对了，真的会很顺手。尤其是在 AI 时代，它不只是给人用的工具，很多时候也天然适合给 Agent 干活。
