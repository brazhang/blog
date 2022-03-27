---
title: 使用vue开发桌面应用
categories:
  - 技术
tags:
  - Vue
date: 2021-09-24 09:11:07
---

Vue 不仅限于开发 Web 和[原生移动](https://vue-community.org/guide/ecosystem/mobile-apps.html)应用程序，它还允许构建桌面应用程序。以下是人们选择 Vue 而不是本地语言解决方案的一些重要原因：

- **跨平台**：所有应用程序均采用JavaScript开发，可打包为Windows/MacOS/Linux；
- **易于构建**：框架允许您简单地开发 Web 应用程序，然后使用打包程序将其“转换”为桌面应用程序；
- **社区**：如果您维护一个开源桌面项目，您将更有可能为它找到贡献者。

然而，有一些缺点，所有 JavaScript 驱动的桌面应用程序都很常见。它们往往具有较大的包大小（至少 30 MB），并且已知内存使用量很大。

使用 Vue 构建桌面应用程序主要有两种方式：基于 HTML+CSS 的解决方案或原生 GUI。第一个是 Electron 和 NW.js，第二个是 Vuido。



#### [Electron](https://electronjs.org/)

[Electron](https://electronjs.org/)是由 GitHub 开发的开源库，用于使用 HTML、CSS 和 JavaScript 构建跨平台桌面应用程序。Electron 通过将[Chromium](http://www.chromium.org/)和[Node.js](https://nodejs.org/en/)组合成一个运行时来实现这一点，并且可以为 Mac、Windows 和 Linux 打包应用程序

Electron 是目前最流行的用于编写 JavaScript 桌面应用程序的库。一些比较知名的应用程序是[Slack](https://slack.com/)和[Discord](https://discordapp.com/) messengers、[Atom](https://atom.io/)代码编辑器和[Visual Studio Code](https://code.visualstudio.com/) IDE。

| 优点                                                         | 缺点                 |      |
| ------------------------------------------------------------ | -------------------- | ---- |
| 容易上手                                                     | 打包比较大           |      |
| 良好的开发文档                                               | 内存占用高           |      |
| 无需更改现有源代码                                           | 包中未受保护的源代码 |      |
| 有[Vue CLI 插件](https://github.com/nklayman/vue-cli-plugin-electron-builder) |                      |      |
| 成熟的开发者社区                                             |                      |      |

#### [NW.js](https://nwjs.io/)

[NW.js](https://nwjs.io/)（以前称为 node-webkit）是一个使用 HTML、CSS 和 JavaScript 构建桌面应用程序的框架。与[Electron](https://vue-community.org/guide/ecosystem/desktop-apps.html#electron)类似，它基于 Chromium 和 Node.js。NW.js 允许您直接从浏览器调用 Node.js 代码和模块，并在您的应用程序中使用 Web 技术。此外，您可以轻松地将 Web 应用程序打包为本机应用程序。

| 优点                                                         | 缺点                    |
| ------------------------------------------------------------ | ----------------------- |
| 容易上手                                                     | 打包比较大              |
| 无需更改现有源代码                                           | 内存占用高              |
| 编译为受保护的二进制文件                                     | 使用率明显低于 Electron |
| 有[Vue CLI 插件](https://github.com/NataliaTepluhina/vue-cli-plugin-nwjs) |                         |

#### [Quasar Framework](https://quasar.dev/)

[Quasar Framework](https://quasar.dev/)是一个允许跨平台应用程序开发的框架。使用大型 VueJS 组件库设计您的应用程序，然后使用 Quasar 强大而简单易用的 CLI 通过 Electron 为桌面自动构建您的应用程序。如果您希望对您的代码做更多的事情而不仅仅是 Electron 应用程序，那么 Quasar 是您跨平台应用程序想法的绝佳解决方案。

| 优点                                               | 缺点         |
| -------------------------------------------------- | ------------ |
| 容易上手                                           | 桌面也比较大 |
| 无需更改现有源代码                                 | 内存占用高   |
| 可以打包到各种平台，Android，iOS，以及桌面，网页等 |              |

#### [Vuido](https://vuido.mimec.org/)

[Vuido](https://vuido.mimec.org/)是一个基于 Vue.js 创建原生桌面应用程序的框架。使用 Vuido 的应用程序可以使用原生 GUI 组件在 Windows、OS X 和 Linux 上运行。

在幕后，Vuido 使用[libui](https://github.com/andlabs/libui)库为每个桌面平台提供原生 GUI 组件，以及用于 Node.js的[libui-node](https://github.com/parro-it/libui-node)绑定。

Vuido 和 Electron 或 NW.js 之间的核心区别在于，您不会在 Vuido 中使用 HTML 标签或 CSS 样式，您只能使用操作系统平台提供的一组预定义的 UI 组件。

| 优点                                                  | 缺点                                   |
| ----------------------------------------------------- | -------------------------------------- |
| 容易上手                                              | 外观仅限于操作系统本机 GUI 组件        |
| 无需更改现有源代码                                    | 没有 Vue CLI 插件，只有 Vue CLI 2 样板 |
| 可以打包到各种桌面版本                                |                                        |
| 与 Electron 或 NW.js 应用程序相比，提供更小的封装尺寸 |                                        |
| 有据可查                                              |                                        |



总结，如果你想使用vue来构建全平台应用，而且还要颜值OK的话，选择[Quasar Framework](https://quasar.dev/) 是正确的
