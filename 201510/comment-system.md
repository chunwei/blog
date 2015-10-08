## 1.需求
>
> ##### 在现有的旅游项目基础上增加网页评论功能：评论分两级展示、发表新评论、回复旧评论、点赞
> * 多入口：景区页面、子景点页面、美食特产、酒店页面等都可以独立进行评论；
> * 多公号：项目是开放平台允许各方公众号接入，因此需要公众号appid和openid联合确定用户（昵称、头像）
> * 关联性：每个景区需要一个独立的评论汇总页面，包含针对景区本身及其子景点和相关美食特产等的评论
> * 分页：控制每页的展示长度，点more按钮加载更多
> * 组件化：要求可以在任意页面任意位置插入评论组件，插入式最小化对原系统影响
> * 无刷新：全过程ajax（首次载入、新增评论、点赞、加载更多）
> * hash路由：列表和评论输入框之间支持浏览器返回
![2.PNG](https://github.com/chunwei/blog/raw/master/assets/2.PNG)

## 2.框架
> 
```
注：原系统是一个java web project，运行于Tomcat容器的Servlet/JSP构造页面
```
> 基于上面的需求大致勾勒出的系统框架，如下图
> * 为尽量少破环原系统的组织结构，基本定调是前后端分离，通过ajax收发json格式进行数据交互。
> * 后台开发几个新的api接口，解析前端发来的json对象，根据参数按需返回json数据对象
> * 如果每次都去微信服务器获取用户数据（头像、昵称等）非常费时，所以考虑建立本地用户数据库，每天闲时自动与微信服务器同步
> * 点赞计数和评论计数考虑放到Redis缓存中，避免每次查询
> * 最新评论列表也考虑缓存一份list在Redis中
> * 系统较简单前端打包成少2个js：vender.js和comment.js，在原页面body结束前引入
> * 原页面只需要在希望的位置插入评论容器，如<div id='comment'></div>
> * 前端元素维护自身的状态，加快用户响应速度，为了简化状态，考虑用单向数据流，
> * 用户有操作时先改变前端数据状态，更新界面，同时向后端发出ajax；
> * 待收到后端返回的数据后，再次更新前端数据状态，更新必要的界面；
![3.PNG](https://github.com/chunwei/blog/raw/master/assets/3.PNG)

## 3.前端选型
> 
> * 这次前后端分离的比较彻底，前端专心处理用户界面的渲染和状态维护，比较契合ReactJS思想，因此决定选React作为UI组件的基础。
> * hash路由管理选React-Router与React匹配，其实简单的跳转直接用window.location.hash也是可以的。
> * 数据管理方面暂时没有引入flux，存粹的jquery ajax操作，配合react组件的props和states已经能满足需要。
> * 编译打包用Webpack，配合插件使用不但对React、ES6支持比较好，在后面的调试阶段还能实时看到变化。 
> 
> ##### 简单介绍一下`React`、`React-Router`和`Webpack`:
>> 
***
>> **[React 详情](https://facebook.github.io/react/)**  
***
>>  
 * **仅仅是UI**  
许多人使用React作为MVC架构的V层。 尽管React并没有假设过你的其余技术栈， 但它仍可以作为一个小特征轻易地在已有项目中使用。  
 * **虚拟DOM**  
React为了更高超的性能而使用虚拟DOM作为其不同的实现。 它同时也可以由服务端Node.js渲染 － 而不需要过重的浏览器DOM支持  
 * **数据流**  
React实现了单向响应的数据流，从而减少了重复代码，这也是它为什么比传统数据绑定更简单。  
>>  
***
>> **[React Router 详情](https://github.com/rackt/react-router)**   
***
>> React Router keeps your UI in sync with the URL. It has a simple API with powerful features like **lazy code loading**, dynamic route matching, and location transition handling built right in. Make the URL your first thought, not an after-thought.  
***
>> **[Webpack 详情](http://webpack.github.io/docs/what-is-webpack.html)** 
***
>> webpack is a module bundler.  
>> webpack takes modules with dependencies and generates static assets representing those modules.  
>> ![Webpack](http://webpack.github.io/assets/what-is-webpack.png)  

## 4.开发&测试
>
```
本来想开发和测试分开写的，写着写着发现这两块纠缠太多，还是放一起好了 
```  
>
> 讲了那么多，该动手干活了。  
> ## 环境配置
```
注：本文的系统环境是windows， 浏览器调试用Chrome
 ```  
> ####先配置开发环境
需要**npm**进行包管理 ，安装**nodejs**，装完之后自带npm 。  
到[Nodejs官网下载](https://nodejs.org/)并安装。  
可以打开cmd 试试`node -v `和`npm -v`, 正常显示版本信息应该就ok了。  
>
> ####再配置调试环境
在编写业务代码前，先把调试环境配置好  
#####希望建立什么样的调试环境？
- 运行在浏览器中
- 可以同时在本机和远程设备设备如手机上调试
- 代码和式样文件编辑的同时，自动编译，浏览器自动同步更新
>
>>
#####注：这里所谓的“编译”，是指：
- 把JSX转成纯 javascript    ---[深入JSX](http://facebook.github.io/react/docs/jsx-in-depth.html)
- 把ES6等新语法转成ES5兼容语法  ---[深入ES6](http://www.infoq.com/cn/es6-in-depth/)
- 对CSS做浏览器兼容处理
- 对图片资源按设定处理，如小图转base64嵌入格式
- 文件版本控制
- JS、CSS、图片统一打包、合并、分割生产输出
>
#####怎么做？
- 安装和配置**webpack、webpack-dev-server** 
- 配置一个**express数据api服务器**
- 安装和配置**Weinre** 
>
>开始吧!  
###### 安装和配置**webpack、webpack-dev-server** 
1.安装**webpack**  
```
npm install webpack --save-dev
```  
>
npm install `package` 会在当前的目录node_modules目录安装指定的包，  
要指定版本话用@，如`jquery@2.1.4`，不指定的话默认安装最新版  
`--save-dev`指令告诉npm在package.json添加一条开发依赖（发布的时候不需要）
>>这样做的好处是当我们把代码上传到git等版本库时不需要上传依赖，也就是node_modules里东西  
使用者clone完后只需`npm install` (不指定包)就会自动安装package.json里定的所有依赖包。  
>
2.安装**webpack-dev-server**用于实时调试和热更新  
```
npm install webpack-dev-server --save-dev
```
>
3.配置**webpack.config**
>在工程根目录下新建`webpack.config.js`   
>
 ```javascript
var path = require('path');
var node_modules = path.resolve(__dirname, 'node_modules');
var pathToReactMini = path.resolve(node_modules, 'react/dist/react.min.js');
var pathToReactRouterMini = path.resolve(node_modules, 'react-router/umd/ReactRouter.min.js');
var pathToJQuery = path.resolve(node_modules, 'jquery/dist/jquery.min.js');
module.exports = {
  resolve: {
    alias: {
      'react': pathToReactMini,
      'jquery': pathToJQuery,
      'react-router': pathToReactRouterMini
    }
  },
  entry: {//入口，这里放"webpack/hot/dev-server"是为了热更新
    comment: ["webpack/hot/dev-server", path.resolve(__dirname, 'src/pages/index/index.js')]
  },
  output: {//输出
    path: path.resolve(__dirname, 'public', 'build'),
    filename: '[name].js'
  },
  module: {
    loaders: [{
      test: /\.jsx?$/,// A regexp to test the require path. accepts either js or jsx
      loader: 'babel-loader'// The module to load. "babel" is short for "babel-loader"
    }, {
      test: /\.css$/,
      loader: 'style!css!autoprefixer?{browsers:["last 2 version", "> 1%"]}' //shortcut, Run both loaders style-loader , css-loader and autoprefixer-loader
    }, {
      test: /\.(png|jpg)$/,
      loader: 'url-loader?limit=16384'  //images that er 25KB or smaller in size will be converted to a BASE64 string and included in the CSS file where it is defined
    }],
    noParse: [pathToReactMini, pathToReactRouterMini,pathToJQuery]
  }
}
```
>
> 配置中用到了几个常用的loader，分别对应下面的需求：
>  - babel-loader用于处理React的JSX和ES6语法  ---[深入Babel](https://babeljs.io/)
>  - style-loader、css-loader、autoprefixer用于处理CSS引用及浏览器前缀
>  - url-loader用于处理图片
> 安装对应的包：  
>
```
npm install babel-loader style-loader css-loader autoprefixer-loader --save-dev
```
> 有了这个配置在工程根目录下运行webpack就会在public/build目录下生产编译打包好的js文件。  
为了方便后面运行，在package.json文件里加一点脚本：  
```javascript
"scripts": {
    "dev-server": "webpack-dev-server --host 0.0.0.0 --devtool eval --progress --colors --hot --content-base public/build"
  }
```
>这是注册一个npm run指令，可以通过`npm run dev-server`来运行冒号后面的脚本  
>这串脚本的作用是启动webpack开发服务器， watch文件变化，实时编译合并  
默认读取`webpack.config.js`配置信息，可通过--config来指定配置文件。  
>> ##### webpack-dev-server 的一些可选参数说明：
- --devtool eval ：为你的代码创建源地址。当有任何报错的时候可以让你更加精确地定位到文件和行号
- --content-base <file/directory/url/port>: base path for the content.
- --quiet: don’t output anything to the console.
- --no-info: suppress boring information.
- --colors: add some colors to the output.
- --no-colors: don’t used colors in the output.
- --host <hostname/ip>: hostname or IP.  **默认localhost，指定ip以便在其他设备上调试**
- --port <number>: port.  默认在8080端口监控
- --inline: embed the webpack-dev-server runtime into the bundle.
- --hot: adds the HotModuleReplacementPlugin and switch the server to hot mode. 
- --hot --inline also adds the webpack/hot/dev-server entry.
>
>自此webpack-dev-server基本配置完成，此时运行`npm run dev-server`  
你会看到命令行输出类似：
```
Hash: 65115b420cf6af9a5ad1
Version: webpack 1.12.2
Time: 59ms
webpack: bundle is now VALID.
http://0.0.0.0:8080/webpack-dev-server/
webpack result is served from /
content is served from C:\Users\chunwei\WebstormProjects\comment\public\build
```
>打开浏览器，输入
```
http://localhost:8080/   或者
http://localhost:8080/webpack-dev-server/
```
>能看到开发服务器已经运行，只是没有内容。  
>关闭开发服务器方法是`Ctrl+C`
>
###### 配置一个**express数据api服务器**
>一般应用都需要后台服务器或Mock服务器来提供数据，  
但是，一般不用webpack-dev-server作为后端服务器，让webpack-dev-server只服务于webpack生成的静态文件  
后台服务器或Mock服务器另外启一个，并与webpack-dev-server协同使用。  
为了让我们的评论系统run起来，我们也需要后台服务器  
当然在开发的时候最好不要与现有的后端系统纠缠在一起，这个时候其实只需要一个单纯的API端点，提供和保存数据。  
为了尽量简单，我们只需要配置一个最基本nodejs环境的express服务器，并使用一个JSON文件数据库。  
>
###### 来看看怎么做  
0.创建JSON数据库，在工程根目录下新建comments.json，输入一些虚拟数据。  
>
1.安装**express**  
```
npm install express body-parser --save-dev
```
>
2.配置**express**，在工程根目录下新建server.js，输入下面内容  
```javascript
var fs = require('fs');
var path = require('path');
var express = require('express');
var bodyParser = require('body-parser');
var app = express();
var RSVP={};
RSVP.uuid_chars="0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".split("");
RSVP.uuid=function(len,radix){var chars=RSVP.uuid_chars,uuid=[],i;radix=radix||chars.length;if(len){for(i=0;i<len;i++){uuid[i]=chars[0|Math.random()*radix]}}else{var r;uuid[8]=uuid[13]=uuid[18]=uuid[23]="-";uuid[14]="4";for(i=0;i<36;i++){if(!uuid[i]){r=0|Math.random()*16;uuid[i]=chars[(i==19)?(r&3)|8:r]}}}return uuid.join("")};
app.set('port', (process.env.PORT || 3000));
app.use('/', express.static(path.join(__dirname, 'public')));
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({extended: true}));
app.get('/comments.json', function(req, res) {
  fs.readFile('comments.json', function(err, comments) {
    res.setHeader('Cache-Control', 'no-cache');
    var data={Status:"Success",Result:JSON.parse(comments)};
    res.json(data);
  });
});
app.post('/comments.json', function(req, res) {
  fs.readFile('comments.json', function(err, data) {
    var comments = JSON.parse(data);console.log(req.body);console.log(req.query);console.log(req.params);
    var comment=JSON.parse(req.body.params);
    var tempid=comment.id;
    comment.id='real-'+RSVP.uuid(10);
    comment.author.name='Nick-'+RSVP.uuid(5);
    comment.author.avatar="http://tb.himg.baidu.com/sys/portrait/item/65f7696d6465766963659913";
    var pId=comment.syncId;
    if(pId&&pId!=0){//这是回复评论
      var parentComment;
      for(var item of comments){
        if(item.id==pId) {
          parentComment=item;
          break;
        }
      }
      if(!!!parentComment.replys)parentComment.replys=[];
      parentComment.replys.push(comment);
    }else{
      comments.push(comment);
    }
    //comments.push(req.body);
    fs.writeFile('comments.json', JSON.stringify(comments, null, 4), function(err) {
      res.setHeader('Cache-Control', 'no-cache');
      comment.tempid=tempid;
      var returndata={Status:"Success",Result:[comment]};
      res.json(returndata);
    });
  });
});
app.post('/like', function(req, res) {
  fs.readFile('comments.json', function(err, data) {
    var comments = JSON.parse(data);
    var likeAction=req.body;
    var cId=likeAction.commentId;
    var like=parseInt(likeAction.like);
    var whichOne;
    if(cId) {
      for (var item of comments) {
        if (item.id == cId) {
          whichOne = item;
          break;
        }
        if(item.replys) {
          for (var reply of item.replys) {
            if (reply.id == cId) {
              whichOne = reply;
              break;
            }
          }
        }
        if(whichOne)break;
      }
      if(whichOne){
        whichOne.liked = like;
        whichOne.likedCount = parseInt(whichOne.likedCount) + (like > 0 ? 1 : -1);
      }
    }
    //comments.push(req.body);
    fs.writeFile('comments.json', JSON.stringify(comments, null, 4), function(err) {
      res.setHeader('Cache-Control', 'no-cache');
      //res.json(comments);
      var returndata={Status:"Success",Result:comments};
      res.json(returndata);
    });
  });
});
app.listen(app.get('port'),'0.0.0.0', function() {
  console.log('Server started: http://localhost:' + app.get('port') + '/');
});
```
>配置在3000端口监听，具体业务逻辑现在可以先忽略，只看接口  
- 定义/服务public文件夹下的静态资源
- 定义/comments.json get和post api来获取和保存评论数据
- 定义/like post api来保存点赞数据
>
3.在package.json文件的scripts段增加
```javascript
"start": "node server.js",
```
>运行下面命令来启动
```
npm start
```
>通过在浏览器打开下面地址来测试  
```
http://localhost:3000/comments.json
http://localhost:3000/like
```
>能看到express服务器已经运行  
>关闭方法也是`Ctrl+C`
>
> ## 程序组织
接下来看看程序怎么组织。  
首先组件化开发，同一组件内的资源尽量放在一起，方便就近取用、移植和复用，减少外部依赖。  
目录结构差不多是这样安排的：  
![7.PNG](https://github.com/chunwei/blog/raw/master/assets/7.PNG)   
看着有点花，下面一步一步来：  
首先不管用什么工具建一个工程文件夹，我用的是webstorm，  
我的工程文件夹名为react-comment。  
在此目录下新建public、src文件夹，在src下新建components、pages文件夹 ,如下：  
react-comment/  
>> public/  
>> src/  
>>> components/  
>>> pages/  
> 
>回到项目根目录  
>
```cd react-comment```  
```npm init```  
>
npm init  会引导你创建一个package.json文件，包括名称、版本、作者这些信息等  
>
`npm install react --save`  
>
npm install `package` 会在当前的目录node_modules目录安装指定的包，  
要指定版本话用@，如`react@0.1.3`，不指定的话默认安装最新版  
`--save`指令告诉npm在package.json添加一条运行时依赖  
>
`npm install react-router jquery --save`  
>
现在react、react-router、jquery都被安装到了node_modules目录中  
>
> ##组件：分解和组装
根据上图我已经把这个评论页面分解成一个个小的组件，包括评论组件和点赞组件
>
> - CommentBox
>   - CommentList
>     - Comment
>       - CommentHeader
>         - CommentAuthor
>         - LikeButton
>       - CommentContent
>       - CommentFooter
>   - CommentBar
>   - CommentForm
>
> 所谓组件就是可以组装起来的零件，一个大的系统可以分解成小系统，  
系统可以分解成组件，组件还可以再分解成更小的组件，  
> 反过来同一个的组件也可以被组装在不同的系统里，实现复用  
这个关系就像螺丝钉和发动机。  


## 5.测试

## 6.部署