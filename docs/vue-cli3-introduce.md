## 前言
 - vue cli3 发布已有段时间了，相比 vue-cli 2.X 创建的目录： vue-cli 3 创建的目录不见了 webpack 的配置，config目录也被移除，从而减少了维护成本。同时 webpack也升级到了4，性能有了很大提升（编译，打包速度很快）。最大的好处还是不需eject，通过升级 CLI service 和插件便可进行修复或更新配置。因此本次升级可谓一劳永逸。
 - 本篇内容主要介绍vue cli3整个升级过程以及升级中遇到的问题及解决方案（webpack-chain，加载器，插件方面）。希望对自己或者他人在后续升级时有些许的帮助。

**Tip:**  `Vue CLI 附带 vue inspect 命令可查看内部 Webpack 配置。对比老配置不断调整最终得出vue.config.js 配置`

## vue cli3 特性及变化
### vue.config.js
   - 这是一个**可选**配置文件，目的减少webpack 配置，让开发人员集中在业务代码中。
   - 如果你的项目是多入口或者要做 splitChunks优化或者有特殊的loader或plugin处理等情况则需要增加此文件完成你的配置。亦或是在package.json中增加vue的json配置。
   - **建议用vue.config.js形式进行配置。**

### Babel
 - 因为Vue CLI 使用了 Babel 7 中的新配置格式 ，所以项目需要通过** babel.config.js** 进行配置。.babelrc 或 package.json 中的 babel 字段配置不再生效。
 - 另外通过引入** "@vue/cli-plugin-babel"** devDependency （依赖了**"@vue/babel-preset-app"**它包含了 babel-preset-env、JSX 支持以及为最小化包体积优化过的配置），去除对**"transform-vue-jsx", "transform-runtime", "babel-loader"**的依赖。

### index 文件
 - 根目录下的index.html默认移入到 public/index.html中
 - 客户端环境变量可直接被使用，如BASE_URL：`<link rel="icon" href="<%= BASE_URL %>favicon.ico">
`
- 放置在 public 目录下的资源将会直接被拷贝，**而不会经过 webpack 的处理**。并通过通过 <%= BASE_URL %> 设置链接前缀。如：
``` html 
<script src="<%= BASE_URL %>es6-promise.auto.min.js"></script>
```
### 环境变量和模式
 - 模式：默认** 'development' 'production' 'test'** 三种模式。可通过 ** --mode** 选项参数覆盖默认的模式 如：`"build-off": "vue-cli-service build --mode off"` 为off 模式
 - 特殊环境变量 NODE_ENV：体的值取决于运行的模式。上述例子 NODE_ENV的值为 'off'
 - 特殊环境变量 BASE_URL： 等于 vue.config.js 中baseUrl 选项值（默认'/'），即应用部署的基础路径。
 - 自定义变量：**必须以VUE_APP_ 为开头定义变量**，只有这样才能在代码中访问如：VUE_APP_SDK_CONFIG在代码中可通过 process.env.VUE_APP_SDK_CONFIG 访问。
 - 自定义变量可在 .env 文件中以“键=值”对的形式进行定义。这样可在所有的环境中被载入进行访问。如果需要在在指定的模式下载入 需要以 `.env.[mode]`定义文件名。