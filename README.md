## 概述
1.物理像素（physical pixel）
一个物理像素是显示器（手机屏幕）上最小的物理显示物理单元，在操作系统的调度下，每一个设备都有自己的颜色值和亮度值。  
2.设备独立像素（density-independent pixel）
设备独立像素（也叫密度无关像素），可以认为是计算机坐标系统中得一个点，这个点代表一个可以由程序使用的虚拟像素（比如：css像素），然后由相关系统转换为物理像素。  
3.设备像素比（device pixel ratio）
设备像素比(简称dpr)定义了物理像素和设备独立像素的对应关系  
设备像素比＝物理像素／设备独立像素 （在x方向或者y方向）
### dpr
设备像素比DPR(devicePixelRatio)是默认缩放为100%的情况下，设备像素和CSS像素的比值  
DPR = 设备像素 / CSS像素(某一方向上)  
screen.width/screen.height获取理想窗口的大小  
window.devicePixelRatio获取dpr大小  
以iphone5为例，iphone5的CSS像素为320px\*568px，DPR是2，所以其设备像素为640px\*1136px

    640(px) / 320(px)  = 2
    1136(px) / 568(px) = 2
    640(px)*1136(px) /  320(px)*568(px) = 4

### rem
相对长度单位。相对于根元素(即html元素)font-size计算值的倍数  

### postcss
PostCSS 本身并不处理任何具体的任务，只有当我们为其附加各种插件之后，它才具有实用性。

PostCSS 就像是一个使能器（enabler），它可以不用完全替代现有的预处理器或后处理器，而只是作为它们的补充工具。PostCSS 的工作机制主要包含解析代码、执行插件、渲染结果三部分：
![image](http://www.w3cplus.com/sites/default/files/blogs/2017/1707/figure-18.png)  
PostCSS 会将 CSS 代码解析成包含一系列节点的抽象语法树（AST，Abstract Syntax Tree）。树上的每一个节点都是 CSS 代码中某个属性的符号化表示。换言之，如果你写了条件语句并对应三种结果，那么在抽象语法树中就会有一个包含三个分支的节点，每个分支就是符号化表示的结果。

### 实现
安装postcss       npm install --save-dev postcss  
安装express  
启动文件
```javascript
var express = require('express');
var webpackHandle = require("./webpack/webpackHandle");
var bodyParser = require("body-parser");
app = express();

webpackHandle(app);

app.set('port', process.env.PORT || 3333);

app.use(express.static('./dist'));

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: false }));


app.listen(app.get('port'), function() {
    console.log(`服务启动 http://localhost: ${app.get('port')}`);
});

```

安装webpack  
根目录下创建webpack文件夹，webpack文件夹下创建webpack.common.js和webpack.dev.js以及webpackHandle.js
webpack.common.js  
```javascript
const path = require('path');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const HtmlWebpackPlugin = require('html-webpack-plugin');

// 获得路径
const resolve = function(src) {
  return path.join(__dirname, "..", src);
};

module.exports = {
  entry: {
    app: resolve('src/index.js'),
    css: resolve('src/index.css')
  },
  output: {
    filename: '[name].bundle.js',
    path: resolve('dist')
  },
  plugins: [
		new CleanWebpackPlugin([resolve('dist')]),
		new HtmlWebpackPlugin({
      template: resolve('src/index.html')
    })
  ],
  module: {
    rules: [{
      test: /\.css$/,
      use: [
        {
          loader: 'style-loader',
        },
        {
          loader: 'css-loader',
          options: {
            importLoaders: 1,
          }
        },
        {
          loader: 'postcss-loader',
          options: {
            config: {
              path: resolve('postcss.config.js')
            }
          }
        }
    ]
    }]
  },
  resolve: {
    // 可以忽略的文件类型
    extensions: ['.js'],
    // 别名
    alias: {

    }
  },
};

```
webpack.dev.js
```javascript
const merge = require('webpack-merge');
const common = require('./webpack.common.js');
const webpack = require('webpack');

process.env.NODE_ENV = "development";

module.exports = merge(common, {
  devtool: 'inline-source-map',
  devServer: {
    contentBase: './dist'
  },
  mode: 'development'
});
```
webpackHandle.js
```javascript
var webpack = require('webpack');

var config = require('./webpack.dev.js');

module.exports = function(app) {
  var compiler = webpack(config);

  var watching = compiler.watch({}, (err, stats) => {
    console.log(stats.toString({
      chunks: true, // 使构建过程更静默无输出
      colors: true // 在控制台展示颜色
    }));
    console.log("watch 修改成功");
  });
  watching.invalidate();
  return compiler;
};

```

根目录下创建postcss.config.js作为postcss的配置文件
```javascript
module.exports = {
  plugins: [
    require('./component'),
  ]
}
```

使用postcss提供的api  
walkRules 遍历容器的后代节点，为每个规则节点调用回调。
walkDecls 遍历容器的后代节点，为每个声明节点调用回调。
insertAfter 在容器中依次插入新节点。
rule.append 在容器中依次插入新声明。
postcss.rule 新建一个规则节点

创建方法遍历css规则，匹配其中的dpx，获取其中的值进行运算插入容器中  
根目录下创建component文件夹，文件夹下创建index.js  
index.js

```javascript
var postcss = require('postcss');

module.exports = postcss.plugin('myplugin', function myplugin(options) {
  return function(root) {
    options = options || {};

    root.walkRules((rule) => {
      let list = [];
      rule.walkDecls((decl) => {
        let data = getDpx(decl);
        if (data) {
          list.push(data);
          decl.remove();
        }
      })
      insertDpx(rule, list);
    })
  }
});


const getDpx = (decl) => {
  let dpxReg = /(\d+)(dpx)([\s]+|[;]|$)/;
  let value = decl.value;
  if (value.match(dpxReg)) {
    let num = 0;
    value.replace(dpxReg, function(a, b) {
      num = parseFloat(b);
    })
    return {
      prop: decl.prop,
      value: num
    }
  }
  return undefined;
}

const insertDpx = function(rule, list) {
  if (!list || list.length === 0) {
    return;
  }
  let unit = 'px';
  for (let i = 3; i > 0; i--) {
    let ruleName = `[data-dpr="${i}"] ${rule.selector}`;
    let newRule = createRule(ruleName);
    list.forEach(decl => {
      createDecl(newRule, decl.prop, (decl.value * i) + unit);
    });
    rule.parent.insertAfter(rule, newRule);
  }
}

const createRule = function(name) {
  return postcss.rule({ selector: name })
}
const createDecl = function(rule, prop, value) {
  rule.append({ prop, value });
}

```

参考  
[响应式网页开发基础：DPR 与 viewport](https://zhuanlan.zhihu.com/p/26131956)  
[等比例缩放rem](https://www.zhihu.com/question/21504656)  
[深入postcss](http://www.w3cplus.com/preprocessor/postcss-book.html)