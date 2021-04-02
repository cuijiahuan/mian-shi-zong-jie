## 一、跨域的方式

​	什么是跨域：由于js的同源策略限制，不允许a.com域名下的js操作b.com下的对象

​	当协议、子域名、主域名、端口号中任意一个不相同时，都算作不同域。

​	跨域请求可以发出去，服务端也可以收到请求正常返回结果，只是结果被浏览器拦截了

​	同源策略：限制从一个源加载的文档或脚本与来自另一个源的资源进行交互。这是一个用于隔离潜在恶意文件的安全机制。它的存在可以保护用户隐私信息，防止身份伪造(读取cookie)，但是有三个标签是允许跨域加载资源：

​	<img src="">    <link href="">    <script src="">

​	处理跨域的方法：

### 1、JSONP原理

利用``<script>`元素可以跨域加载资源的策略，网页可以得到从其他来源动态产生的数据。其中JSONP请求一定需要对方的服务器做支持才可以。

优点：兼容性好，可用于解决主流浏览器的跨域数据访问的问题。

缺点：仅支持get方法，具有局限性。

流程：

- ​	声明一个回调函数，将函数名( fn )当作参数值，要传递给跨域请求数据的服务器，函数形参为要获取目标数据( 服务器返回的data )。
- 创建一个`<script>`标签，把需要跨域的API数据接口地址，赋值给`<script`的src，还要再这个地址中向服务器传递该函数名( 如问号传参：?callback=fn )。
- 服务器接收到请求后，把传递的函数名和需要返回的数据拼接成一个字符串( 如：fn([{"name":"zhangsan"}]) )。
- 最后服务器把数据通过HTTP协议返回给客户端，客户端再调用执行之前声明的回调函数( fn )，对返回的数据进行操作。

```javascript
<script type="text/javascript">
    function fn(data) {
    	alert(data.msg);
	} 
</script>
<script type="text/javascript" src="http://crossdomain.com/jsonServerResponse?jsonp=fn"></script>


$.ajax({ url:"http://crossdomain.com/jsonServerResponse", dataType:"jsonp", type:"get",success:function (data){ console.log(data);} });//可以省略 jsonpCallback:"fn",//->自定义传递给服务器的函数名，而不是使用jQuery自动生成的，可省略 jsonp:"jsonp",//->把传递函数名的那个形参callback变为jsonp，可省略 
```

### 2、CORS

​	原理：整个CORS通信过程都是浏览器自动完成，实现CORS通信的关键是服务器，只要服务器实现了CORS接口，就可以跨源通信。

```js
res.header("Access-Control-Allow-Origin", 'https://www.google.com');
res.header('Access-Control-Allow-Methods', 'POST, GET');
res.header('Access-Control-Allow-Headers', 'X-Requested-With');
res.header('Access-Control-Allow-Headers', 'Content-Type');
res.Header("Access-Control-Allow-Credentials", "true")
```

​	优点：支持各种HTTP 方法。

​	缺点：兼容性不如JSONP。

​	流程：

```javascript
需要在服务器端做一些改造：
header("Access-Control-Allow-Origin:*"）； header("Access-Control-Allow-Methods:POST,GET"）；
       
//在服务器端设置同源策略地址 
router.get("/userlist", function (req, res, next) {
    var user = {name: 'Mr.Cao', gender: 'male', career: 'IT Education'};
    res.writeHeader(200,{"Access-Control-Allow-Origin":'http://localhost:63342'});
    res.write(JSON.stringify(user));
    res.end();
});
```

### 3、WebSocket

​	websocket是H5的一个持久化的协议，它实现了浏览器和服务器的全双工通信，同时也是跨域的一种解决方法。

​	WebSocket和HTTP都是应用层协议，都基于 TCP 协议。但是 WebSocket 是一种双向通信协议，在建立连接之后，WebSocket 的 server 与 client 都能主动向对方发送或接收数据。同时，WebSocket 在建立连接时需要借助 HTTP 协议，连接建立好了之后 client 与 server 之间的双向通信就与 HTTP 无关了。

​	流程：

```javascript
//前端代码： 
<div>user input：<input type="text"></div>
<script src="./socket.io.js"></script>
<script>
    var socket = io('http://www.domain2.com:8080'); 	// 连接成功处理 
	socket.on('connect', function() {     
        // 监听服务端消息     
        socket.on('message', function(msg) {
            console.log('data from server: ---> ' + msg);
        });  
        // 监听服务端关闭     
        socket.on('disconnect', function() {          
            console.log('Server socket has closed.');      
        }); 
    }); 
	document.getElementsByTagName('input')[0].onblur = function() {     
        socket.send(this.value); 
    }; 
</script>

//Nodejs socket后台： 
var http = require('http'); 
var socket = require('socket.io'); // 启http服务 
var server = http.createServer(function(req, res) {     
    res.writeHead(200, {         'Content-type': 'text/html'     });     
    res.end(); 
}); 
server.listen('8080'); 
console.log('Server is running at port 8080...'); // 监听socket连接 
socket.listen(server).on('connection', function(client) {     
    // 接收信息     
    client.on('message', function(msg) {         
        client.send('hello：' + msg);         
        console.log('data from client: ---> ' + msg);     
    });     
    // 断开处理     
    client.on('disconnect', function() {         
        console.log('Client socket has closed.');      
    }); 
});
```

### 4、postMessage

​	如果两个网页不同源，就无法拿到对方的DOM。典型的例子是iframe窗口和window.open方法打开的窗口，它们与父窗口无法通信。HTML5为了解决这个问题，引入了一个全新的API：

​	跨文档通信API：H5新增的一个window.postMessage方法，允许跨窗口通信。

​	postMessage方法的第一个参数是具体的信息内容，第二个参数是接收消息的窗口的源( origin )，即“ 协议 + 域名+ 端口 ”。也可以设置为“ * ”，表示不限制域名，向所有窗口发送。

流程：`http://localhost:63342/index.html`页面向`http://localhost:63342/index.html`传递“跨域请求信息”

```javascript
//发送信息页面 http://localhost:63342/index.html 
<html lang="en">   
    <head>       
    <meta charset="UTF-8">       
        <title>跨域请求</title>    
	</head>   
	<body>       
    	<iframe src="http://localhost:3000/users/reg" id="frm"></iframe>       
		<input type="button" value="OK" onclick="run()">   
	</body>   
</html>   
<script>      
    function  run(){           
    	var frm=document.getElementById("frm");           
    	frm.contentWindow.postMessage("跨域请求信息","http://localhost:3000");      
	}   
</script>

//接收信息页面 http://localhost:3000/message.html  
window.addEventListener("message",function(e){  
    //通过监听message事件，可以监听对方发送的消息。   
    console.log(e.data);   
},false);
```

### 5、nginx反向代理

​	原理：通过nginx配置一个代理服务器( 域名和domain1相同，端口不同 )，作为跳板机，反向代理访问domain2接口，并且可以修改cookie中domain信息，方便当前域cookie写入，实现跨域登录。

### 6、document.domain + iframe

​	适用于二级域名相同的情况下，比如 a.test.com 和 b.test.com

​	原理：两个页面都通过 js 强制设置 document.domain 为基础主域，就实现了同域。

​	流程：页面`a.zf1.cn:3000/a.html`获取页面`b.zf1.cn:3000/b.html`中a的值

```javascript
// a.html
<body>
 helloa
  <iframe src="http://b.zf1.cn:3000/b.html" frameborder="0" onload="load()" id="frame"></iframe>
  <script>
    document.domain = 'zf1.cn'
    function load() {
      console.log(frame.contentWindow.a);
    }
  </script>
</body>

// b.html
<body>
   hellob
   <script>
     document.domain = 'zf1.cn'
     var a = 100;
   </script>
</body>
```

## 二、前端网络安全

### 1、XSS

跨站脚本攻击：指攻击者利用网站没有对用户提交数据进行转义处理或者过滤不足的缺点，进而添加一些代码到web页面中，使别的用户访问都会执行相应的嵌入代码。

解决方法：

​	不信任客户端提交的数据，只要是客户端提交的数据都应该先进行响应的过滤处理方可进行下一步操作。

- 转义字符，对于括号，尖括号，斜杠进行转义，对于富文本进行白名单过滤
- CSP，通过指定有效域来使服务器减少或消除XSS攻击载体，如设置HTTP Header中的Content-Security-Policy:policy；或者使所有内容只允许加载本站资源：Content-Security-Policy:default-src 'self'；或者只允许加载HTTPS协议的图片：Content-Security-Policy:img-src https://*

### 2、CSRF

​	跨站请求伪造：通过伪装成受信任用户的请求来攻击受信任的网站

​	原理：攻击者构造出一个后端请求，诱导用户点击或者自动发送请求，如果用户是在登录状态下，后端就以为用户在操作，从而进行相应的逻辑

​	完成一次CSRF攻击，受害者必须依次完成两个步骤：

1. 登录受信任的网站，并在本地生成cookie
2. 在不登出信任网站的情况下，访问危险网站

防范：

1. GET请求不对数据进行更改
2. 不让第三方网站访问到cookie
3. 组织第三方网站请求接口
4. 请求时附带验证信息，比如验证码或者Token

### 3、点击劫持

​	原理：一是攻击者使用一个透明的iframe，覆盖在一个网页上，然后诱使用户在该页面进行操作，此时用户将在不知情的情况下点击透明的iframe页面；二是攻击者使用一张图片覆盖在网页，遮挡页面原有位置的含义。

​	解决方法： 使用HTTP响应头 X-Frame-Options，它有三个值，分别是：

- DENY，表示页面不允许通过iframe的方式展示，浏览器会拒绝当前页面加载的任何iframe
- SAMEORIGIN，表示页面可以在相同域名下通过iframe的方式展示，iframe页面的地址只能为同源域名下的页面
- ALLOW-FROM，表示页面可以在指定来源的iframe中展示

### 4、中间人攻击

中间人攻击（Man-in-the-MiddleAttack，简称“MITM攻击”）是一种“间接”的入侵攻击，这种攻击模式是通过各种技术手段将受入侵者控制的一台计算机虚拟放置在网络连接中的两台通信计算机之间，这台计算机就称为“中间人”。

中间人攻击是攻击方同时与服务端和客户端建立起了连接，并让对方认为连接是安全的，但是实际上整个通信过程都被攻击者控制了。攻击者不仅能获得双方的通信信息，还能修改通信信息。

通常来说不建议使用公共的 Wi-Fi，因为很可能就会发生中间人攻击的情况。如果你在通信的过程中涉及到了某些敏感信息，就完全暴露给攻击方了。

当然防御中间人攻击其实并不难，只需要增加一个安全通道来传输信息。HTTPS 就可以用来防御中间人攻击，但是并不是说使用了 HTTPS 就可以高枕无忧了，因为如果你没有完全关闭 HTTP 访问的话，攻击方可以通过某些方式将 HTTPS 降级为 HTTP 从而实现中间人攻击。

## 三、this指向问题

1. this永远指向一个对象
2. this的指向完全取决于函数调用的位置

修改this的指向

	### call（）方法

​	语法规则：call(obj, arg1, arg2 ...argN)

​	参数说明：obj：函数内this要指向的对象；arg1...argN：参数列表。

```javascript
var zs = {name: 'zhangsan'}
function f(age) {
    console.log(this.name)
    console.log(age)
}
f(23)  // undefined   23
f.call(zs, 23)  // zhangsan  23
```

	### apply（）方法

​	语法规则：apply(obj, [arg1, arg2 ...argN])

​	参数说明：obj：函数内this要指向的对象；[arg1...argN]：参数列表。

```javascript
var zs = {name: 'zhangsan'}
function f(age.sex) {
    console.log(this.name + age + sex)
}
f.apply(zs, [23, 'nan'])  // zhangsan23nan
```

	### bind（）方法

bind使用时不会调用函数，而是返回一个原函数的拷贝和指定的this值，可以定义一个变量去接收返回的原函数拷贝

```javascript
var obj = {name: 'zhangsan'}
function f(x, y) {
    console.log(x + y)
    console.log(this)
}
f.bind(obj, 5, 6)  // 返回的是原函数拷贝和this指向
var new_f = f.bind(obj, 5, 6)
new_f(); // 输出11和obj{name:'zhangsan'}
```

## 四、eventloop

​	事件循环，是指浏览器或Node的一种解决js单线程运行时不会阻塞的一种机制，也就是我们经常使用异步的原理。

## 五、避免重绘和重排

1. 不在布局信息改变时作DOM查询
2. 使用cssText或者className一次性改变属性
3. 对于多次重排的元素，使其脱离文档流，让它的改变不影响其他元素
4. 使用 fragment（创建虚拟节点对象）document.createDocumentFragment()

## 六、Object assign()方法

​	Object.assign() 方法用于将所有可枚举属性的值从一个或多个源对象分配到目标对象。它将返回目标对象。

​	语法： Object.assign( target, ...sources )

​		其中 target 是目标对象， source 是源对象。

```javascript
var arr1 = {a:1,b:2}
var arr2 = {a:2,b:3}
var arr3 = Object.assign({a:3,b:4}, arr1, arr2)
//输出 {a:2,b:3}
```

​	使用 Object.assign() 可以实现深拷贝。

## 七、防抖和节流

