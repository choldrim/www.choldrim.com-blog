title: (译)ES6环境搭建简单入门
date: 2016-04-22
tags: [javascript, ES6]
category: javascript

---

本文译自：http://ccoenraets.github.io/es6-tutorial/ecmascript6-setup-babel.html

## 1、配置一个babel项目
当前浏览器还没完全支持所有的ECMAScript6（aka ECMAScript 2015）新特性（[这里](http://kangax.github.io/compat-table/es6/)查看各浏览器对ES6支持情况对比）。因此，你需要使用一个编译器（转译器），将你的ES6代码转换成浏览器支持的ES5。有很多工具可以做这个转换工作，[Babel](http://babeljs.io/)实际上已经是一个ES6转换的标准工具了。Bebel还可以可以编译其他版本的ECMAScript，如：React'sJSX，不过这不在这篇教程的范围了。^.^

通过这篇文章，你将了解到如何使用Babel配置ES6基础开发环境和运行ES6程序。

### 步骤 1：先拉取一个demo程序
1. 拉取[es6-tutorial](https://github.com/ccoenraets/es6-tutorial/)项目，这个项目包含一个ES5版本的抵押贷款程序
 ```shell
 git clone https://github.com/ccoenraets/es6-tutorial
 ```
 (如果你不想克隆整个项目，也可以到项目主页下载master.zip解压使用)

2. 进入项目目录，使用浏览器打开`index.html`，点击**Calculate**按钮可以计算抵押扣款（译者：你看懂这个程序了吗？其实我没懂...但他计算啥不重要）。
 ![index.html](http://ccoenraets.github.io/es6-tutorial/images/calc-file.jpg)

### 步骤 2：配置Babel
上一步骤的页面中可以看到，不需要任何转换器也能正常运行，因为页面的ES脚本是使用ES5编写的。在这个步骤中，我们将使用Babel去配置一个能让浏览器“支持”ES6特性的环境。
1. 打开终端，(cd)进入es6-tutorial目录
2. 输入一下命令创建一个package.json文件
 ```shell
 npm init
 ```
 npm将会询问你项目名等等其他问题，全部使用默认，一路`enter`到底就行了
3. 输入一下命令安装babel-cli和babel-core模块
 ```shell
 npm install babel-cli babel-core --save-dev
 ```
 当然，安装Babel方式多多，也可以参考Babel官方文档的其他安装方式，将Bebel安装到全局，直接使用命令行接口调用Babel）
4. 输入一下命令安装 **ECMAScript 2015 preset**：
 ```shell
 npm install babel-preset-es2015 --save-dev
 ```
 在Babel6中，每一个转换器都是可以独立安装的模块。preset是一组插件集合，直接安装preset可以免去将几十个插件单独安装之苦

5. 在你的项目中安装 [http-server](https://github.com/indexzero/http-server)，**http-server**是一个很轻量级的web服务器，在开发过程中，我们可以很轻松的使用它来提供web运行环境
 ```shell
 npm install http-server --save-dev
 ```
6. 使用你熟悉的编辑器打开`packages.json`。在`scripts`部分，删除 `test`部分的script，并添加两个新的scripts：一个是 **babel**，用于编译生成ES5版本的main.js；另一个脚本是**start**，用于启动本地web server。修改后`scripts`部分类似于：
 ```json
 "scripts": {
     "babel": "babel --presets es2015 js/main.js -o build/main.bundle.js",
     "start": "http-server"
 }
 ```

7. 在`es6-tutorial`目录下创建`build`目录用于生成编译文件
 ```shell
 mkdir build
 ```

### 步骤3：编译和运行
1. 在终端下进入`es6-tutorial`目录，输入一下命令运行**babel**编译main.js：
 ```shell
npm run babel
 ```

2. 使用编辑器打开**index.html**，如下修改`<script>`标签，然其编译后的js脚本`<build/main.bundle.js>`。（`<build/main.bundle.js>`是<js/main.js>编译后的版本）
 ```html
 <script src="build/main.bundle.js"></script>
 ```

3. 打开一个新的终端（或新的终端标签页），同样是在`es6-tutorial`目录，输入一下命令，启动**web-server**（其实就是启动我们上面所安装的http-server）：
 ```shell
npm start
 ```
 默认使用的是8080端口，如果你机器上的8080端口被占用了，可以修改`package.json`的`start`部分，重新指定一个端口：
 ```json
"scripts": {
    "babel": "babel --presets es2015 js/main.js -o build/main.bundle.js",
    "start": "http-server -p 9000"
}
 ```

4. 打开浏览器，输入：`[http://localhost:8080](http://localhost:8080)`

5. 点击 **Calculate**按钮测试抵押贷款计算
![Calculate](http://ccoenraets.github.io/es6-tutorial/images/calc-http.jpg)

6. 使用编辑器打开`build/main.bundle.js`，可以发现生成的代码和编译源文件`js/main.js`几乎是一样的，是因为源文件中没有包含ECMAScript6的新特性。你可以加入一些ES6的特有语法进去，然后重复以上的编译过程试试^_^

 **good luck**

### 文章中引入的资源参考
- [Babel](http://babeljs.io/)
- [ES6 compatibility table](https://kangax.github.io/compat-table/es6/)
- [http-server repo](https://github.com/indexzero/http-server)




