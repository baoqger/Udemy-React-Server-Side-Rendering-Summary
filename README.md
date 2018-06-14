# Udemy-React-Server-Side-Rendering-Summary
Udemy React Server Side Rendering总结

根据自己的理解，实现SSR的方案还是很多的，所以应该没有一种方法是固定，本来写程序就是一个件很自由的事情。看看这个项目里实现的一些细节，总结如下：

#### 简单起步 

从最简单的原理来说，SSR就是返回的是一个完整的HTML文档，而不是SPA的index.html文件那样空空如也。所以需要在server端对react的组件进行渲染，然后进行返回。
这个过程需要跟node服务(此处使用express框架)的路由进行整合。

比如，下面这个最简单的case：

```
const express = require('express');
const React = require('react');
// 关键方法renderToString
const renderToString = require('react-dom/server').renderToString;
// 需要渲染的组件Home
const Home = require('./client/components/Home').default;
const app = express();

app.get('/', (req, res) => { // express路由的响应
  // 利用renderToString渲染Home组件
  const content = renderToString(<Home />);
  // response返回
  res.send(content);
});

app.listen(3000, () => {
  console.log('listening on port 3000');
});
```
其中的关键是renderToString这个API，它能够把组件渲染成HTML字符串。

另外，还有一个问题是，renderToString(<Home />),这里是JSX语法，node当然不支持，所以需要用babel进行转码，通过配置webpack文件可以实现这个转码的能力。

webpack.server.js
```
const path = require('path');

module.exports = {
  // Inform webpack that we're building  a bundle
  // for nodejs, rather than for  the browser
  target: 'node',

  // Tell webpack the root file of our server application
  entry: './src/index.js',

  // Tell webpack where to put the output file that is generated
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'build')
  },
  // tell webpack to run babel on every file it runs through
  module: {
    rules: [
      {
        test: /\.js?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        options: {
          presets: [
            'react',
            'stage-0',
            ['env', { targets: { browsers: ['last 2 versions'] }}]
          ]
        }
      }
    ]
  }
};
```

#### 两个js bundle文件

延续着上面的方法，renderToString把Home组件在server端渲染成HTML字符串，但是它之只有HTML，而没有JS的部分，也就是说，如果给Home组件加一个Button按钮
并去触发button的点击事件，在client端这个事件是无法触发的：

```
import React from 'react';

const Home = () => {
  return (
    <div>
      <div>I'm the best home component...</div>
      <button onClick={() => console.log('hi there')}>Press me!</button>
    </div>
  );
};

export default Home;
```
原因就是renderToString没有JS，只有HTML。

所以我们需要自己构造一个模板，然后用<script>标签从server端向client端传一个js文件，如下所示：
index.js(server端的入口文件)  
```
//静态资源默认public文件夹
app.use(express.static('public'));

app.get('/', (req, res) => {
  const content = renderToString(<Home />);
  // 构造一个HTML模板
  const html = `
    <html>
      <head></head>
      <body>
        // note: 这个div元素是root
        <div id="root">${content}</div>
        // js bundle文件
        <script src="bundle.js"></script>
      </body>
    </html>
  `;
  res.send(html);
});
```
如上所示。

当然，这里还有一个问题，就是我们需要两个JS的bundle文件是。因为server端的一些内容比如API KEY，这些敏感信息是不适合传递到client端的。
为此，我们需要再写一个client.js入口文件，作为client端的入口，还要再写一个webpack.client.js文件，对client端的内容进行打包。

webpack.client.js
```
const path = require('path');

module.exports = {
  // 入口文件不同
  entry: './src/client/client.js',

  // Tell webpack where to put the output file that is generated
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'public') // 输出文件的路径也不同, 这个跟express服务中的static资源文件夹对应上
  },
  // babel部分一样
  module: {
    rules: [
      {
        test: /\.js?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        options: {
          presets: [
            'react',
            'stage-0',
            ['env', { targets: { browsers: ['last 2 versions'] }}]
          ]
        }
      }
    ]
  }
};
```
而client.js这个入口文件应该是怎样的呢？其实就是普通的react应用的入口文件就好了：

client.js
```
import React from 'react';
import ReactDOM from 'react-dom';
import Home from './components/Home';
// 如意render函数挂载的元素，#root，这个跟上面的自己构造的模板是对应的
ReactDOM.hydrate(<Home />, document.querySelector('#root')); 
```
对于这个架构，可以理解为：server端渲染生成了纯HTML的组件，并同时把client端的bundle文件发过去，到了client端这个bundle文件接管了整个页面，附加上各种 
JS事件和生命周期的内容等等。


