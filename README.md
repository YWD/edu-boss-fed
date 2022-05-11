# 说说 application/json 和 application/x-www-form-urlencoded 二者之间的区别。
## application/json
- 请求的参数格式是JSON字符串 如{"key1": "value1", "key2": "value2"}
- 请求内容不需要进行转译
## application/x-www-form-urlencoded
- 请求的参数格式是通用的URL参数格式 如 key1=value1&key2=value2
- 请求的参数内容如果有特殊符合 如 “?&%”等分隔符，需要进行转译，如果不转译就会发生参数阶截断
  如 key1=val&ue1&key2=value2

# 说一说在前端这块，角色管理你是如何设计的。
基于角色的访问控制方法（Role-Based Access Control,简称RBAC）是目前公认的解决大型企业的统一资源访问控制的有效方法
- 首先设计好资源管理，比如路由、菜单
- 添加角色
    - 定义角色名称、ID
    - 添加该角色可以访问的资源列表
    - 添加该角色的对应用户
- 角色列表展示，提供搜索、编辑、删除

# @vue/cli 跟 vue-cli 相比，@vue/cli 的优势在哪？
- 抽离cli service层，使构建更新更加简单
- 插件化，如果你要做深度的vue-cli定制化，不建议直接写在vue.config.js中，而是封装在插件中，独立的维护这个插件，然后项目再依赖这个插件。这样就可以简化升级的成本和复杂度。
- 提供GUI界面
- 快速原型开发，直接将一个vue文件跑起来，
- 现代模式，先进的浏览器配合先进的代码(ES6之后),同时兼容旧版本的浏览器，先进的代码不管从文件体积还是脚本解析效率都有较高的提升。

# 详细讲一讲生产环境下前端项目的自动化部署的流程。
Github Actions 自动构建前端项目并部署到服务器的流程
- 1、设置何时触发自动构建和部署
- 2、通过构建服务器执行这些过程：下载源码 -> 打包构建 -> 发布 Release -> 上传构建结果到 Release
- 3、将构建结果Release 下载到目标服务器，执行解压缩，安装依赖，运行项目

在你需要部署的项目下，建立一个 yml 文件，放在 .github/workflow 目录下。你可以命名为 main.yml，里面包含的是要自动执行的脚本代码。

这个 yml 文件的内容如下：
```yml
name: Publish And Deploy Demo
on: 
  push:
    tags:
      - 'v*' # 打tag，推送到github时，触发自动构建和部署

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:

    # 下载源码
    - name: Checkout
      uses: actions/checkout@master

    # 打包构建
    - name: Build
      uses: actions/setup-node@master
    - run: npm install
    - run: npm run build
    - run: tar -zcvf release.tgz .nuxt static nuxt.config.js package.json package-lock.json pm2.config.json

    # 发布 Release
    - name: Create Release
      id: create_release
      uses: actions/create-release@master
      env:
        GITHUB_TOKEN: ${{ secrets.TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false

    # 上传构建结果到 Release
    - name: Upload Release Asset
      id: upload-release-asset
      uses: actions/upload-release-asset@master
      env:
        GITHUB_TOKEN: ${{ secrets.TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }}
        asset_path: ./release.tgz
        asset_name: release.tgz
        asset_content_type: application/x-tgz

    # 部署到服务器
    - name: Deploy
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        password: ${{ secrets.PASSWORD }}
        port: ${{ secrets.PORT }}
        script: |
          cd /app/realworld-nuxtjs
          wget https://github.com/KlayHao/realworld-nuxtjs/releases/latest/download/release.tgz -O release.tgz
          tar zxvf release.tgz
          npm install --production
          pm2 reload pm2.config.json
```
创建一个 个人访问项目的 TOKEN
提交代码到github上， 在GitHub项目 依次点击 Settings -> Secrets -> new secrets
- 添加 TOKEN，将创建的个人访问token 填入
- 添加 服务器信息 HOST、PORT，登录的用户名和密码 USERNAME、PASSWORD


# 你在开发过程中，遇到过哪些问题，又是怎样解决的？请讲出两点。
## 1、在开发react-native项目中，产品提出了一个需求，需要全局使用某一种字体，包括第三方的UI组件的字体也需要修改？

解决过程：
- 1. 需要引入字体文件，android和ios都有特定的引入字体资源的方式
- 2. 需要修改可以使用字体的组件，每个文件单独去修改组件的样式，这种方式肯定不现实，首先项目自定义的组件非常多，其次第三方UI组件也不可能去修改源码，在思考这个问题的过程中，我发现所有可以使用字体的 react-native 组件就只有 Text 和TextInput, 而且第三方UI组件也是基于项目的react-native的这两个组件，我阅读了这两个组件的源码，做了如下修改
```javascript
// 项目根目录的index.js中修改
import React from 'react';
import {Text, TextInput} from 'react-native';

// Text组件是 React.forwardRef() 返回的函数组件，
const TRender = Text.render;
Text.render = function (props, ref) {
    return TRender({
        ...props,
        style: [
            {fontFamily: 'fontFamilyXXX'},
            props.style,
        ]
    }, ref)
}

// TextInput组件是普通的类组件，修改原型上的render 方法就可以做到全局修改
const textInputRender = TextInput.prototype.render
TextInput.prototype.render = function () {
    this.props = {
        ...this.props,
        style: [
            {fontFamily: 'fontFamilyXXX'},
            this.props.style,
        ]
    }
    return textInputRender.call(this)
}

```

## 2、在开发react-native 的项目中，出现了网络请求堵塞的情况
经过各种测试发现 react-native 的 fetch 方法存在这个问题，测试了axios这个库不会出现这个问题，所以将网络请求都换成axios

# 针对新技术，你是如何过渡到项目中？
1、需要对新技术有全面的了解，以便在遇到问题时能有解决的方案；
2、判断新技术是否适合项目具体的哪些功能模块，
3、项目中新的需求点，可以使用新技术开发，而稳定的旧的功能点，不应该随意去改动，而是要慎重，考虑影响范围，可以在未来需要重构时，使用新技术修改。
