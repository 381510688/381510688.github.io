---
title: 编写高质量代码：改善JavaScript程序建议--面向对象编程
date: 2017-05-19 18:44:34
categories: JavaScript
tags: [JavaScript,编写高质量代码]
---

> JavaScript是基于对象的弱类型语言，它是以对象为基础，以函数为模型，以原型为继承机制的开发模式。

## 建议1：参照Object构造体系分析prototype机制

​	**对象（Object）是没有原型的，只有构造函数拥有原型，而构造类的实例对象能够通过prototype属性访问原型对象。**prototype表示类的原型，就是构造类拥有的原始成员。构造函数的prototype属性存储着一个引用对象指针，该指针指向一个原型对象。

​	所有的函数在其定义时就已经自动创建和初始化好了prototype属性，这个初始化好的prototype属性指向一个只包含一个constructor属性的对象，并且这个constructor属性指向这个function自身。

```
var obj = new Object()
console.log(obj.constructor); // function Object() { [native code] }
```

​	当构造函数实例化后，所有实例对象都可以访问构造函数的原型成员。**如果在原型对象中声明一个成员，则所有实例对象都可以共享它。**

​	Object与Function之间关系非常微妙，它们都是高度抽象的类型，互为对方的实例。

```javascript
console.log(Object instanceof Function); // true
console.log(Function instanceof Object); // true
```

![类型、原型和对象实例之间的关系](/images/类型、原型和对象实例之间的关系.png)

​	原型关系是一种动态的关系，如果添加一个新的属性到原型中，那么该属性会立即被所有基于该原型创建的对象继承。

```javascript
function P(x){this.x = x;}
var p1 = new P("p1");
var p2 = new P("p2");
P.prototype.y = 1;
console.log(p1.x, p1.y); // p1 1
console.log(p2.x, p1.y); // p2 1
```

​	![原型属性与本地特性之间的关系](/images/原型属性与本地特性之间的关系.png)

## 建议2：不要直接检索对象属性值

```javasc
var modelValue = obj.obj_1.model; // TypeError
var modelValue = obj.obj_1 && obj.obj_1.model; // undefined
```

稳妥起见，可以通过&&运算符避免上述错误！

## 建议3：防止原型反射

使用for...in语句，可以遍历对象中所有属性（包括原型）；`Object.keys()`只遍历自身可枚举属性。

- 使用`hasOwnProperty`方法过滤原型属性
- 使用`typeof`运算符排除方法函数

```javascript
for(var name in obj) {
	if(obj.hasOwnProperty(name) && typeof obj[name] !== 'function') {
      	console.log(name, obj[name]);
	}
}
```

## 建议4：谨慎处理对象的Scope

​	在JavaScript中，function是作用于词法范围而不是动态运行范围的（词法作用域）。

​	当function作为对象的方法运行是，this是该对象的引用；如果该function没有作为对象的方法，则this代表全局对象。

```javascript
var a = 1;
function f1() {
	var a  = 2;
	function f2() {
		console.log(this.a); // this指向window
	} 
	f2();
}
f1(); // 1
```

​	解决上述问题的通常做法是在将外部function的this值保存到某变量中，在内部函数中更使用。

```javascript
var obj = {
	myVal: 'hello',
	fn: function() {
		var self = this;
		function inner() {
			console.log(self.myVal);
		}
		inner();
	}
}
obj.fn();  // hello
```

​	在JSON字符串中加上一对括号，这样做可以迫使eval方法在评估JavaScript代码时强制作为表达式执行从而得到JSON对象，而不是作为语句执行。

```javascript
var JSONString = '{"name":"ligang","age":27}';
var JSONObject = eval("(" + JSONString + ")");
console.log(JSONObject.name);
```

注意：`eval`和`window.eval`

```javascript
function testEvalScope() {
	eval('var a = 1');
	console.log(a);
}
testEvalScope();
console.log(typeof a); // undefined
```

```javascript
function testEvalScope() {
	window.eval('var a = 1');
	console.log(a);
}
testEvalScope();
console.log(typeof a); // number
```

## 建议5：this是动态指针，不是静态引用

​	this指向的对象是由this所在执行域（运行环境）决定的，而不是由this所在的定义域决定的。它始终指向当前调用对象。在JavaScript中类似指针特性的标识还有如下3个：

- callee：函数的参数集合包含的一个静态指针，它始终指向参数集合所属的函数；
- prototype：函数包含的一个半静态指针，在默认状态下它始终指向函数附带的原型对象，不过可以改变这个指针，使它指向其他对象；
- constructor：对象包含的一个指针，它始终指向创建该对象的构造函数。

（1）函数的引用和调用

引用函数能够改变函数的执行作用域，而调用函数不会改变函数的执行作用域。

```javascript
var obj = {
    name: '对象obj',
    f: function() {
        return this;
    }
};
obj.o1 = {
    name: '对象o1',
    me: obj.f   // 引用对象obj的方法f
};
obj.o2 = {
    name: '对象o2',
    me: obj.f() // 调用对象obj的方法f
};
console.log(obj.o1.me().name); // 对象o1
console.log(obj.o2.me.name);   // 对象obj
```

（2）call和apply

call、apply可以直接改变执行函数的作用域。

```javascript
function f() {
    if(this.constructor === arguments.callee) console.log('实例对象');
    else if(this === window) console.log('windwo对象');
    else console.log(`其他对象 constructor = ${this.constructor}`);
}

f();        // windwo对象
new f();    // new f()
f.call(1);  // 其他对象 constructor = function Number() { [native code] }
```

（3）异步调用

通过事件机制或定时器来延迟或调整函数的调用时间和时机。由于**调用函数的执行作用域不再是原来定义的作用域**，所有函数中this总是指向发生该事件的对象或window对象。

```javascript
var obj = {
    value: 123,
    f: function() {
        console.log(this.value, arguments);
    }
};

var btn = document.getElementById('btn');
btn.addEventListener('click', obj.f); // 空
btn.addEventListener('click', obj.f.bind(obj, event)); // 123
```

```javascript
var obj = {
	f: function() {
		if(this == obj) console.log("this = obj");
		if(this == window) console.log("this = window");
	}
}
obj.f(); // this = obj
setTimeout(obj.f, 1000); // this = window
setTimeout(function(){
  obj.f.call(obj)
}, 1000); // this = obj
setTimeout(obj.f.bind(obj), 1000); // // this = obj
```

## 建议6：使用享元类

享元类就是类的类型，即创建类型的类。享元类能够接受类作为参数，即享元类操作的是类，返回类。

```javascript
function O(x) {
    return function() {
        this.x = x;
        this.get = function() {
            console.log(this.x);
        }
    }
}
var o = new O(1);
var f = new o();
f.get(); // 1
```

如果构造函数有返回值，并且返回值是引用类型，那么经过new运算符计算后，返回的再不是构造函数自身对应的实例对象，而是构造函数包含的返回值（即引用类型值）。

```javascript
function F() {
	this.x = 1;
	return function() {
		return this.x;
	}
}
F.prototype.y = function() {
	console.log("call y");
}

var f = new F();
console.log(f.x);	// undefined，构造函数有新返回值
console.log(F()());	// 1，this指向window
console.log(f.y()); // TypeError，无当前成员
```