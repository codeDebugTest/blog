# vue-cli3-plugin 模版插件开发指南

## 前言
在vue-cli 中我们可以通过定制自己的模版库，然后在vue init 命令中指定模版库从而按照自定义模版快速搭建新vue项目。在官方推出vue-cli3后，虽然减少了很多webpack配置。但是在我们的项目中难免会定制自己的vue.config.js、babel.config.js、eslintrc.js文件甚至环境配置文件env.xxx也会定义自己的项目环境变量。那么这时需要新建一个业务项目，这些配置拷贝过去？显然我们也可以像vue-cli 那样定义自己的vue-cli3 项目模版，避免重复操作，避免copy过程中可能出现的误操作。本篇将给你带来如何搭建自己的vue-cli3 项目模版，做到开箱即用。

## 设计
vue-cli3 有两种插件一种是cli插件，一种是service 插件。显然cli 插件符合我们的需求，我们的目标是 使用 **vue create xxx** 命令初始化一个前端项目时，可以从 git repo 去拉取项目初始化信息。  
在vue-cli3 官网中提到[远程preset](https://cli.vuejs.org/zh/guide/plugins-and-presets.html#%E8%BF%9C%E7%A8%8B-preset)
因此我们的项目结构如下：
> * preset.json: 包含 preset 数据的主要文件（必需）。
> * generator.js: 一个可以注入或是修改项目中文件的 Generator。 
> * prompts.js: 一个可以通过命令行对话为 generator 收集选项的 prompts 文件。

``` bash
# 从 GitHub repo 使用 preset: 使用username/repo, 不要指定 git clone 后的地址
vue create --preset username/repo my-project
```
``` bash
# 公司内部 repo 使用 preset: direct:url 确保使用 --clone 选项
vue create --preset direct:url my-project
```
repo 参数指定参考：[download-git-repo](https://github.com/flipxfx/download-git-repo)


### preset.json
preset 是一个包含创建新项目所需预定义选项和插件的 JSON 对象，**让用户无需在命令提示中选择它们**。  
在 vue create 过程中保存的 preset 会被放在你的 home 目录下的一个配置文件中 (~/.vuerc)。你可以通过直接编辑这个文件来调整、添加、删除保存好的 preset。preset示例：
``` json
{
    "useConfigFiles": true,
    "cssPreprocessor": "less",
    "plugins": {
        "@vue/cli-plugin-babel": {},
        "@vue/cli-plugin-eslint": {
            "config": "standard",
            "lintOn": ["save"]
        },
        "vue-cli-plugin-xxxx-tpl": {
            "prompts": true
        }
    },
    "router": true,
    "vuex": false
}
```
* 显示指定插件的版本：
``` json
{
    "plugins": {
        "@vue/cli-plugin-babel": {
            "version": "^3.5.0"
        }
    }
}
```
**对于官方插件来说这不是必须的，CLI自动使用 registry 中的最新版本**

* 允许插件的命令提示，保持灵活性  
插件在项目创建的过程中都可以注入它自己的命令提示，**不过当你使用了一个 preset，这些命令提示就会被跳过**，因为 Vue CLI 假设所有的插件选项都已经在 preset 中声明过了。  
   
**那怎样在 preset 只声明需要的插件，同时让用户选择插件注入的命令提示来保留一些灵活性呢？**
答案是在插件选项中指定 "prompts": true 来允许注入命令提示：
``` json
{
  "plugins": {
    "@vue/cli-plugin-eslint": {
      // 让用户选取他们自己的 ESLint config
      "prompts": true
    }
  }
}
```

### generator
generator主要用于向项目中注入或者修改项目中的文件。  
CLI 插件可以包含一个 generator.js 或 generator/index.js 文件。插件内的 generator 将会在两种场景下被调用：
> * 项目的初始化创建过程中，CLI 插件作为项目创建 preset 的一部分被安装。

> * 插件在项目创建好之后通过 vue invoke 独立调用时被安装。  
``` javascript
/**
 * generator 应该导出一个函数
 * @param {Object} api GeneratorAPI 实例
 * @param {Object} opts 插件prompts.js 解析后的答案作为选项被传递进来
*/
module.exports = (api, opts) => {
    // 扩展package.json 文件
    api.extendPackage({
        scripts: {},
        dependencies: {}
    });

    // 复制并用 ejs 渲染 `./template` 内所有的文件
    api.render('./template', opts); 

    // 上述template写入磁盘后，调用该回调函数
    api.onCreateComplete(callback);
}
```

vue-cli3 提供的核心Api: (可以查看vue-cli3源码GeneratorAPI.js)
#### extendPackage 
扩展项目中的 package.json。如：dependencies, scripts  

``` javascript
module.exports = (api, opts) => {
    api.extendPackage({
        scripts: {
            'dev': 'npm run serve',
            'build-dev': 'vue-cli-service build --mode development',
            'analyze': 'vue-cli-service build --report'
        },
        dependencies: {
            'axios': '^0.18.0',
            'lodash': '^4.17.11',
            'moment': '^2.24.0'
        },
        devDependencies: {
            'body-parser': '^1.18.3',
            'multiparty': '^4.2.1'
        }
    });
}
```

#### render
将模版中的文件用ejs 渲染，并拷贝到初始化的项目中。如：
``` javascript
/**
 * generator 应该导出一个函数
*/
module.exports = (api, opts) => {
    // ...
    // 复制并用 ejs 渲染 `./template` 内所有的文件
    api.render('./template', opts); 
}
```
1. **当你需要创建一个以 . 开头的文件时，模板项目中需要用 _ 替代**
![](https://github.com/codeDebugTest/blog/blob/master/img/fileName.png) 

2. **第二个参数是插件prompts.js 解析后，输入的答案作为选项被传递进来**

3. **模版使用EJS进行渲染处理**


### prompts
prompts.js 其实就是你在初始化项目时，系统会询问你的配置选项问题，比如你的项目需不需要安装 vuex? 需不需要安装 vue-router? 如：
```javascript
const getGitUserInfo = require('./lib/gitUserInfo');

module.exports = [
    {
        name: 'author',
        type: 'input',
        required: true,
        message: 'Author?',
        default: getGitUserInfo()
    },
    {
        name: 'replaceTemplates',
        type: 'confirm',
        message: 'Use custom templates? ',
        default: true
    }
];
```

## 本地调试与测试
是时候验证下插件的效果了。那么怎样在插件发布前，测试插件的正确性呢？
1. 创建一个新vue 项目
```bash
$ vue create my-app
```

2. 安装插件
```bash
$ npm install --save-dev file:/user/home/xxx/xxx/your/plugin
```

3. invoke plugin
```bash
$ vue invoke vue-cli-plugin-<YOUR-PLUGIN-NAME>
```

## 参考栗子
1. [vuetifyjs](https://github.com/vuetifyjs/vue-cli-plugin-vuetify)
2. [custom-tpl](https://github.com/natee/vue-cli-plugin-custom-tpl)
2. [vue cli 3.0 自定义 template](https://github.com/vuejs/vue-cli/issues/2400)
3. [How to build a Vue CLI plugin](https://dev.to/vuevixens/how-to-build-a-vue-cli-plugin-3b6b)
