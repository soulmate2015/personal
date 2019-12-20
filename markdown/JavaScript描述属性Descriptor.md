# JavaScript描述属性

对象的每个属性都有一个描述对象（Descriptor），用于控制该属性的行为。

可以用 `Object.getOwnPropertyDescriptor` 方法获取该属性的描述对象:

```javascript
const someObject = { foo: 123 }
Object.getOwnPropertyDescriptor(someObject, 'foo')

/**
{
  value: 123,                   // 属性值
  writable: true,               // 是否可以写入
  enumerable: true,             // 可枚举性
  configurable: true            // 
}
*/
```

## 对象的描述属性

* ### value

  该属性对应的值，默认为undefined

* ### writable

  能否修改属性的值，如果直接使用字面量定义对象，默认值为true

* ### enumerable

  表示该属性是否可枚举，即是否通过for-in循环或Object.keys()返回属性，如果直接使用字面量定义对象，默认值为true

* ### configurable

  表示能否通过delete删除此属性，能否修改属性的特性，或能否修改把属性修改为访问器属性，如果直接使用字面量定义对象，默认值为true

* ### set

  一个给属性提供 setter 的方法(给对象属性设置值时调用的函数)，如果没有 setter 则为 undefined。该方法将接受唯一参数，并将该参数的新值分配给该属性。默认为 undefined

* ### get

  一个给属性提供 getter 的方法(访问对象属性时调用的函数,返回值就是当前属性的值)，如果没有 getter 则为 undefined。该方法返回值被用作属性值。默认为 undefined


## 修改对象的描述属性

可以通过 `Object.defineProperty()` 或者 `Object.defineProperties()` 修改对象的描述属性。

设置对象某个属性的描述属性：

```javascript
const someObject = new Object();

Object.defineProperty(someObject, 'name', {
  value: '张三',
  writable: true,
  enumerable: true,
  configurable: false,
})

// someObject.name => 张三

delete someObject.name;     // => false

// someObject.name => 张三
```

设置对象的多个属性的描述属性：

```javascript
const someObject = {
  name: '张三'
}

Object.defineProperties(someObject, {
  sex: {
    value: '男'
  },
  age: {
    get: () => {
      return 24
    }
  }
})

// someObject             => { name: '张三', sex: '男' }
// someObject.age         => 24

someObject.sex = '未知';

// someObject.sex         => 男

Object.getOwnPropertyDescriptor(someObject, 'name')
// => {value: "张三", writable: true, enumerable: true, configurable: true}

Object.getOwnPropertyDescriptor(someObject, 'sex')
// => {value: "男", writable: false, enumerable: false, configurable: false}

Object.getOwnPropertyDescriptor(someObject, 'age')
// => {get: ƒ, set: undefined, enumerable: false, configurable: false}

```


## 描述属性的应用

装饰器, 数据双向绑定 等。。

一个简单的装饰器例子：

```javascript
function readonly(target, name, descriptor) {
  descriptor.writable = false;
  return descriptor
}

class Person {
  static some() { return '11111111' }
}

Person.some = () => { return '22222222' }

Person.some();     // => 22222222

class Person {
  @readonly
  static some() { return '11111111' }
}

Person.some = () => { return '22222222' }

Person.some();     // => 11111111

```

一个数据双向绑定的简单例子：

html 部分
```html
<body>
    <p>
        input1=><input type="text" id="input1">
    </p>
    <p>
        input2=>
        <input type="text" id="input2">
    </p>
    <div>
        我每次比input1的值加1=>
        <span id="span"></span>
    </div>
</body>
```

javascript 部分
```javascript
let oInput1 = document.getElementById('input1');
let oInput2 = document.getElementById('input2');
let oSpan = document.getElementById('span');
let obj = {};
Object.defineProperties(obj, {
    val1: {
        configurable: true,
        get: () => {
            oInput1.value = 0;
            oInput2.value = 0;
            oSpan.innerHTML = 0;
            return 0
        },
        set: (newValue) => {
            oInput2.value = newValue;
            oSpan.innerHTML = Number(newValue) ? Number(newValue) : 0
        }
    },
    val2: {
        configurable: true,
        get: () => {
            oInput1.value = 0;
            oInput2.value = 0;
            oSpan.innerHTML = 0;
            return 0
        },
        set: (newValue) => {
            oInput1.value = newValue;
            oSpan.innerHTML = Number(newValue)+1;
        }
    }
})
oInput1.value = obj.val1;
oInput1.addEventListener('keyup', function() {
    obj.val1 = oInput1.value;
}, false)
oInput2.addEventListener('keyup', function() {
    obj.val2 = oInput2.value;
}, false)
```

