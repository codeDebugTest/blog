# vue老项目升级vue-cli3指南（二）：vue cli3升级手册

## 升级步骤
### 1. 初始化新项目 
笔者先用vue cli3 初始化一个新项目（vue create xxxx）, 具体过程参考 [vue 创建一个项目](https://cli.vuejs.org/zh/guide/creating-a-project.html#vue-create)。
配置过程：
1. 选取 <手动选择特性>配置
2. 选取 <babel>,<Linter / Formatter>
3. 选取 <ESLint + standard> 标准配置
4. 选取 <Lint on save> 保存时 Eslint检查代码

初始化后得到 package.json依赖配置文件（主要使用devDependencies配置）, babel.config.js 配置文件。这其中的配置基本上可以完全使用到我们的老项目中。从而减少了自主配置过程。

#### 1.1 devDependencies 变化:
新配置：
``` javascript
"devDependencies": {
    "@babel/core": "^7.0.0",
    "@vue/cli-plugin-babel": "^3.2.0",
    "@vue/cli-plugin-eslint": "^3.2.0",
    "@vue/cli-service": "^3.2.0",
    "@vue/eslint-config-standard": "^4.0.0",
    "babel-eslint": "^10.0.1",
    "babel-plugin-lodash": "^3.3.4",
    "babel-plugin-veui": "^1.0.0-alpha.21",
    "body-parser": "^1.18.3",
    "eslint": "^5.8.0",
    "eslint-plugin-vue": "^5.0.0",
    "eslint-plugin-html": "^5.0.0",
    "less": "^3.0.4",
    "less-loader": "^4.1.0",
    "less-plugin-est": "^2.1.0",
    "vue-template-compiler": "^2.5.21"
}
```
是不是有种清爽的感觉，对比老项目发现少了令人恶心的babel依赖，因为babel-loader babel-eslint, babel-plugin-transform-vue-jsx手动配置起来那简直。。。。。。 

#### 1.2 babel.config.js: 
``` javascript
module.exports = {
    'presets': [
        '@vue/app'
    ]
};
```
对比之前的 presets, plugins 配置真良心啊！！！

### 2. vue-config.js
  这是一个可选的配置文件，你可以通过此文件修改全局的配置。并可通过vue inspect 命令审查项目webpack config.
#### 2.1 多入口
在 `pages`里配置，build打包后会产出 common.xxx.js, vendor.xxx.js, [entryName].xxx.js 几个文件，默认都会引用这几个文件。
**可通过明确配置chunks，优化入口的引用文件** 如：
```javascript
pages: {
    home: {
        entry: 'src/home/main.js',
        template: 'entry/home.html',
        filename: 'home/index.html',
        chunks: ['chunk-vendors', 'home']   // home.html 只会引用 chunk-vendors.xxx.js, home.xxx.js, 并不会引用 common.xxx.js
    },
    client: {
        entry: 'src/client/main.js',
        template: 'entry/client.html',
        filename: 'client/index.html'
    }
}
```
#### 2.2 transpileDependencies
默认情况下 babel-loader 会忽略所有 node_modules 中的文件。如果你想要通过 Babel 显式转译一个依赖，可以在这个选项中列出来。如：
``` javascript
    transpileDependencies: ['vue-awesome', 'vue-void']
```
#### 2.3 css.loaderOptions 
向 CSS 相关的 loader 传递选项。如：
``` javascript
css: {
    loaderOptions: {
        less: {
            javascriptEnabled: true,
            modifyVars: {
                '@default-base-font-family': 'Helvetica Neue, Arial, PingFang SC, STHeiti, Microsoft YaHei, SimHei, sans-serif',
                '@default-base-heading-family': 'Helvetica Neue, Arial, PingFang SC, STHeiti, Microsoft YaHei, SimHei, sans-serif'
            }
        }
    }
}
```
#### 2.4 webpack:
Vue cli 中有默认配置，一般来说通过 configureWebpack 修改配置够用了，该配置最终会被webpack-merge合并入最终配置。但对于复杂的，精细粒度的配置则需要通过 chainWebpack 进行配置。[webpack-chain 参考配置](https://github.com/neutrinojs/webpack-chain)
- 条件配置：
``` javascript
// Example: Only add minify plugin during production
config
    .when(process.env.NODE_ENV === 'production', config => {
        config
        .plugin('minify')
        .use(BabiliWebpackPlugin);
    });

// Example: Only add minify plugin during production,
// otherwise set devtool to source-map
config
    .when(process.env.NODE_ENV === 'production',
        config => config.plugin('minify').use(BabiliWebpackPlugin),
        config => config.devtool('source-map')
    );
```
- Config module rules:
1. 新增
``` javascript
// Example
config.module
.rule('compile')
    .use('babel')
    .loader('babel-loader')
    .options({ presets: ['@babel/preset-env'] });
``` 
2. 修改 option
``` javascript
config.module
.rule('compile')
    .use('babel')
    .tap(options => merge(options, {
        plugins: ['@babel/plugin-proposal-class-properties']
    }));
``` 
3. 条件规则 oneOfs
``` javascript
config.module
.rule('css')
    .oneOf('inline')
    .resourceQuery(/inline/)
    .use('url')
        .loader('url-loader')
        .end()
    .end()
    .oneOf('external')
    .resourceQuery(/external/)
    .use('file')
        .loader('file-loader')
``` 
- 插件配置:
1. 新增
``` javascript
config.plugin('hot')
    .use(webpack.HotModuleReplacementPlugin);
```
2. 修改参数
``` javascript
// Example
config.plugin('env')
.tap(args => [...args, 'SECRET_KEY']);
```
    
## 踩坑问题
### 1. less inline javascript
**问题：Inline JavaScript is not enabled. Is it set in your options?**
**原因：** less3.x以后的版本需要增加 **javascriptEnabled: true**
**解决方案：**
``` javascript
    css: {
        loaderOptions: {
            less: {
                javascriptEnabled: true
            }
        }
    }
```


### 2. incorrect peerDependencies for vue-template-compiler
**问题：** Error: [vue-loader] vue-template-compiler must be installed as a peer dependency, or a compatible compiler implementation must be passed via options.

但是 vue-template-complier, vue-loader 都是已经安装了啊。并且vue-cli3 默认配置里面(config.module.rule)已经进行了如下配置：
``` javascript
    {
        test: /\.vue$/,
        use: [
          {
            loader: 'cache-loader',
            options: {
              cacheDirectory: 'xxx/node_modules/.cache/vue-loader',
              cacheIdentifier: 'ab103f44'
            }
          },
          {
            loader: 'vue-loader',
            options: {
              compilerOptions: {
                preserveWhitespace: false
              },
              cacheDirectory: 'xxx/node_modules/.cache/vue-loader',
              cacheIdentifier: 'ab103f44'
            }
          }
        ]
      }
```
**原因：**经过在vue-loader github仓库发现有人提到了这个问题。并且尤大进行了解释：
***You don't use it directly. However, its version must be the same with vue to ensure correct behavior. Making it a peer dep makes it possible to explicitly pin both vue and vue-template-compiler to the same version.*** 意思说 **vue 与 vue-template-complier 版本号不一致** [issue 详情](https://github.com/vuejs/vue-loader/issues/560)
确实发现项目依赖vue 与 vue-template-complier 版本号不一致。保持一致问题就解决了。

### 3. Parsing error: Adjacent JSX elements must be wrapped in an enclosing tag
**原因：** 该问题是由于 eslint 配置："parser": "babel-eslint" 的位置不对。把此配置移入到 “parserOptions” 中即可。
``` javascript
  {
      "root": true,
-     "parser": "babel-eslint",
+     "parserOptions": {
+         "parser": "babel-eslint"
+     },
```

### 4. svg 文件始终会以URL的形式引入到项目中。
需求 svg-inline-loader, svg-loader 分别对不同路径下的svg进行处理。
最开始的配置是：
``` javascript
configureWebpack: {
    module: {
        rules: [
            {
                test: /\.svg$/,
                loader: 'svg-inline-loader',
                include: [
                    resolve('src/common/img/icons')
                ]
            },
            {
                test: /\.svg$/,
                loader: 'svgo-loader',
                include: [
                    resolve('static/img')
                ]
            }
        ]
    }
}
```
**原因**：
通过vue inspect 发现原来有个vue cli 有个**默认svg配置**：
``` javascript
/* config.module.rule('svg') */
{
    test: /\.(svg)(\?.*)?$/,
    use: [
        /* config.module.rule('svg').use('file-loader') */
        {
            loader: 'file-loader',
            options: {
            name: 'img/[name].[hash:8].[ext]'
            }
        }
    ]
}
```
**解决方案一**：
通过 chainWebpack 删除原有配置
``` javascript
chainWebpack: config => {
    config.module.rules.delete('svg');
}
```
结合上面的configureWebpack配置,满足需求。

**解决方案二**：
通过 chainWebpack 覆盖原有svg配置
``` javascript
chainWebpack: config => {
    config.module
        .rule('svg')
        .include
        .add(resolve('src/common/img/icon'))
        .end()
        .use('file-loader')
        .loader('svg-inline-loader')
    }
}
```

### 5. webpack optimization split chunk
由于原项目最后打包体积过大（2M左右），因此在webpack2 使用了CommonsChunkPlugin 进行了代码分离。分离后的代码块不会变动可以被浏览器进行缓存。但是在webpack4 中该插件被移入到 optimization.splitChunks 中。默认状态下webpack4会基于以下条件，自动进行代码分离：
- 分离的块可以被共享或者来自node_modules的模块
- 分离的块必须大于 30Kb（minimize, gzip压缩前）
- 根据需要加载块时的最大并行请求数将小于或等于5
- 初始页面加载时的最大并行请求数将小于或等于3
> **为了满足最后两个条件，webpack有可能受限于包的最大数量值，生成的代码体积往上增加。**

#### 配置
- minSize(默认是30000， 30kb)：形成一个新代码块最小的体积
- minChunks（默认是1）：在分割之前，这个代码块最小应该被引用的次数（译注：保证代码块复用性，默认配置的策略是不需要多次引用也可以被分割）
- maxInitialRequests（默认是3）：一个入口最大的并行请求数
- maxAsyncRequests（默认是5）：按需加载时候最大的并行请求数。
- chunks的值应该是[all, async, initial]其中的一个。
- **缓存组（Cache Group）**
cacheGroup是最关键的配置上面那些参数可以不用管。
> cacheGroup 默认模式会将所有来自node_modules的模块分配到一个叫vendors的缓存组；所有重复引用至少两次的代码，会被分配到default的缓存组。一个模块可以被分配到多个缓存组，优化策略会将模块分配至跟高优先级别（priority）的缓存组，或者会分配至可以形成更大体积代码块的组里。


### optimization split chunk 配置
> 原始配置webpack2 如下：
``` javascript
new HtmlWebpackPlugin({
    filename: 'index.html',
    template: 'index.html',
    inject: true,
    minify: {
        removeComments: true,
        collapseWhitespace: true,
        removeAttributeQuotes: true
    },
    chunks: ['vendor', 'quill', `app`],
    // necessary to consistently work with multiple chunks via CommonsChunkPlugin
    chunksSortMode: 'dependency'
})),
new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor',
    minChunks: function (module, count) {
        // any required modules inside node_modules are extracted to vendor
        return (
            module.resource &&
            /\.js$/.test(module.resource) &&
            module.resource.indexOf(
                path.join(__dirname, '../node_modules')
            ) === 0 &&
            module.resource.indexOf('quill') === -1 
        );
    }
}),
new webpack.optimize.CommonsChunkPlugin({
    name: 'quill',
    chunks: config.entries.map(({name}) => name)
}),
```

> 最终配置： vue-cli3(webpack4)
``` javascript
// split chunks optimization when not 'development' mode
config.when(process.env.NODE_ENV !== 'development', config => {
    config.plugin('html')
        .use(HtmlWebpackPlugin)
        .tap(args => [{
            filename: 'index.html',
            template: 'public/index.html',
            inject: true,
            minify: {
                removeComments: true,
                collapseWhitespace: true,
                removeAttributeQuotes: true
            },
            chunks: ['vendor', 'quill', 'app'],
            // necessary to consistently work with multiple chunks via CommonsChunkPlugin
            chunksSortMode: 'dependency'
        }]);
    // split chunks
    config.optimization
        .splitChunks({
            cacheGroups: {
                default: false,
                vendor: {
                    chunks: 'initial',
                    name: 'vendor',
                    test(module, count) {
                        // any required modules inside node_modules are extracted to vendor
                        return (
                            module.resource
                            && /\.js$/.test(module.resource)
                            && module.resource.indexOf(
                                path.join(__dirname, './node_modules')
                            ) === 0
                            && module.resource.indexOf('quill') === -1
                            && module.resource.indexOf('lego-sdk') === -1
                        );
                    }
                },
                quill: {
                    chunks: 'initial',
                    name: 'quill',
                    test(module, count) {
                        return (
                            module.resource
                            && module.resource.indexOf(
                                path.join(__dirname, './node_modules')
                            ) === 0
                            && module.resource.indexOf('quill') !== -1
                        );
                    }
                }
            }
        });
});
```

//TODO:
### 6. dev server 