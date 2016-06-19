title: es6+angularjs+sass+gulp+browserify等组合的前端开发环境搭建
date: 2016-06-18
tags: [javascript, es6, sass, web-frontend, angularjs, gulp, browserify]
category: javascript

---

![es6 angular gulp](http://7xrkyd.com1.z0.glb.clouddn.com/simple-web-framework/fp3Y9c4lxcPDmFXxn59RA.jpeg)

其实我一直是在基础设施和运维周边转，只是前段时间，因为公司前端都去忙项目去了，没空管基础设施的平台开发，另外自己也懂一些html知识嘛，所以，“顶硬上”吧，顺便也接触一些新东西。

以前了解的web前端知识基本上就知道div+css，懂一些bootstrap的使用，其余一概不知了。

有一天，发现了一个朋友写的前端代码后，只在web前端边缘“裸奔”的我被惊呆了...

第一次见到html标签和属性还可以直接嵌入变量，前端还可以像后端web框架一样实现路由...

第一次看到css还可以调用函数，声明变量，条件判断...

我写这篇文章的触发点完全是因为，这些新奇的东西让常年驻扎在后端的我震惊了。 但是这些新奇的东西搭建起来确花费了我不少时间，一是我对前端了解不多，二是前端框架琳琅满目，网上的意见也很不统一。因此我想写一篇将这些组件整合起来的博客，并使用一个demo项目给大家参考，希望能让大家节省一些纠结的时间。

### es6

es6, ECMAScript 6是JavaScript语言的下一代标准，在2015年6月正式发布了。距离现在1年多了，但是主流的几款浏览器还没完全支持es6的语法，所以在编写完代码后需要将其转换成浏览器所支持的es5代码。es6教程推荐阮一峰大神的 [ES6标准入门](http://es6.ruanyifeng.com) 一书，书本写的很严谨，权威。不过搭建部分个人觉得写的有些略复杂了，es6环境搭建可以参考我翻译的一篇文章 [(译)ES6环境搭建简单入门](http://www.choldrim.com/2016/04/21/es6-tutorial-trs/)。

说说es6给我的感觉吧，让我觉得最大的区别是原生支持使用`import..from..`从其他文件导入模块和肥箭头匿名函数。另外原生支持class，但是并没有其他语言的class那么强，支持了静态函数，但是却不支持静态属性-.-|(静态属性已经在es7的提案中)。还有从语法的简洁度来说，（个人觉得）还不如coffeescript，搞不懂为什么还非得要使用这么长的`function`关键字...

### angularjs
我没用过其他框架，所以对比的东西我也说不了多少，让我觉得比起“裸奔”html最大区别是，我可以在html标签的属性或内容嵌入js数据，并且这些数据在后台的更新时会自动更新到前台，让静态html也可以动态起来。这一功能在对于使用restful的web架构来说，极为方便。

### sass
Sass 是对 CSS 的扩展，它允许你使用变量、嵌套规则、 导入等功能， 并且完全兼容 CSS 语法。可以让你的代码复用度上提高了不少。

语法可以参考阮一峰写的 [SASS用法指南](http://www.ruanyifeng.com/blog/2012/06/sass.html)，内容写的不多，比较精简，看一遍就基本入门了。

### gulp

gulp是一个构建工具，类似于makefile，将一系列构建步骤写成一个文件，需要构建的时候执行调构建命令就可以了。

gulp任务的核心是管道流，如同linux的命令行的管道

举个linux命令行的栗子：

```shell
# 这条命令将产生一个32位的随机字符串
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

`cat`的结果输入到管道，`tr`接受上一个管道的结果，然后过滤字符，完成后将结果输入下一个管道...这样一直传递下去。

说完linux的栗子，举一个`gulpfile`的栗子

```javascript
gulp.task("minify", ()=>{
    return gulp.src("scripts/**/*.js")
        .pipe(concat("all.js"))
        .pipe(uglify())
        .pipe(rename("all.min.js"))
        .pipe(gulp.dest("dest"));
});
```

`gulp.src("scripts/**/*.js")`先将`scripts`目录下的所有js文件读取，通过管道将结果交给`concat("all.js")`，`concat`将管道数据合并到一个文件，输出给`uglify`，`uglify`将数据压缩，最后`rename`重命名为`all.min.js`，`gulp.dest`则将数据保存到`dest`目录。

整个过程是不是很像linux命令中的管道？

### browserify

browserify可以将js模块的依赖全部编译到一个文件中，举一个栗子看看browserify能做什么吧

创建两个文件：`foo.js`和`main.js`

```javascript
// foo.js
module.exports = function(name){
    return "hello " + name;
}

// main.js
var foo = require("./foo.js");
var text = "" + foo("choldrim");
console.log(text);
```

在命令行执行：

```shell
# 先安装browserify （cnpm的安装请看后面）
cnpm install browserify

# 编译
node_modules/browserify/bin/cmd.js main.js > bundle.js

# 运行
node bundle.js
# hello choldrim
```

下面这个栗子是对node模块的引用，还可以这样子

```javascript
var unique = require('uniq');
var data = [1, 2, 2, 3, 4, 5, 5, 5, 6];
console.log(unique(data));
```

```shell
# 安装uniq模块
cnpm install uniq

# 编译 运行
node_modules/browserify/bin/cmd.js main.js | node
# [ 1, 2, 3, 4, 5, 6 ]
```

关于browserify和webpack的区别，以及RequireJS、CommonJS、Sea.js的对比文章太多了，如 [RequireJS, Sea.js, Browserify和webpack的对比](https://github.com/boxizen/boxizen.github.io/issues/9)，[Webpack Compared](http://survivejs.com/webpack/webpack-compared/)，前端还正是遍地开花了，一言不合，发文章黑，一言不合，重写一个框架。

其实各种类似的产品很多，假如说你一款都没有用过的话，看再多的对比都是没有用的，因为你不知道他们在比什么，最简单的方式就是去使用，使用过了才知道真正的痛点在什么地方，这里不做对比，不说痛点，就只用`browserify`！

### 整合使用

前面对各部分都做了些简单的介绍，那怎么结合使用呢？

俗话说，`A demo is worth a thousand words`，此处我做了一个栗子项目，托管在github上

项目地址：[simple-web-framework](https://github.com/choldrim/simple-web-framework)

详细的说明我放到了项目主页了，其他我就不说多了，自己意会去吧 ;)

#### cnpm

因为墙的原因，npm的速度如果不使用代理简直是慢到想砸电脑，不过感谢taobao，对npm仓库做了个国内的镜像源

怎么使用这个快熟的镜像源呢？

如果你使用的是`bash`，请在你`~/.bashrc`文件最后添加如下内容，如果你用的是`zsh`，请在`~/.zshrc`文件末尾添加如下内容
```shell
alias cnpm="npm --registry=https://registry.npm.taobao.org \
--cache=$HOME/.npm/.cache/cnpm \
--disturl=https://npm.taobao.org/dist \
--userconfig=$HOME/.cnpmrc"
```

添加完之后执行`source ~/.zshrc`或者`source ~/.bashrc`之后，就可以使用`cnpm`啦 ;)

#### 参考

[《ECMAScript 6标准入门》](http://es6.ruanyifeng.com/)
[Gulp挑战Grunt，背后的哲学](http://www.w3ctech.com/topic/74)
[前端工程的构建工具对比 Gulp vs Grunt](https://segmentfault.com/a/1190000002491282)
[Browserify 使用指南](http://zhaoda.net/2015/10/16/browserify-guide/)
[browserify.org](http://browserify.org/)
[TAONPM](https://npm.taobao.org/)
[SASS用法指南](http://www.ruanyifeng.com/blog/2012/06/sass.html)
[SASS中文文档](http://sass.bootcss.com/docs/sass-reference/)
[Webpack Compared](http://survivejs.com/webpack/webpack-compared/)
