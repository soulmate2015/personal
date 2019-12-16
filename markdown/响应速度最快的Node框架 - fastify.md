# [fastify](https://github.com/fastify/fastify) 响应速度最快的Node框架

## 核心模块

* [fast-json-stringify](https://github.com/fastify/fast-json-stringify) 序列化模块

  速度更快的序列化模块，效率是JSON.stringify()的2X倍。触发V8的性能优化，最终将字符串转化成为一个Buffer。

  ```javascript
  const fastJson = require('fast-json-stringify')
  const stringify = fastJson({
    title: 'Example Schema',
    type: 'object',
    properties: {
      firstName: {
        type: 'string'
      },
      lastName: {
        type: 'string'
      },
      age: {
        description: 'Age in years',
        type: 'integer'
      },
      reg: {
        type: 'string'
      }
    }
  })

  console.log(stringify({
    firstName: 'Matteo',
    lastName: 'Collina',
    age: 32,
    reg: /"([^"]|\\")*"/
  }))
  ```

* [pino](https://github.com/pinojs/pino) 日志模块

  开销非常低的日志记录模块。

  ```javascript
  const logger = require('pino')()

  logger.info('hello world')

  const child = logger.child({ a: 'property' })
  child.info('hello child!')
  ```

  输出：

  ```javascript
  {"level":30,"time":1531171074631,"msg":"hello world","pid":657,"hostname":"Davids-MBP-3.fritz.box","v":1}
  {"level":30,"time":1531171082399,"msg":"hello child!","pid":657,"hostname":"Davids-MBP-3.fritz.box","a":"property","v":1}
  ```

* [find-my-way](https://github.com/delvedor/find-my-way) 路由模块 A crazy fast HTTP router

  一个速度令人疯狂，内部使用高性能[基数树](https://en.wikipedia.org/wiki/Radix_tree)(又名：紧凑前缀树)，支持路由参数，通配符的独立路由模块。

  ```javascript
  const http = require('http')
  const router = require('find-my-way')()

  router.on('GET', '/', (req, res, params) => {
    res.end('{"message":"hello world"}')
  })

  const server = http.createServer((req, res) => {
    router.lookup(req, res)
  })

  server.listen(3000, err => {
    if (err) throw err
    console.log('Server listening on: http://localost:3000')
  })
  ```

* [ajv](https://github.com/epoberezkin/ajv) JSON校验模块

  快速的JSON模式校验模块，可用于node，web端。

  ```javascript
  // ...
  var valid = ajv.validate(schema, data);
  if (!valid) console.log(ajv.errors);
  // ...
  ```
  or
  ```javascript
  // ...
  var valid = ajv.addSchema(schema, 'mySchema').valida('mySchema', data);
  if (!valid) console.log(ajv.errorsText());
  // ...
  ```

  > 执行效率:

  ![执行效率](https://camo.githubusercontent.com/a5a018f4bb73429787af199c0335f5a472f2ca8c/68747470733a2f2f63686172742e676f6f676c65617069732e636f6d2f63686172743f636878743d782c79266368743d626873266368636f3d3736413446422663686c733d322e3026636862683d33322c342c31266368733d36303078343136266368786c3d2d313a253743646a76253743616a762537436a736f6e2d736368656d612d76616c696461746f722d67656e657261746f722537436a73656e25374369732d6d792d6a736f6e2d76616c69642537437468656d69732537437a2d736368656d612537436a73636b253743736b65656d61732537436a736f6e2d736368656d612d6c696272617279253743747634266368643d743a3130302c39382c37322e312c36362e382c35302e312c31352e312c362e312c332e382c312e322c302e372c302e32)
