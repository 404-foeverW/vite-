### 配置文件
```
{
    // 公共基础路径
    base: 默认值(/) || 绝对路径名(/foo/) || 完整的URL(https://foo.com/) || 空字符串或./,

}
```

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
### 