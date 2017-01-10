---
title: 【译】13个最佳实践来保护您的Web应用程序
date: 2017-01-10 14:19:20
tags:
	- web
---

每个人都同意Web应用程序的安全性非常重要，但很少认真对待。 以下是在部署下一个Web应用程序之前应该遵循的13步安全检查清单。

<!--more-->

我已经添加了一些npm模块的链接，在适当的时候，有助于解决这些问题。


# 使用SSL进行通信

如果你的web应用程序应该做一件事，就是这样。 确保与服务器的所有通信都通过SSL。 这确保用户和服务器之间的通信是加密的。
通常，如果您使用的是nginx或某些负载均衡器，SSL将在它到达您的应用程序之前被终止。 但是，如果您只是使用Express服务器，它仍然可以支持SSL。 查看此[StackOverflow](http://stackoverflow.com/questions/11804202/how-do-i-setup-a-ssl-certs-for-an-express-js-server)响应，其中链接到有关如何在Express服务器上设置SSL的相关文档。
您可以从StartSSL [LetsEncrypt](https://letsencrypt.org/)免费获得SSL证书，因此没有理由没有SSL证书。 （[ipgof的说明](https://www.reddit.com/r/node/comments/5ltgxc/13_security_best_practices_for_your_web/)，StartSSL证书已经开始不信任Mozilla）。

# 始终转义用户数据

确保用户通过表单输入的数据被转义。 有不同的方法这样做。 一种常见的方法是使用像[Validator.js](https://github.com/chriso/validator.js)这样的清理器，它确保您的表单提交正确的数据类型。
在服务器端，您不应该直接将数据输入原始数据库查询。 更多关于这一点下面。

# 对数据库查询使用准备的语句

创建数据库请求，直接将用户信息传递到语句中是灾难的。 相反，请考虑使用预[准备语句](https://en.wikipedia.org/wiki/Prepared_statement)。
预准备语句是由应用程序创建并发送到数据库的模板。 某些值未指定。
```sql
INSERT INTO PRODUCT (name, price) VALUES (?, ?)
```
准备的语句对[SQL注入](https://en.wikipedia.org/wiki/SQL_injection)具有弹性，因为稍后使用不同协议传输的参数值不需要正确转义。 如果原始语句模板不是从外部输入派生的，则不能发生SQL注入。
如果你使用一个ORM来访问数据库（Mongoose，Sequelize，Waterline等），ORM通常会通过使用预备语句来处理SQL注入。 检查你的ORM的文档，看看他们是否这样做。

# 从请求网址中删除敏感信息

在构建Web应用程序时，我们通常在向用户显示数据时遵循RESTful约定。 例如，您可能有一个用户个人资料页面，并且网址可能如下所示：

```
	/view/users/:userId
```

在这种情况下，我们向最终用户公开显示userId。 虽然这可能是好的，也可能有理由隐藏这些ID。 同样，您可以设想公开网址披露敏感信息的情况。
没有办法检查这个程序，但你应该通过你的根定义，仔细检查你的路由和查询字符串不显示敏感信息。

# 只允许重定向到已列入白名单或硬编码的网址

没有基于用户输入重定向到某处的服务器端重定向。 相反，每个重定向应该是一个硬编码的URL。 这样可以轻松测试并确保重定向到达您期望的位置。

# 停用所有未使用的API路由

在将Web应用程序部署到生产环境之前，请确保正在使用所有API路由，并禁用任何未使用或未受保护的API路由。 如果您使用自动生成REST API端点（如Sails或Feathers）的库，这一点尤其重要。

# CSRF令牌应存在于创建或更新数据的所有页面中

跨站点请求伪造（CSRF）是一种强制用户在其当前登录的Web应用程序上执行不需要的操作的攻击。

确保所有页面都具有CSRF保护，或者至少创建或更新数据的所有页面（调用非GET API端点的页面）。

在Node.js中，您可以使用csurf模块，为Express应用程序提供CSRF中间件。

```javascript
var cookieParser = require('cookie-parser')
var csrf = require('csurf')
var bodyParser = require('body-parser')
var express = require('express')
 
// setup route middlewares 
var csrfProtection = csrf({ cookie: true })
var parseForm = bodyParser.urlencoded({ extended: false })
 
// create express app 
var app = express()
 
// parse cookies 
// we need this because "cookie" is true in csrfProtection 
app.use(cookieParser())
 
app.get('/form', csrfProtection, function(req, res) {
  // pass the csrfToken to the view 
  res.render('send', { csrfToken: req.csrfToken() })
})
 
app.post('/process', parseForm, csrfProtection, function(req, res) {
  res.send('data is being processed')
})
```

在视图内（取决于您的模板语言;此处演示了handlebars样式），将`csrfToken`value设置为名为`_csrf`的隐藏输入字段的值：

```html
<form action="/process" method="POST">
  <input type="hidden" name="_csrf" value="{{csrfToken}}">
  <input type="text" name="email">
  <button type="submit">Submit</button>
</form>
```
如果你使用像Bedrock或Sails的MVC框架，CSRF保护通常内置。阅读项目文档以了解如何启用它。

# 提供注销或退出session过期的功能

简单，但经常被忽视。 确保用户能够注销您的网站并使任何会话过期。 人们可以从公共计算机使用您的Web应用程序。 如果您使用的用户身份验证系统，如PassportJS，这通常是微不足道的：

```javascript
app.get('/logout', function(req, res){ 
  req.logout(); //provided by passport
  res.redirect('/'); 
});
```
重要的是，测试这个，以确保session和cookie正在如预期的被删除。

# HTTP仅Cookie属性用于所有页面和链接

这是另一个与预防XSS不可逆性相关的问题。 使用HttpOnly标记标记Cookie将告诉浏览器该特定的Cookie只应由服务器访问。 恶意用户将无法从JavaScript访问Cookie。

在Node应用程序中实现这应该很容易。 只需更新您的会话配置：

```javascript
app.use(session({  
  secret: 'My super session secret',  
  cookie: {  httpOnly: true,  secure: true  } 
}));
```

有一个关于[great article on Coding Horror](https://blog.codinghorror.com/protecting-your-cookies-httponly/)，更详细地讨论到HTTP-Only Cookies。 我建议读读它。

# 敏感HTML表单输入字段的AUTOCOMPLETE = off属性

另一个简单，但仍被忽视的安全方面。 在HTML表单中，应确保所有敏感输入字段都具有HTML属性`autocomplete=off`。

```html
<input type="email" name="email" autocomplete="off"/>
```
这会阻止浏览器自动设置某些属性。 您不应该为每个输入元素设置此值，因为它会损害用户体验。 只是审慎和思考通过用例。

# 将X-Frame-Options设置为DENY，SAMEORIGIN或ALLOW-FROM

X-Frame-Options HTTP响应头可以用于指示是否应允许浏览器在<frame>或<iframe>中呈现页面。 网站可以使用此功能来避免Clickjacking攻击，确保其内容不会嵌入其他网站。 为包含HTML内容的所有响应设置X-Frame-Options标题。 可能的值为“DENY”，“SAMEORIGIN”或“ALLOW-FROM <url>”。

一般来说，除非您有足够的理由允许通过iframe查看您的Web应用程序，最好设置X-Frame-Options：DENY。

在节点应用程序中，您可以使用像[Helmet](https://www.npmjs.com/package/helmet)这样的模块来执行此操作，该模块提供多个安全HTTP头，包括此修复程序。

```javascript
var express = require('express')
var helmet = require('helmet')
var app = express()
 
app.use(helmet({
  frameguard: {action: 'deny'}
}))
```

# 设置安全HTTP头

从上一点开始，还有一些其他HTTP头，你应该为你的应用程序设置。

# 防止强力和DDOS攻击

为了防止您的网站遭到大量请求的攻击并随后崩溃，您应该以某种类型的速率限制为您的所有请求。

如果您正在使用Express构建Node应用程序，则可以使用[express-rate-limit](https://www.npmjs.com/package/express-rate-limit)中间件。 [ratelimiter npm](https://www.npmjs.com/package/ratelimiter)模块也很好，但它有一个Redis依赖。

```javascript
var RateLimit = require('express-rate-limit');
 
app.enable('trust proxy'); // only if you're behind a reverse proxy (Heroku, Bluemix, AWS if you use an ELB, custom Nginx setup, etc) 
 
var limiter = new RateLimit({
  windowMs: 15*60*1000, // 15 minutes 
  max: 100, // limit each IP to 100 requests per windowMs 
  delayMs: 0 // disable delaying - full speed until the max limit is reached 
});
 
//  apply to all requests 
app.use(limiter);
```






