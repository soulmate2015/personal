# JavaScript原型链

## 构造函数的简单例子：

```javascript
function Person() {

}

const person = new Person();
// person.name => undefined
person.name = 'Tony';
```

在这个例子中，Person 是一个构造函数，我们使用 new 创建了一个实例对象 person


## prototype

每个函数都有一个 prototype 属性：

```javascript
function Person() {

}
Person.prototype.name = 'Tony';

const person1 = new Person();
const person2 = new Person();

// person1.name => 'Tony'
// person2.name => 'Tony'
```

Person 函数的 prototype 属性指向了一个对象，这个对象正是调用该构造函数而创建的实例的原型，也就是 person1 和 person2 的原型。

每个JavaScript对象在创建的时候就会与之关联另一个对象，这个对象就是原型，每一个对象都会从原型"继承"属性。

构造函数和实例原型之间的关系：

<table>
  <tr>
    <td bgcolor=#F0FFFF>
      <img src="https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/prototype1.png">
    </td>
  </tr>
</table>


## `__proto__`

每一个JavaScript对象都有一个 `__proto__` 属性，该属性指向该对象的原型。但也有一个特例 __null__ ，`null` 属于JavaScript中7个基本类型之一，表示没有对象。

可以用一个例子来证明：

```javascript
function Person() {

}
const person = new Person();

// person.__proto__ === Person.prototype => true
```

我们来补充一下刚才的关系：

<table>
  <tr>
    <td bgcolor=#F0FFFF>
      <img src="https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/prototype2.png">
    </td>
  </tr>
</table>


## constructor

实例对象和构造函数都可以指向原型, 原型也可以通过 **constructor** 属性指向关联的构造函数。由于一个构造函数可以生产无数个实例，所以原型无法指向具体的实例。

可以用一个例子来证明：

```javascript
function Person() {

}

// Person.prototype.constructor === Person => true
```

再来更新一下刚才的关系：

<table>
  <tr>
    <td bgcolor=#F0FFFF>
      <img src="https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/prototype3.png">
    </td>
  </tr>
</table>

简单总结一下它们之间的关系：

```javascript
function Person() {

}
const person = new Person();

// person.__proto__ === Person.prototype                => true
// Person.prototype.constructor === Person              => true
// Object.getPrototypeOf(person) === Person.prototype   => true
```


## 实例与原型

当读取实例的属性时，如果找不到，就会查找与对象关联的原型中的属性，如果还查不到，就去找原型的原型，一直找到最顶层为止。

举个例子：

```javascript
function Person() {

}

Person.prototype.name = 'Tony';

const person = new Person();

person.name = 'Daisy';
// person.name => Daisy

delete person.name;
// person.name => Tony
```

当我们删除了 person 的 name 属性时，读取 person.name，从 person 对象中找不到 name 属性就会从 person 的原型也就是 person.`__proto__` ，也就是 Person.prototype 中查找

但是如果在 Person.prototype 上查找中也没有找到呢？


## 原型的原型

原型也是一个对象，既然是对象，我们就可以用最原始的方式创建它:

```javascript
const A = new Object();
```

原型对象也是通过 Object 构造函数生成的，结合之前所述，实例的 `__proto__` 指向构造函数的 prototype ，再更新下关系图：

<table>
  <tr>
    <td bgcolor=#F0FFFF>
      <img src="https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/prototype4.png">
    </td>
  </tr>
</table>


## 原型链

最后我们来看一下 Object.prototype 的原型:

```javascript
// Object.prototype.__proto__             => null
// Object.prototype.__proto__ === null    => true
```

> null 表示“没有对象”，即该处不应该有值。

Object.prototype.__proto__ 的值为 null 跟 Object.prototype 没有原型，其实表达了一个意思。

所以查找属性的时候查到 Object.prototype 就可以停止查找了。

最后一张关系图也可以更新为：

<table>
  <tr>
    <td bgcolor=#F0FFFF>
      <img src="https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/prototype5.png">
    </td>
  </tr>
</table>


思考：

为什么 Array.`__proto__`、String.`__proto__`、Number.`__proto__`、Boolean.`__proto__` 等可以对对应基本数据类型对象的 `__proto__` 属性都指向 Function.prototype ?
