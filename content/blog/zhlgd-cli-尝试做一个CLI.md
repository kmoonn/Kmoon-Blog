+++
title = "zhlgd-cli: 尝试做一个 CLI"
date = "2026-06-07T17:01:02+08:00"
tags = ["CLI"]
+++

一直觉得在命令行里敲命令然后回车，有一种莫名的爽感，特别是当黑框里返回自己想要的东西的时候。

AI 如今发展的很快，CLI（Command Line Interface）被重新推到了历史舞台中央，因为日常我们使用的可视化操作界面（GUI）对于 AI 来说并不友好。对于 AI 来说，可以通过命令直接调用并且返回清晰的结果，才是真正友好且适合 AI 工作的。

目前互联网上最火的 CLI 应该是字节跳动开源的[飞书 CLI（lark-cli）](https://github.com/larksuite/cli/blob/main/README.zh.md)，可以让人类和 Agent 都能在终端中操作飞书，比如收发消息、创建文档、多维表格、查询日历和会议等等。

CLI 对于 Agent 来说也是一种工具调用，而且是更适合的一种工具，之前也看到文章讨论 CLI 是否会取代 MCP。

所以！为了紧跟 AI 发展，我决定尝试自己写一个 CLI，看看怎么个事！

## 先想需求

不管干什么，总得得结合实际需求，毕竟没实际用途干着也没劲。仔细想了想日常工作也就来自两方面：学校、公司。公司这边肯定没法下手（压根不需要我），字节这边每天都会有层出不穷的工具，各种花色任你选择，说实话真有点“军备竞赛”那味道了。

所以我把思路转移到学校这边，学校这边能做的感觉挺多的，毕竟学校的网站、各种平台都比较老旧（有点跟不上时代）。突然想到智慧理工大（武理的一个大 OA 平台）应该是日常大家使用最多的平台，而且最近刚好网站改版，让我们把它 CLI 化吧！

## 开发流程

第一步肯定是要了解一下 CLI 的开发流程，我现在除了知道应该是要用 node 开发，其他完全零基础啊。好的，开始上网搜搜！

![](https://cdn.kmoon.fun/2026/2026-06-08T12-32-53-351Z.png)

找到了一篇 2019 年的知乎文章：[用Node.js开发一个Command Line Interface (CLI)](https://zhuanlan.zhihu.com/p/38730825)，本来我看到这个时间就不打算继续看了，但是换个思路一想，这么久远的还能被我搜到，说不定是一篇优质好文，而且这个标题也是直抒胸臆，所以我决定仔细看一看。

看完后大概知道是什么流程了，如我们所料啊！的确是用 Node.js 开发，难怪平时网上安装 CLI 的时候都会让我确保本地有 node 环境，而且安装的时候也是用 npx 安装。

开发流程主要分下面这几步：

1. 创建 npm 模块
2. 增加 bin 入口文件
3. 全局安装
4. 发布

发布到社区就可以给大家使用了，是不是看起来很简单，我们继续下一步。

## 环境配置

简单配置一下 [node](https://nodejs.org/zh-cn) 环境，我平时一般用 [nvm](https://www.nvmnode.com/)  管理 node 环境，毕竟因为版本不同排查过很久一次 bug。

node 环境配置好后检查一下：

![](https://cdn.kmoon.fun/2026/2026-06-08T11-45-40-376Z.png)

创建 npm 模块：npm init

![](https://cdn.kmoon.fun/2026/2026-06-08T12-36-11-738Z.png)

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

> - type 这里选择 module，因为新版 ES 模块化（用 import/export，现代前端 / CLI 工具必用）
> - commonjs：老版 Node.js 模块化（用 require() 导入）

在 package.json 文件增加 bin 的对象：

```json
{
  "bin": {
    "zhlgd": "./bin/index.js"
  }
}
```

在项目根目录新建一个 bin 文件夹，并在里面新建一个 index.js 文件。

文件结构：

```
zhlgd-cli/
├── bin/
│   └── index.js
└── package.json
```

编写 bin/index.js 代码

第一行必须写：#!/usr/bin/env node（告诉系统用 node 运行）

```js
#!/usr/bin/env node

// 你的命令行逻辑开始
console.log('✅ 智慧理工大命令行工具启动成功！');
console.log('👉 命令：zhlgd');
```

全局安装，把命令链接到全局。

在项目根目录执行：

```bash
npm link
```

和下面图片一样就说明成功了

![](https://cdn.kmoon.fun/2026/2026-06-08T11-45-00-454Z.png)

然后我们测试一下我们的命令，新开一个终端输入：`zhlgd-cli`，可以直接看到输出：

![](https://cdn.kmoon.fun/2026/2026-06-08T11-44-22-324Z.png)

后面我们继续开发更多的子命令就可以啦。

## zhlgd help

接下来让我们完成第一个最简单的命令，也是每个 CLI 都有的命令，那就是 `help` 查看帮助信息。

我们认识一个新东西，官方推荐的 CLI 开发框架：[commander](https://github.com/tj/commander.js)。

先安装一下依赖

```
npm install commander
```

写着写着发现不用自己写 help 命令，框架已经自带了。

下面是使用框架修改后的入口文件 index.js ：

```js
#!/usr/bin/env node

// 导入 commander 库，用于处理命令行
import { Command } from 'commander';
const program = new Command();

// 配置 CLI 基础信息
program
    .name('zhlgd')          // 命令名称
    .version('1.0.0', '-v, --version')  // 版本号
    .description('一个基于 Node.js 开发的智慧理工大 CLI 工具');  // 描述

// 解析命令行输入的参数
program.parse(process.argv);
```

运行一下看看效果：

![](https://cdn.kmoon.fun/2026/2026-06-08T12-31-14-420Z.png)

## zhlgd login

下面啃一个硬骨头：**登录**，毕竟只有登录了才能做后面的事情。

1. 用户输入账号、密码 → 点击登录
2. 前端校验：账号/密码不能为空
3. 记住密码：写入Cookie（加密存储）
4. 向后端请求 **RSA公钥**
5. 使用公钥 **加密账号、密码**
6. 把加密后的账号密码塞进表单隐藏域
7. 自动提交登录表单 → 后端验证 → 登录成功/失败

```js
//获取key
$.ajax({
    url : "rsa?skipWechat=true",
    dataType : "json",
    type : "POST",
    success:function(data){
    var encrypt = new JSEncrypt();
    encrypt.setPublicKey(data.publicKey);
    $("#ul").val(encrypt.encrypt(u));
    $("#pl").val(encrypt.encrypt(p));
    $("#loginForm")[0].submit();
    }
});
```