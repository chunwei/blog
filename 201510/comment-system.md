1. ## 需求
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

2. ## 框架
> 
> 基于上面的需求大致勾勒出的系统框架，如下图
原系统是一个java web project，运行于Tomcat容器的Servlet/JSP构造页面
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

3. ## 前端选型
> 
> * 这次前后端分离的比较彻底，前端专心处理用户界面的渲染和状态维护，比较契合ReactJS思想，因此决定选React作为UI组件的基础。
> * hash路由管理选React-Router与React匹配，其实简单的跳转直接用window.location.hash也是可以的。
> * 数据管理方面暂时没有引入flux，存粹的jquery ajax操作，配合react组件的props和states已经能满足需要。
> * 编译打包用webpack，不但对react、ES6支持比较好，配合插件使用在后面的调试阶段还能实时看到变化。 
> 
#### 简单介绍一下`React`、`React-Router`和`Webpack`:
>> 
***
>> **[React 详情](https://facebook.github.io/react/)**  
***
>>  
`仅仅是UI`  
许多人使用React作为MVC架构的V层。 尽管React并没有假设过你的其余技术栈， 但它仍可以作为一个小特征轻易地在已有项目中使用。  
`虚拟DOM`  
React为了更高超的性能而使用虚拟DOM作为其不同的实现。 它同时也可以由服务端Node.js渲染 － 而不需要过重的浏览器DOM支持  
`数据流`  
React实现了单向响应的数据流，从而减少了重复代码，这也是它为什么比传统数据绑定更简单。  
***
>> **[React Router 详情](https://github.com/rackt/react-router)**   
***
>> React Router keeps your UI in sync with the URL. It has a simple API with powerful features like lazy code loading, dynamic route matching, and location transition handling built right in. Make the URL your first thought, not an after-thought.  
***
>> **[Webpack 详情](http://webpack.github.io/docs/what-is-webpack.html)** 
***
>> webpack is a module bundler.  
>> webpack takes modules with dependencies and generates static assets representing those modules.  
>> ![Webpack](http://webpack.github.io/assets/what-is-webpack.png)  


4. ## 开发&测试
>
```本来想开发和测试分开写的，写着写着发现这两块纠缠太多，还是放一起好了 ```  
>
>讲了那么多，该动手干活了。  
```注：本文的系统环境是windows ```  
>先配置一下开发环境，需要nodejs。到[Nodejs官网下载](https://nodejs.org/)并安装。  
装完之后自带npm (node 环境里的包管理工具)。  
可以打开cmd 试试`node -v `和`npm -v`, 正常显示版本信息应该就ok了。  
>
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
> 所谓组件就是可以组装起来的零件，一个大的系统可以分解成小系统，系统可以分解成组件，  
> 反过来同一个的组件也可以被组装在不同的系统里，实现复用  
这个关系就像螺丝钉和发动机。

5. ## 测试

6. ## 部署