### 常见配置文件
```
{
  // 项目根目录(index.html文件所在的位置)
  root: 默认值(process.cwd()) || 绝对路径名(/foo/) || 空字符串或./,
  // 公共基础路径(项目的部署路径)
  // 一般会对相对路径的资源引用进行修改
  // 生产环境下需要配置对base的映射，以便找到资源
  base: 默认值(/) || 绝对路径名(/foo/) || 完整的URL(https://foo.com/) || 空字符串或./,
  // 在构建时定义全局常量，将代码中的特定标识符替换成指定的值。
  // 替换发生在构建时，因此替换的值在开发时不可见。
  // 作用范围: .js/.ts/.vue/.jsx/.tsx 中的脚本代码
  // 如果需要放在dom中，需先自定义变量存储
  define: {
    ALLOWED_ORIGINS: JSON.stringify(['https://example.com', 'https://test.com'])，
    TIMESTAMP: 'Date.now()',
    RANDOM: 'Math.random()',
    myFunc: '() => console.log("hello")',
    NULL_VALUE: null,
    UNDEFINED_VALUE: 'undefined',
    CONFIG: JSON.stringify({ a: 1 }),
    COUNT: 10,  
    ENABLE_DEBUG: true,
  },
  // 配置模块的解析行为，它会影响如何查找和处理模块。
  resolve: {
    // 模块的路径别名，一定要设置为绝对路径，否则可能找不到模块。
    alias: {
      '@': path.resolve(__dirname, 'src'), // 使用@代替src目录
      '@components': path.resolve(__dirname, 'src/components'), // 使用@components代替src/components目录
      '@utils': path.resolve(__dirname, 'src/utils') // 使用@utils代替src/utils目录
    },
    // 配置导入时可以省略的扩展名。
    extensions: ['.js', '.ts', '.jsx', '.tsx', '.json', '.vue'], 
    // 配置需要去重的依赖包。
    dedupe: ['vue'],
  },
  // 开发服务器配置
  server: {
    // 指定服务器的主机名，默认值是localhost。
    host: '0.0.0.0',
    // 指定服务器监听的端口号，默认值是3000。
    port: 3000,
    // 是否自动打开浏览器，默认值是false。
    open: false,
    // 跨域代理（企业级必备）
    proxy: {
      // 将/api请求代理到目标服务器env.VITE_API_BASE_URL
      '/api': {
        // 目标服务器地址
        target: env.VITE_API_BASE_URL,
        // 改变请求源，修改请求头中的 Host 字段，使其与目标服务器匹配
        changeOrigin: true,
        // 重写路径
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      // 启用 WebSocket 代理
      ws: true,  
    },
    // 用于在开发服务器启动时预加载一些模块，从而优化首次访问时的加载速度，针对的是业务代码的优化
    warmup: {
      // 指定需要预热的客户端文件
      clientFiles: [
        './src/components/CommonComponent.vue',
        './src/utils/helpers.js',
        './src/store/index.js'
      ]
    }
  },
  // 插件配置
  plugins: [vue()],
  // CSS 配置
  css: {
    // 配置 CSS 预处理器
    preprocessorOptions: {
      scss: {
        // 在每个 .scss 文件开头自动注入
        additionalData: `@import "@/styles/variables.scss"; @import "@/styles/mixins.scss";`
      },
      less: {
        // 修改自定义变量的值
        modifyVars: {
          'primary-color': '#1DA57A', // 修改主题色
        },
        // 启用 JS 特性
        javascriptEnabled: true,
      },
    },
    // 自定义 PostCSS 的处理行为,可以通过插件实现自动添加浏览器前缀、CSS 压缩、未来 CSS 语法支持等功能。
    postcss: {
      plugins: [
        // 自动添加浏览器前缀
        autoprefixer({
          overrideBrowserslist: ['> 1%', 'last 2 versions'] // 指定浏览器范围
        }),
        // 其他 PostCSS 插件...
      ],
    },
    // 是否生成 CSS source map。
    devSourcemap: true 
  },
  // 用于限制 Vite 开发服务器可以访问哪些目录和文件。(monorepo项目)
  fs: {
    // 启用或禁用对特定目录的访问。
    strict: true(禁止) | false(启用),
    // 允许访问的目录列表。
    allow: [
      // 允许访问项目根目录, searchForWorkspaceRoot函数会自动查找 monorepo 项目的根目录
      searchForWorkspaceRoot(process.cwd()),
      // 允许访问 src 目录
      path.resolve(__dirname, 'src'),
    ],
    // 配置禁止访问的文件或目录。
    deny: [
      // 禁止访问.env文件
      '**/.env*',
    ]
  },
  // 自定义依赖预构建行为，它可以将第三方依赖转换为 ESM 格式并进行缓存（加速启动）
  optimizeDeps: {
    // 显式指定需要预构建的依赖。
    include: ['vue', 'vue-router', 'pinia'],
    // 指定不需要预构建的依赖。
    exclude: [ 'your-local-package', 'some-esm-only-package' ],
    // 是否强制重新预构建依赖。
    force: true | false
  },
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
    // 是否生成manifest.json 文件（后端集成必备）
    manifest: true,
    // 是否启用压缩或指定压缩方式
    minify: boolean | 'terser' | 'esbuild'(默认) 
    // 是否生成sourcemap（安全）
    sourcemap: false,
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
          // 配置代码分割生成的 chunk 文件命名
          chunkFileNames: "assets/js/[name]-[hash].js",
          配置入口文件的命名
          entryFileNames: "assets/js/[name]-[hash].js",
          // UI 库：element-plus 单独打包
          'ui-vendor': ['element-plus'],
          // 工具库：lodash、axios 单独打包
          'utils-vendor': ['lodash-es', 'axios'],
          // 自定义输出静态文件名
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

### 补充
#### 删除console.log的方法
通过esbuild
```
export default defineConfig({
  build: {
    minify: 'esbuild',
    drop: ['console', 'debugger']
  }
})

```
通过terser
```
export default defineConfig({
  build: {
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true
      }
    }
  }
})
```
选择性删除
```
export default defineConfig({
  build: {
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: false,  // 不完全删除 console
        pure_funcs: [
          'console.log',     // 只删除 log
          'console.info',    // 只删除 info
          'console.debug'    // 只删除 debug
        ]
      }
    }
  }
})

```
#### vite --debug transform
查看控制台日志，能看到转换耗时久的文件