---
title: npm 私有库开发和发布规范
date: 2023-09-21 14:04:34
tags: 
- npm
- verdaccio
categories: node相关
---
## 前言
通过 {% post_link npm-verdaccio %} 已经建立了团队内部的私有 npm，并且也知道怎么去进行私有包的发布。

那这一节讲一下怎么规范的进行包的开发，以及要发布的一些规范，当然不同的团队有自己的规范，仅供参考。

## 开发规范
主要包含以下几个环节:
1. 编写 `README.md` 文件，描述包的使用方法和注意事项
2. 提供类型声明文件 (`.d.ts`)
3. 提供 demo，方便使用者快速上手
4. 提供单元测试，确保包的正确性
5. 在导出的模块的函数中添加JSDoc注释

### 1. 编写 `README.md` 文件
`README.md` 文件无疑是安装者了解这个包的第一途径，因此是非常重要的。 通常README文件应该至少包含以下内容：
1. **项目名称和描述**：在文件的开头，应该清楚地标明项目的名称和描述，让读者了解这个项目是做什么的；**注意**: 描述会显示在npm包页面的概览中，所以请确保它是有意义的。
2. **安装指南**：提供详细的步骤，说明如何安装和设置这个包。这可能包括需要的依赖、环境变量等。
3. **使用示例**：提供一些基本的代码示例，让用户知道如何在他们的项目中使用你的包。
4. **API 文档**：如果你的包提供了API，你应该在readme中详细说明每个公开的函数或方法，包括它们的参数、返回值和可能抛出的错误。
5. **Changelog**: 应该提供版本 change log 记录或者跳转到 CHANGE_LOG.md 文件的外链

<!--more-->

### 2. 提供类型声明文件 (`.d.ts`)
如果你的包是用 TypeScript 编写的，那么你可以直接使用 TypeScript 编译器生成类型声明文件。如果你的包是用 JavaScript 编写的，那么你需要手动编写类型声明文件。

确保类型声明文件的完整性和正确性，这样用户在使用你的包时就可以获得更好的开发体验。

### 3. 提供 demo，方便使用者快速上手
我认为既然需要将一段逻辑抽成一个公共库，说明肯定是具备一定的复杂度的，因此在库开发过程中，一定要有 demo 目录，里面要提供可直接运行的 demo 实例

### 4. 提供单元测试，确保包的正确性
在发布之前，确保你的包是可以正常工作的。编写适当的测试，并使用测试框架（如[Jest](https://jestjs.io/)或[Mocha](https://mochajs.org/)）来运行它们。

如果是类似于组件库的，那么可能就会需要编写 e2e 测试，可以使用[playwright](https://playwright.dev/)

### 5. 在导出的模块的函数中添加[JSDoc](https://www.jsdoc.com.cn/)注释
为了减少库的使用成本，在导出的模块的函数中添加[JSDoc](https://www.jsdoc.com.cn/) 注释，给复杂的函数添加 example，这样用户在使用你的包时，就可以获得更好的开发体验。例如：

```javascript
/**
 * 两个数相加
 * @param {number} a 要相加的第一个数
 * @param {number} b 要相加的第二个数
 * @returns {number} 两个数的和
 *
 * @example
 * // returns 3
 * add(1, 2)
 */
export function add(a, b) {
  return a + b;
}
```

这样，当用户在编辑器中输入`add(`时，就会显示这个函数的说明、参数和返回值、example。

所以请尽可能详细的编写JSDoc注释，站在用户的角度，想想他们可能会遇到什么问题，然后在注释中解答这些问题。

## 发版规范
主要包含以下几个环节:
1. 打版本 tag
2. `package.json` 文件的信息要补充完整
3. 版本号遵循语义化版本规则
4. 包发布需要在`change log`中记录版本变更（使用单独的`CHANGE_LOG.md`或者在`README.md`中记录）


### 1. 打版本 tag
每次 release 发布的时候，都是一个稳定的版本，那么它的分支应该是在 master 分支，并且也应该为 master 分支打一个相同 release 版本的 tag。 以便后续可以回溯版本历史。

因此在发布包之前，你需要确保你的代码已经合并到 master 分支，并且 master 分支已经打上了tag，tag的格式为 `v1.0.0`，其中`1.0.0`为发布的版本号。

### 2. `package.json` 文件的信息要补充完整
发布一个npm包时，你需要在 `package.json` 文件中填写一些重要的字段。以下是一些必要和建议填写的字段：

**必要的字段**:
1. `name`：你的包的名字。这个名字必要且唯一，不能与npm仓库中已有的其他包名冲突。
2. `version`：你的包的版本号。遵循[语义化版本](https://semver.org/lang/zh-CN/)规则。
3. `main`：指定入口文件的位置，其他人在引用你的包时，会默认引用这个文件。

**建议的字段**:
1. `description`：对你的包进行描述，这将在npm网站或命令行工具中显示。
2. `keywords`：一组关键词，有助于在npm中搜索到你的包。
3. `author`：包的作者信息。
4. `module`：指定 ES6 模块的入口文件位置。（使用ESM规范引入文件时，通常会引入module字段指向的文件，main字段通常用于指向CommonJs规范的文件）
5. `files`: 一个数组，当你的包被发布或安装时，只有在 files 字段中列出的文件或文件夹才会被包含。
6. `scripts`：包含脚本的对象，用于运行测试或构建你的代码等。
7. `repository`：项目的仓库地址。
8.  `bugs`：一个URL，用户可以通过这个URL报告你的包的问题。
9.  `homepage`：你的项目的主页URL。
10. `engines`： 用于指定项目运行所需的 node 和 npm 版本。这对确保你的项目在特定的 Node.js 或 npm 环境中运行非常重要。
11. `exports`：这个字段允许你更精确地控制模块的公开API，以及提供一种方式来优雅地加载ESM和CommonJS版本的模块，[了解更多](https://webpack.js.org/guides/package-exports/)。
12. `types`：指定TypeScript类型定义文件的位置。如果 types 字段没有被指定，TypeScript 会默认查找项目根目录下的 index.d.ts 文件。
13. `publishConfig`：一个对象，用于配置发布包时的一些[参数](https://docs.npmjs.com/cli/v10/using-npm/config)，比如发布的仓库地址、发布的tag等。

这是一个基本的`package.json`文件示例：
```json
{
  "name": "your-package-name",
  "version": "1.0.0",
  "description": "A brief description of your package",
  "main": "dist/index.js",
  "files": [
    "dist/*.js",
    "dist/*.css",
    "types/index.d.ts"
  ],
  "types": "types/index.d.ts",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/repository.git",
    "directory": "plugins/plugin-name" // 如果你的包在仓库的某个子目录下，可以指定这个字段
  },
  "keywords": ["keyword1", "keyword2"],
  "author": "Your Name <your.email@example.com>",
  "bugs": {
    "url": "https://github.com/username/repository/issues"
  },
  "engines": {
    "node": ">=14.0.0"
  },
  "homepage": "https://github.com/username/repository#readme",
  "license": "ISC",
  "dependencies": {
    "dependency1": "^1.0.0"
  },
  "devDependencies": {
    "devDependency1": "^1.0.0"
  },
}
```
这只是一个基础的例子，你可以根据你的实际需求调整这些字段。
> 更多内容请查看官方文档：[package.json](https://docs.npmjs.com/cli/v9/configuring-npm/package-json/)

### 3. 版本号遵循语义化版本规则
版本号发布应严格遵循 [语义化版本 2.0.0](https://semver.org/lang/zh-CN/) 规则, 它遵循 `MAJOR.MINOR.PATCH` 的格式, 所以开发人员要根据你本次发布的内容来合理升级包的版本号。

>  版本格式：主版本号.次版本号.修订号，版本号递增规则如下：
>
>  1. 主版本号：当你做了不兼容的 API 修改，
>  2. 次版本号：当你做了向下兼容的功能性新增，
>  3. 修订号：当你做了向下兼容的问题修正。
>
>  先行版本号及版本编译信息可以加到“主版本号.次版本号.修订号”的后面，作为延伸。

有时候会有先行版本的情况(也就下文说的预发布版本)，就是开发人员的代码还在自己分支中，但是要测试效果，这时候就可以在自己分支打 alpha 版的包，或者 beta 版的包，比如 `1.0.0-alpha.1`，`1.0.0-beta.1`
```text
{
  "name": "your-package",
  "version": "1.0.0-alpha.1",
  ...
}
```
然后安装就是:
```text
npm install your-package@1.0.0-alpha.1
or
npm install your-package@alpha
```
> 这时候要指定版本号或者 dist-tag 的方式来安装这些预发布的版本，不然默认是不会安装这些预发布版本的包的

> 默认的 dist-tag 是 latest

一直到测试没问题之后，将代码合并到 master 分支，然后打 release 版本，比如上述的 alpha 包如果要打 release 版本，那么版本号就应该是 `1.0.1` 版本。

如果是同时有多个开发人员在迭代这个包的情况，也是同样的道理，一个走 alpha tag，一个走 beta tag，然后各自在各自的开发分支 publish 这些预发布包，并进行安装测试，同时安装的时候，要指定具体的版本。 一直到测试没问题之后，将开发分支合并到 master 分支，才能打 release 版本出来

遇到这种预发布版本的时候，判断优先级的时候，必须把版本依序拆分为主版本号、次版本号、修订号及先行版本号后进行比较。由左到右依次比较每个标识符号，第一个差异值用来决定优先层级（其中字母连接号以ASCII排序进行比较、其他都相同时栏位多的先行版本号优先级较高），比如:
```text
1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-alpha.beta < 1.0.0-beta < 1.0.0 < 2.0.0 < 2.1.0 < 2.1.1
```

### 4. 包发布需要在`change log`中记录版本变更（使用单独的`CHANGE_LOG.md`或者在`README.md`中记录）
Change log（变更日志）是一个记录项目版本中所有显著更改的文档。它可以帮助用户和开发者理解项目的新版本中发生了什么更改。
> 具体可以看 [如何维护更新日志](https://keepachangelog.com/zh-CN/1.0.0/)

#### 怎样编写高质量的更新日志？

##### 1. 指导原则

* 记住日志是写给人而非机器的，所以要保持简洁易读。
* 同类改动应该分组放置，例如新特性、bug修复等。
* 不同版本应分别设置链接，方便用户查看特定版本的代码更改。
* 新版本在前，旧版本在后
* 应包括每个版本的发布日期

##### 2. 变动类型

1. `Added` 新添加的功能。
2. `Changed` 对现有功能的变更。
3. `Deprecated` 已经不建议使用，即将移除的功能。
4. `Removed` 已经移除的功能。
5. `Fixed` 对 bug 的修复。
6. `Security` 对安全性的改进。

#### 如何减少维护更新日志的精力？

在文档最上方提供 `Unreleased` 区块以记录即将发布的更新内容。

这样做有两个好处：
- 大家可以知道在未来版本中可能会有哪些变更。
- 在发布新版本时，直接将 `Unreleased` 区块中的内容移动至新发布版本的描述区块就可以了。

以下是一个Change log的样例：

```markdown
# ChangeLog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- 新增了新的用户界面
- 在设置中新增了语言选择选项

### Changed
- 更新了登录页面的设计
- 优化了搜索算法的性能

### Deprecated
- 弃用了老版本的用户界面

### Removed
- 移除了未使用的代码和库

### Fixed
- 修复了用户登录时的错误提示问题
- 修复了在某些情况下应用崩溃的问题

## [1.0.0] - 2022-01-01

### Added
- 初始版本发布，包含了基础的登录和搜索功能

[Unreleased]: https://github.com/yourusername/yourrepository/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/yourusername/yourrepository/releases/tag/v1.0.0
```

每一个版本应该包含以下信息：

- 版本号
- 发布日期
- 在这个版本中进行的更改

这就是一个基本的Change log的结构和样例。一个好的Change log应该是为了让读者更好的理解你的项目，所以请保持清晰和简洁。


## 将发布规范流程做成脚手架来检查
以上就是一个比较标准的团队内部开发 npm 包的一个规范流程，有很多的约束需要检查，光靠人工其实有点浪费。

比较好的情况是将以上标准流程的检查，做成一个发布脚手架，通过交互式命令发布包。目的是为了简化发布包的流程，减少出错的可能性。




