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
