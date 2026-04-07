### 配置文件
```
{
    // 公共基础路径(项目的部署路径)
    base: 默认值(/) || 绝对路径名(/foo/) || 完整的URL(https://foo.com/) || 空字符串或./,
    // 构建配置
    build: {
      outDir: './dist', // 指定输出目录
      // 监听源码变化，一旦文件修改，自动重新执行生产构建。
      // 开启方式还有 vite build --watch
      watch: {
        // 仅监听 src 目录源码
        include: ['src/**'],
        // 忽略 node_modules、dist 目录的变化
        ignored: ['**/node_modules/**', '**/dist/**']
      },
      // 排除不打包的文件
      external: ['vue', 'vue-router', 'pinia'],
      // 打包配置
      rollupOptions: {
        // 设置入口文件，可为多个
        input: {
          main: resolve(__dirname, 'index.html'),
          nested: resolve(__dirname, 'nested/index.html')
        }
        // 设置输出文件
        output: {
          // 把不同模块手动拆分到不同的 chunk（代码块）中，实现按需加载、减少首屏体积。
          manualChunks: {
            // UI 库：element-plus 单独打包
            'ui-vendor': ['element-plus'],
            // 工具库：lodash、axios 单独打包
            'utils-vendor': ['lodash-es', 'axios'],
            // 自定义输出文件名
            assetFileNames: (assetInfo) => {
              // 按扩展名分类存放
              const ext = assetInfo.name.split('.').pop()
              if (['png', 'jpg', 'jpeg', 'gif', 'svg'].includes(ext)) {
                return 'images/[name]-[hash].[ext]'
              }
              if (['css'].includes(ext)) {
                return 'css/[name]-[hash].[ext]'
              }
              if (['woff', 'woff2', 'eot', 'ttf', 'otf'].includes(ext)) {
                return 'fonts/[name]-[hash].[ext]'
              }
              return 'assets/[name]-[hash].[ext]'
            }
          }
        },
        // Rollup生态的插件
        plugins : []
      }
    }
}
```