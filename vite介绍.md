## 定义
vite是一个主打极速开发+高效生产构建的打包工具，它利用**[浏览器原生ES模块](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Modules)**支持，实现开发时的快速热更新，生产时使用rollup打包，开发时使用esbuild打包。
## 核心原理
在开发时，vite让浏览器缓存文件，不用每次重新编译和重新下载。
它把文件分为两类，一类是**依赖模块**，一类是**源码模块**，对于这两种文件，vite采用了不同的缓存策略。
- 依赖模块：vue、react、lodash 这种第三方包，在开发时不需要重新编译，只需要在第一次编译时下载一次，之后直接从缓存中读取。
  - 策略：强缓存策略(max-age=31536000,immutable)，浏览器会缓存这些文件，除非手动清除缓存，否则浏览器会一直使用这些文件。
- 源码模块：开发时业务代码(包括非JS文件)。
  - 策略：协商缓存策略，浏览器会向服务器发送请求，服务器会根据请求头中的`If-None-Match`和`If-Modified-Since`字段来判断文件是否需要更新，如果需要更新，服务器会返回新的文件，否则返回304状态码，浏览器会从缓存中读取文件。
## vite的构建流程
### 开发环境的构建
#### 1.启动阶段
- 命令执行：运行指令`vite`或`npm run dev`，读取项目根目录下的vite.config.js配置文件，合并默认配置
- 依赖预构建:
  - 目的：虽然浏览器支持ESM，但有些第三方库可能还是CommonJS格式，或者内部模块相互依赖。如果浏览器直接请求，可能会触发大量HTTP请求，导致页面卡顿。
  - 过程：使用esbuild扫描代码，找出所有第三方依赖。
  - 结果：将依赖转化为ESM格式，并打包成一个文件存入到`node_modules/.vite`目录下。
  - 重新构建的条件：
    - package.json 中的 dependencies 列表
    - 包管理器的 lockfile，例如 package-lock.json, yarn.lock，或者 pnpm-lock.yaml
    - 可能在 vite.config.js 相关字段中配置过的
    只有在上述其中一项发生更改时，才需要重新运行预构建。
#### 2.请求与编译阶段
- 请求拦截：当浏览器访问locahost:5173(默认端口号)时，vite会拦截所有请求
- 处理HTML：首先返回index.html文件。浏览器解析HTML，遇到`<script type="module">`标签，会向服务器请求该模块。
- 处理JS：
  - vite会根据请求的路径，找到对应的文件。
  - 通过**转换插件链**处理文件（例如：将 .vue 单文件组件拆解成 JS/CSS/Template，将 TypeScript 转为 JavaScript，将 Sass 转为 CSS）。
  - 路径重写：如果代码中有 import { ref } from 'vue'，Vite 会将其重写为指向预构建好的缓存路径 /node_modules/.vite/deps/vue.js。
  - 返回处理后的JS代码。
- 递归加载：浏览器解析返回的JS，发现新的import依赖(如 import App from './App.vue')，会再次向服务器请求，vite再次拦截并处理，直到所有资源都加载完成。
#### 热更新(HMR)阶段
- 监听变化： Vite通过chokidar监听文件系统变化。
- WebSocket通信：当保存修改文件时，Vite通过WebSocket向客户端发送消息，告知哪个模块发生变化。
- 局部更新：客户端(指浏览器里运行的Vite内置客户端脚本)收到消息后，通过 import 动态重新请求该模块，并利用 HMR API 接口替换旧模块，保持页面状态不丢失。
### 生成环境的构建
#### 1.构建准备
- 命令执行：运行指令`vite build`
- 加载配置：读取ite.config.js中的build配置项。
#### 2.Rollup打包流程
- 入口解析：
  - 找到入口文件，默认是`index.html`中引入的JS(注意在vite项目中`index.html`应在根目录下面)。
- 依赖图构建：
  - 从入口开始，使用解析器将代码解析为AST(抽象语法树)。
  - 分析import和export语句，构建模块依赖关系图。
  - 在这个过程中，Vite回应用各种插件来转换代码。
- 代码转换与Tree-shaking：
  - 转换：将非标准代码(JSX,TS,Sass等)转换为标准的ES5/ES6代码。
  - Tree-shaking：标记未引用的代码或未被使用的导出，并在生成阶段将其删除，减少打包体积。
- 代码分割：
  - 根据配置(如manualChunks)或自动分析，将代码拆分成多个chunk。
- 生成产物：
  - 将处理号的各个chunk写入磁盘(默认是dist目录)。
  - 生成.css文件(提取JS中的CSS)
  - 生成资源文件(如图片、字体等)，并加上hash值，防止缓存问题。
  - 替换环境变量(如process.env.NODE_ENV)为对应的值。
  - 移除开发时的代码(如console.log)。
#### 3.后处理
- 生成manifest：如果配置了`build.rollupOptions.output.manualChunks`，vite会生成一个manifest文件，用于记录chunk的名称和对应的文件路径。
- HTML注入：Vite会回读index.html，将打包生成的JS/CSS文件注入到HTML中，替换开发时的路径。
## vite钩子
### enforce修饰符
Vite 插件可以通过 enforce 属性来调整钩子的执行顺序（相对于 Vite 内核插件）
- `pre`：在Vite核心插件之前执行。
- `默认(无)`：在Vite核心插件之后执行。
- `post`： 在Vite核心插件和其他默认插件之后执行。
### 独有钩子
- `config`：在解析Vite配置前调用，可以修改或合并配置(修改对象属性或返回新对象)。
- `configResolved`：在解析Vite配置后调用，可以获取到最终配置。
- `configureServer`：在开发服务器启动前调用，可以添加中间件、代理、WebSocket(**只在 dev 环境执行**)。
- `transformIndexHtml`：转换 index.html 的专用钩子。在HTML解析后调用，可以修改HTML内容。
- `handleHotUpdate`：在执行默认的 HMR（热模块替换）逻辑之前，自定义对文件变更的处理方式。
### 模板解析钩子
- `resolveId`：在解析模块时调用(即遇到一个import语句)，可以修改模块引入路径。
- `load`：在resolveId解析成功后，加载模块内容时调用，可以修改加载的内容或为虚拟模块提供代码。
- `transform`：在文件被加载后、但传递给其他插件或浏览器之前调用，对代码进行转译、修改。
### Rollup插件钩子
- `buildStart`：在rollup构建开始时调用。
- `buildEnd`：在rollup构建结束时调用。
- `outputOptions`：在rollup生成输出配置时调用，可以修改输出配置。
- `moduleParsed`：在rollup解析模块时调用，可以修改模块解析结果。
- `renderStart`：在rollup准备开始生成输出文件时调用，可以修改输出文件。
- `renderChunk`：对每个生成的chunk进行操作（如压缩）。
- `generateBundle`：所有chunk生成完毕，生成最终文件对象。
- `writeBundle`：在所有文件写入磁盘后调用。
- `closeBundle`：在rollup构建关闭时调用。
### 钩子的执行顺序
#### 开发环境
- `config` -> `configResolved` -> `configureServer` -> `buildStart` -> `transformIndexHtml`(注:浏览器请求的是HTML时) -> ?(`handleHotUpdate` -> (注：仅当文件变更时触发)) `resolveId` -> `load` -> `transform`
#### 生产环境
- `config` -> `configResolved` -> `outputOptions` -> `buildStart` -> `resolveId` -> `load` -> `transform` -> `moduleParsed` -> `buildEnd` -> `renderStart` -> `renderChunk` -> `generateBundle` -> `transformIndexHtml` -> `writeBundle` -> `closeBundle`

