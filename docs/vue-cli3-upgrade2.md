# vue老项目升级vue-cli3指南（三）：vue cli3打包优化

## webpack-bundle-analyzer
工欲善其事，必先利其器。webpack有个插件，可以查看项目打包，每个包的体积，每个包里面的包一些情况: webpack-bundle-analyzer，借助此插件可进行打包分析。我们先看下怎样引入此插件。

### webpack配置(未使用 vue-cli3):
未使用vue-cli3时我们可以进行如下配置：
- package.json 
```javascript
    // ...
    "analyze": "NODE_ENV=production npm_config_report=true npm run build"
    // ...
    "devDependencies": {
        // ...
        "webpack-bundle-analyzer": "^3.0.3"
    }
```

- config/index.js 
```javascript
    // Run the build command with an extra argument to
    // View the bundle analyzer report after build finishes:
    // `npm run build --report`
    // Set to `true` or `false` to always turn it on or off
    bundleAnalyzerReport: process.env.npm_config_report
```

- webpack.prod.conf.js 
```javascript
    if (config.common.bundleAnalyzerReport) {
        let BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
        webpackConfig.plugins.push(new BundleAnalyzerPlugin());
    }
```

### vue-cli3 配置
**需说明的我们不需要在package.json devDependencies 中添加webpack-bundle-analyzer的依赖配置。vue-cli-service 中包含了webpack-bundle-analyzer依赖。**
在vue-cli3中我们可以在vue.config.js 中通过env环境变量来判断是否开启 analyze

- vue.config.js
```javascript
    // bundle analyze
    config.when(process.env.OPEN_ANALYZE, config => {
        config.plugin('BundleAnalyzerPlugin')
            .use(require('webpack-bundle-analyzer').BundleAnalyzerPlugin);
    });
```
然后在package.json 中增加analyze 指令
```javascript
    "analyze": "vue-cli-service build --mode analyze"
```
[结合前一篇文章](https://github.com/codeDebugTest/blog/blob/master/docs/vue-cli3-upgrade.md)我们需要增加annalyze env文件配置。新增env.analyze文件，内容如下：
```javascript
NODE_ENV=production
OPEN_ANALYZE=true
```
再次运行 npm run analyze 就可以查看项目打包状况：
![](https://github.com/codeDebugTest/blog/blob/master/img/bundle-analyze.png)


## Moment.js
通过打包分析工具会发现最终打包的文件中将 Moment.js 的全部语言包都打包了，导致最终文件徒然增加 100+kB。为了去除不必要引入的语言包，有两个webpack插件可以使用：
1. IgnorePlugin
2. ContextReplacementPlugin

### IgnorePlugin
在vue.config.js中进行如下配置，将移除掉所有语言包
```javascript
configureWebpack: {
    // ...
    plugins: [
        // Ignore all locale files of moment.js
        new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/)
    ]
}
```
改方案也被使用在[create-react-app](https://github.com/facebook/create-react-app/blob/a0030fcf2df5387577ced165198f1f0264022fbd/packages/react-scripts/config/webpack.config.prod.js#L350-L355)中

### ContextReplacementPlugin
如果想保留某个语言包，可以使用ContextReplacementPlugin 在vue.config.js中进行如下配置：
```javascript
const webpack = require('webpack');
// ...
configureWebpack: {
    // ...
    plugins: [
        // load `moment/locale/zh-cn.js`
        new webpack.ContextReplacementPlugin(/moment[\\/]locale$/, /zh-cn/)
    ]
}
```

### 你可能不需要 Moment.js
如果你的项目中没有用到 **时区**，仅仅用了一些简单的函数，这会导致项目引入了很多没有实用的方法。即使你移除了语言包，最终打包体积也有50K+的大小
![](https://github.com/codeDebugTest/blog/blob/master/img/MomentJs.png)   
这里推荐两个库
1. dayjs 
    - 体积小
    - 与Moment.js 具有非常相似的API，因此很容易从moment 平滑过渡到day.js
2. date-fns 
    - API 很简单
    - Webpack 完美的伴侣，可以使用`Tree-shaking`代码优化技术

#### 比较
![](https://github.com/codeDebugTest/blog/blob/master/img/dateCompare.png)


| Name                                     | Size(gzip)                        | Tree-shaking | Popularity(stars) | Methods richness | Pattern    | Timezone Support      | Locale |
| ---------------------------------------- | --------------------------------- | ------------ | ----------------- | ---------------- | ---------- | --------------------- | ------ |
| [Moment.js](https://momentjs.com/)       | 329K(69.6K)                       | No           | 39k               | High             | OO         | Good(moment-timezone) | 123    |
| [date-fns](https://date-fns.org)         | 78.4k(13.4k) without tree-shaking | Yes          | 15k               | High             | Functional | Not yet               | 50     |
| [dayjs](https://github.com/iamkun/dayjs) | 6.5k(2.6k) without plugins        | No           | 17k               | Medium           | OO         | Not yet               | 39     |

#### References
1. 你不需要Moment.js [https://github.com/you-dont-need/You-Dont-Need-Momentjs.](https://github.com/you-dont-need/You-Dont-Need-Momentjs.) 
2. 移除语言包
 - [http://stackoverflow.com/questions/25384360/how-to-prevent-moment-js-from-loading-locales-with-webpack/37172595](http://stackoverflow.com/questions/25384360/how-to-prevent-moment-js-from-loading-locales-with-webpack/37172595)
 - [https://github.com/moment/moment/issues/2373](https://github.com/moment/moment/issues/2373)