### 常见功能
#### import.meta.glob
从文件系统导入多个模块，并返回一个对象，其中键是文件路径，值是导入的模块。
##### 两种模式
- 懒加载模式(默认): 返回函数对象，调用才会加载
```
const modules = import.meta.glob('./dir/*.js')

// 结构：
// {
//   './dir/a.js': () => import('./dir/a.js'),
//   './dir/b.js': () => import('./dir/b.js')
// }
```
使用
```
for (const path in modules) {
  modules[path]().then((mod) => {
    console.log(mod.default) // 拿到模块内容
  })
}
```
- 立即加载模式（eager）: 直接返回模块内容
```
const modules = import.meta.glob('./dir/*.js', { eager: true })

// vite 生成的代码
import * as __glob__0_0 from './dir/foo.js'
import * as __glob__0_1 from './dir/bar.js'
const modules = {
  './dir/foo.js': __glob__0_0,
  './dir/bar.js': __glob__0_1
}

```
注意：
- 所有 import.meta.glob 的参数都必须以字面量传入。你 不 可以在其中使用变量或表达式。
- 该 Glob 模式会被当成导入标识符：必须是相对路径（以 ./ 开头）或绝对路径（以 / 开头，相对于项目根目录解析）或一个别名路径
- 立即加载模式如果需要加载的文件较多，可能会阻塞页面的渲染，不会阻塞HTML的解析
### 导入静态资源会返回解析后的URL
```
import imgUrl from './img.png'
document.getElementById('hero-img').src = imgUrl
```
### 支持动态导入
```
const module = await import(`./dir/${file}.js`)
```
注意：变量仅代表一层深的文件名。如果 file 是 foo/bar，导入将会失败。
### --force使用
作用：强制重新构建所有模块，跳过缓存
使用场景：
- 给内部包 package.json 加了新依赖
- 改了内部包的 dependencies
- 改了内部包的 main/module/types 入口
- 重新 pnpm install
- 内部包引入了新的第三方依赖
### 资源导入识别
vite默认支持识别的文件类型：png|jpeg|gif|svg|mp4|webm|ogg|mp3|wav|flac|aac|woff|woff2|eot|ttf|otf
- ?url: 将资源作为一个 URL 导入
- ?raw: 将资源作为一个字符串导入
- ?inline:  CSS不注入，返回字符串
- ?worker → 当作Web Worker导入
### .env文件
- .env: 所有环境都会加载
- .env.local: 所有环境都会加载，但会被 git 忽略
- .env.[mode]: 只在指定模式下加载
- .env.[mode].local: 只在指定模式下加载，但会被 git 忽略
- .env.development.local: 只在开发模式下加载，但会被 git 忽略
- .env.production.local: 只在生产模式下加载，但会被 git 忽略
- .env.development: 只在开发模式下加载
- .env.production: 只在生产模式下加载
注意：为了防止意外地将一些环境变量泄漏到客户端，只有以 VITE_ 为前缀的变量才会暴露给经过 vite 处理的代码。
```
VITE_SOME_KEY=123
DB_PASSWORD=foobar
```
```
console.log(import.meta.env.VITE_SOME_KEY) // 123
console.log(import.meta.env.DB_PASSWORD) // undefined
```
### 后端集成
#### 目的
让后端负责渲染HTML的项目，也能完整用上vite的现代前端开发能力（毫秒级HMR、原生ESM、TS/JSX编译、资源处理、打包优化等）。
#### 原因
HTML由后端引擎渲染，前端无法直接控制index.html，用不了vite的HRM等能力。
#### 实现
##### 开发环境：后端渲染HTML+vite提供前端资源
核心就是让后端生成的HTML，直接对接vite的开发服务，完整复用HMR能力。
1. 启动两个服务器：后端服务(负责HTML渲染，提供接口)，Vite dev server(负责前端资源和热更新)。
2. 后端渲染的HTML模板中，固定注入两段核心脚本:
   - Vite的HMR客户端：<script type="module" src="http://localhost:5173/@vite/client"></script>
   - 前端入口文件：<script type="module" src="http://localhost:5173/src/main.js"></script>
3. 浏览器访问后端服务，拿到渲染后的HTML，会从Vite dev serve拉取前端资源。vite实时编译TS/VUE/JSX等文件，通过Websocket实现HMR热更新，改代码无需刷新页面。
4. 页面与接口完全同源，无跨域问题，后端的模板变量、权限控制等能力完全保留。
##### 生产环境：后端动态引入静态资源+vite打包
核心是manifest.json解决打包后文件带哈希的路径映射问题：
1. Vite配置中开启manifest：true，执行vite build后，会再打包目录中生成.vite/manifest.json文件，记录源文件与打包后带哈希的文件的映射关系，同时处理CSS，预加载器等依赖。
2. 把vite打包后的静态资源，复制到后端项目的静态资源目录(如 SpringBoot 的resources/static、Laravel 的public目录)。
3. 后端渲染HTML时，读取manifest.json，根据源文件名找到对应的打包后文件路径，动态生成```<script>```，```<link>```标签注入到HTML中，自动处理哈希，资源路径，预加载等问题。
### 虚拟模块
Vite 虚拟模块（Virtual Modules） 是一种不存在于物理文件系统、由插件动态生成的 ESM 模块，核心作用是**在编译时向代码注入动态内容**，常用于插件开发、构建信息注入、动态配置、代码生成等场景。
1. 核心概念：
- 本质：通过vite/rollup插件中的resolveId和load钩子，拦截特定模块ID请求，在内存中返回代码，而非读取磁盘文件。
- 命名规范(Vite官方约定):
  - 用户导入: 使用virtual前缀（如 virtual:build-info）。
  - 内部ID：插件内部使用\0空字符前缀（如 \0virtual:build-info），防止冲突。
2. 核心作用
- 动态内容注入：注入构建时间、Git 信息、版本号、环境变量等编译时数据。
- 无文件代码生成：动态生成路由、国际化、工具函数等，不产生物理文件。
- 插件能力扩展：框架 / 插件向用户代码暴露 API（如 Vue 的 virtual:vue、UnoCSS 的工具类）。
- 开发体验优化：支持 HMR，修改虚拟模块内容可实时更新。
```
// vite.config.js
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [
    {
      name: 'virtual:build-info', // 插件名
      // 1. 解析模块 ID
      resolveId(id) {
        if (id === 'virtual:build-info') {
          return '\0virtual:build-info' // 内部唯一 ID
        }
      },
      // 2. 加载模块内容
      load(id) {
        if (id === '\0virtual:build-info') {
          // 动态生成代码
          return `
            export const buildTime = "${new Date().toISOString()}";
            export const version = "${process.env.npm_package_version}";
            export const mode = "${process.env.NODE_ENV}";
            export const gitHash = "${process.env.GIT_COMMIT_HASH || 'unknown'}";
          `
        }
      }
    }
  ]
})
```
```
// 业务代码
import { buildTime, version, mode, gitHash } from 'virtual:build-info'

console.log('构建时间:', buildTime)
console.log('版本:', version)
console.log('模式:', mode)
console.log('Git Hash:', gitHash)
```
### HMR API
常见功能：
- accept(cb)：当前模块支持热更新，当模块更新时调用回调函数。
- accept(deps, cb)：接受直接依赖项的更新，而无需重新加载自身。
- dispose(cb)：模块更新前，清理上一次模块留下的副作用。例如：销毁定时器、解绑 DOM 事件、删除全局变量、清理副作用等。
- data：于将信息从模块的前一个版本传递到下一个版本。
- decline()：表示此模块不可热更新，如果在传播 HMR 更新时遇到此模块，浏览器应该执行完全重新加载。
- on(event, cb)：监听 HMR 事件，如：connected、update、dispose、error 等。
热更新流程：
```
修改代码 → Vite 监测到 → 重新构建模块 → 通知浏览器 → 清理旧模块 → 加载新模块 → 执行更新
```
具体实例或其他API，请参考[官网文档](https://vitejs.cn/vite3-cn/guide/api-hmr.html#hot-invalidate)

### 代码压缩
#### js压缩
- esbuild(默认)
  - 基于Go编写的web打包工具，支持多线程打包，处理速度快。对于大多数的js语法都能够处理，但对于一些老旧浏览器(如IE11)的非标准语法无法进行兼容处理。
- terser
  - 基于nodejs编写的js打包工具，不支持多线程，需要借助terser-webpack-plugin/vite-plugin-terser-parallel等插件，模拟多线程打包，处理速度慢。能对各类js语法进行兼容处理。
- 设置方式: build.minify: esbuild|terser|false
#### css压缩
- esbuild(默认)
- cssnano
  ```json
  postcss: {
    plugins: [
      cssnano({preset: 'advanced'}) // 极少需要修改默认配置
    ]
  }
  ```
- lightningcss
  ```json
  css: {
    // 转换CSS交给lightningcss
    transformer: 'lightningcss',
    lightningcss: {
      // 关于lightningcss的配置添加在这里
    }
  },
  build: {
    // 构建CSS交给lightningcss
    cssMinify: 'lightningcss',
  }
  ```
- @fullhuman/postcss-purgecss
  - 用于移除未使用的CSS
  ```json
    postcss([
      purgecss({
        content: ['./src/**/*.html']
      })
    ])
  ```
#### html压缩
vite默认不会对html进行压缩，但可以使用插件vite-plugin-html-minifier-terser来实现压缩。
```json
plugins: [
  minifyHtml({
    collapseWhitespace: true,
    removeComments: true,
    minifyCSS: true,
    minifyJS: true,
  }),
]
```
#### 静态资源压缩（图片、字体等）
vite-plugin-imagemin：在构建时自动压缩图片（png, jpg, svg 等）。
### tree-shaking
通过构建工具对ESM模块的导入与导出关系进行静态分析，移除掉未被使用的代码。在生产构建时，vite默认会自动启用tree-shaking。
#### 注意事项
- tree-shaking只支持ESM语法，若第三方依赖使用了非ESM语法，则无法生效。
- 导入第三方依赖时，最好按需引入，这样trees-shaking才能生效。
- 若导入的文件中存在副作用，则可能无法执行tree-shaking。
  - 副作用：在模块被导入时，立即执行了某些影响外部环境的操作，如修改全局变量、修改DOM、添加事件监听器等。
  ```js
  // utils.js
  // 1. 直接修改全局变量
  window.myUtils = { b: 1 }; 
  // 2. 修改DOM
  function modifyDOM() {
    const div = document.createElement('div')
    div.innerText = 'hello, world'
    document.body.appendChild(div)
  }
  modifyDOM();
  // 3. 添加事件监听器
  function addEventListener() {
    window.addEventListener('resize', () => {
      console.log('resize')
    })
  }
  addEventListener();
  export default function sum(a, b) {
    return a + b
  }
  ```
  - 解决方案：
  - 设置sideEffects属性
    - 在**package.json**中设置sideEffects属性，告诉构建工具哪些文件是副作用文件，哪些文件不是。
    ```json
    {
      "sideEffects": [
        "./src/**/*.css",
        "./src/**/*.scss",
        "./src/**/*.sass",
        "./src/**/*.less",
        "./src/**/*.styl",
        "./src/**/*.stylus",
        "./src/**/*.vue",
        "./src/**/*.html",
        "./src/**/*.md"
      ]
    }
    ```
