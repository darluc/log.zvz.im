title: 用 NodeJS 打造影院微服务并部署到 docker 上 — Part 3
date: 2017-10-17 14:39:51
tags:
- Nodejs
---

>大家好，本文是「使用 NodeJS 构建影院微服务」系列的第三篇文章。此系列文章旨在展示如何使用 ES6，¿ES7 …8?，和 expressjs 构建一个 API 应用，如何连接 MongoDB 集群，怎样将其部署于 docker 容器中，以及模拟微服务运行于云环境中的情况。

### ## 以往章节快速回顾

* 我们讲了**什么是微服务**，探讨了**微服务**的**利**与**弊**
* 我们定义了**影院微服务架构**
* 我们设计并实现了**电影服务**和**影院目录服务**
* 我们实现了这些服务的 API 接口，并对这些接口做了**单元测试**
* 我们对运行于**Docker**中的**服务**进行了集成测试
* 我们讨论了**微服务安全**并使其适配了 **HTTP/2 协议**
* 我们对**影院目录服务**进行了**压力测试**


如果你没有阅读之前的章节，那么很有可能会错一些有趣的东西 🤘🏽，下面我列出前两篇的链接，方便你有兴趣的话可以看一下👀。

- [用 NodeJS 打造影院微服务并部署到 docker 上 — Part 1](https://log.zvz.im/2017/05/24/nodejs-cinema-microservice-part1/)
- [用 NodeJS 打造影院微服务并部署到 docker 上 — Part 2](https://log.zvz.im/2017/07/17/nodejs-cinema-microservice-part2/)

在之前的章节中，我们已经完成了以下架构图中的上层部分，接着从本章起，我们要开始图中下层部分的开发了。

![](https://ws1.sinaimg.cn/large/7327fe71gy1fki9tkkgh7j20xh0liad7.jpg)

到目前为止，我们的终端用户已经能够在影院看到电影首映信息，选择影院并下单买票。本章我们会继续构建**影院架构**，并探索**订票服务**内部是如何工作的，跟我一起学点有趣的东西吧。

我们将使用到以下技术：

- NodeJS version 7.5.0
- MongoDB 3.4.1
- Docker for Mac 1.13

要跟上本文的进度有以下要求：

- 已经完成[上一篇文章](https://log.zvz.im/2017/05/24/nodejs-cinema-microservice-part1/)中的例子代码

如果你还没有完成这些代码，我已经将代码传到了 github 上，你可以直接使用[代码库](https://github.com/Crizstian/cinema-microservice/tree/step-1)分支 **step-2**。

## # NodeJS 中的依赖注入

至今为止我们已经构建了两套微服务的 API 接口，不过都没有遇到太多的配置和开发工作，这是由这些微服务自身的特性和简单性决定的。不过这一次，在**订票服务**中，我们会看到更多与其它服务之间的交互，因为这个服务的实现依赖项更多，为了防止写出一团乱麻似的代码，作为好的开发者，我们需要遵循某种设计模式，为此我们将会探究什么是**“依赖注入”**。

想要达成良好的设计模式，我们必须很好地理解并应用 **S.O.L.I.D 原则**，我之前写过一篇与之相关的 javascript 的文章，有空你可以看一下🤓，主要讲述了这些原则是什么并且我们可以从中获得哪些好处。

[S.O.L.I.D The first 5 principles of Ojbect Oriented Design with Javascritp](https://medium.com/@cramirez92/s-o-l-i-d-the-first-5-priciples-of-object-oriented-design-with-javascript-790f6ac9b9fa)

为什么依赖注入如此重要？因为它能给我们带来以下开发模式中的三大好处：

* **解耦**：依赖注入可减少模块之间的耦合性，使其更易于维护。
* **单元测试**：使用依赖注入，可使对于每个模块的单元测试做得更好，代码的 bug 也会较少。
* **快速开发**：利用依赖注入，在定义了接口之后，可以更加容易地进行分工合作而不会产生冲突。

至今为此开发的微服务中，我们曾在 `index.js` 文件中使用到了**依赖注入**

```javascript
// more code

mediator.on('db.ready', (db) => {
  let rep
  // here we are making DI to the repository
  // we are injecting the database object and the ObjectID object
  repository.connect({
    db, 
    ObjectID: config.ObjectID
  })
  .then(repo => {
      console.log('Connected. Starting Server')
      rep = repo
      // here we are also making DI to the server
      // we are injecting serverSettings and the repo object
      return server.start({
        port: config.serverSettings.port,
        ssl: config.serverSettings.ssl,
        repo
      })
    })
    .then(app => {
      console.log(`Server started succesfully, running on port: ${config.serverSettings.port}.`)
      app.on('close', () => {
        rep.disconnect()
      })
    })
})

// more code
```

在 `index.js` 文件中我们使用了手动的依赖注入，因为没有必要做得更多。不过在**订票服务**中，我们将需要一种更好地依赖注入方式，为了厘清个中缘由，在开始构建 API 接口之前，我们要先弄清楚**订票服务**需要完成哪些任务。

* 订票服务需要一个订票对象和一个用户对象，而且在进行订票动作时，我们首先要验证这些对象的有效性。
* 验证有效性之后，我们就可以继续流程，开始买票了。
* 订票服务需要用户的信用卡信息，通过**支付服务**，来完成购票动作。
* 扣款成功后，我们需要通过**通知服务**发送通知。
* 我们还需要为用户生成电影票，并将电影票和订单号信息发送给用户。

所以这次我们的开发任务变得相对重了一些，相应地代码也会变多，这也是我们需要一个单一依赖注入来源的原因，因为我们需要做更多的功能开发。

## # 构建微服务

首先我们来看一下**订票服务**的 **RAML** 文件。

```yaml
#%RAML 1.0
title: Booking Service
version: v1
baseUri: /

types:
  Booking:
    properties:
      city: string
      cinema: string
      movie: string
      schedule: datetime
      cinemaRoom: string
      seats: array
      totalAmount: number


  User:
    properties:
      name: string
      lastname: string
      email: string
      creditcard: object
      phoneNumber?: string
      membership?: number

  Ticket:
    properties:
      cinema: string
      schedule: string
      movie: string
      seat: string
      cinemaRoom: string
      orderId: string


resourceTypes:
  GET:
    get:
      responses:
        200:
          body:
            application/json:
              type: <<item>>

  POST:
    post:
      body:
        application/json:
          type: <<item>>
          type: <<item2>>
      responses:
        201:
          body:
            application/json:
              type: <<item3>>


/booking:
  type:   { POST: {item : Booking, item2 : User, item3: Ticket} }
  description: The booking service need a Booking object that contains all
    the needed information to make a purchase of cinema tickets. Needs a user information to make the booking succesfully. And returns a ticket object.

  /verify/{orderId}:
    type:  { GET: {item : Ticket} }
    description: This route is for verify orders, and would return all the details of a specific purchased by orderid.
```

我们定义了三个模型对象，**Booking** 、**User** 以及 **Ticket** 。由于这是系列文章中第一次使用到 **POST** 请求，因此还有一项 NodeJS 的**最佳实践**我们还没有使用过，那就是**数据验证**。在[“Build beautiful node API's“](https://medium.com/software-engineering/beautiful-node-apis-eaf0b636cbe#.t8kdvpkcv) 这篇文章中有一句很好的表述：

> 一定，一定，一定要验证输入（以及输出）的数据。有 joi 以及 express-validator 等模块可以帮助你优雅地完成数据净化工作。— Azat Mardan

现在我们可以开始开发**订票服务**了。我们将使用与上一章相同的项目结构，不过会稍微做一点点改动。让我们不再纸上谈兵，撸起袖子开始编码！ 👩🏻‍💻👨🏻‍💻。

首先我们在 `/src` 目录下新建一个 `models` 目录

```shell
booking-service/src $ mkdir models

# Now let's move to the folder and create some files

booking-service/src/models $ touch user.js booking.js ticket.js

# Now is moment to install a new npm package for data validation

npm i -S joi --silent
```

然后我们开始编写数据结构验证对象了，**MonogDB**也有内置的验证对象，不过这里需要验证的是数据对象的完整性，所以我们选择使用 joi，而且 joi 也允许我们同时进行数据验证，我们就由 **booking.model.js** 开始，然后是 **ticket.model.js**， 最后是 **user.model.js**。

```javascript
const bookingSchema = (joi) => ({
  bookingSchema: joi.object().keys({
    city: joi.string(),
    schedule: joi.date().min('now'),
    movie: joi.string(),
    cinemaRoom: joi.number(),
    seats: joi.array().items(joi.string()).single(),
    totalAmount: joi.number()
  })
})

module.exports = bookingSchema
```

```javascript
const ticketSchema = (joi) => ({
  ticketSchema: joi.object().keys({
    cinema: joi.string(),
    schedule: joi.date().min('now'),
    movie: joi.string(),
    seat: joi.array().items(joi.string()).single(),
    cinemaRoom: joi.number(),
    orderId: joi.number()
  })
})

module.exports = ticketSchema
```

```javascript
const userSchema = (joi) => ({
  userSchema: joi.object().keys({
    name: joi.string().regex(/^[a-bA-B]+/).required(),
    lastName: joi.string().regex(/^[a-bA-B]+/).required(),
    email: joi.string().email().required(),
    phoneNumber: joi.string().regex(/^(\+0?1\s)?\(?\d{3}\)?[\s.-]\d{3}[\s.-]\d{4}$/),
    creditCard: joi.string().creditCard().required(),
    membership: joi.number().creditCard()
  })
})

module.exports = userSchema
```

如果你不是太了解 `joi` ，你可以去 github 上学习一下它的文档：[文档链接](https://github.com/hapijs/joi/blob/v10.2.0/API.md)

接下来我们编写模块的 `index.js` 文件，使这些校验方法暴露出来：

```javascript
const joi = require('joi')
const user = require('./user.model')(joi)
const booking = require('./booking.model')(joi)
const ticket = require('./ticket.model')(joi)

const schemas = Object.create({user, booking, ticket})

const schemaValidator = (object, type) => {
  return new Promise((resolve, reject) => {
    if (!object) {
      reject(new Error('object to validate not provided'))
    }
    if (!type) {
      reject(new Error('schema type to validate not provided'))
    }

    const {error, value} = joi.validate(object, schemas[type])

    if (error) {
      reject(new Error(`invalid ${type} data, err: ${error}`))
    }
    resolve(value)
  })
}

module.exports = Object.create({validate: schemaValidator})
```

我们所写的这些代码应用了**SOLID 原则**中的**单一责任原则**，每个模型都有自己的校验方法，还应用了**开放封闭原则**，每个结构校验函数都可以对任意多的模型对象进行校验，接下来看看如何为这些模型编写测试代码。

```javascript
/* eslint-env mocha */
const test = require('assert')
const {validate} = require('./')

console.log(Object.getPrototypeOf(validate))

describe('Schemas Validation', () => {
  it('can validate a booking object', (done) => {
    const now = new Date()
    now.setDate(now.getDate() + 1)

    const testBooking = {
      city: 'Morelia',
      cinema: 'Plaza Morelia',
      movie: 'Assasins Creed',
      schedule: now,
      cinemaRoom: 7,
      seats: ['45'],
      totalAmount: 71
    }

    validate(testBooking, 'booking')
      .then(value => {
        console.log('validated')
        console.log(value)
        done()
      })
      .catch(err => {
        console.log(err)
        done()
      })
  })

  it('can validate a user object', (done) => {
    const testUser = {
      name: 'Cristian',
      lastName: 'Ramirez',
      email: 'cristiano@nupp.com',
      creditCard: '1111222233334444',
      membership: '7777888899990000'
    }

    validate(testUser, 'user')
      .then(value => {
        console.log('validated')
        console.log(value)
        done()
      })
      .catch(err => {
        console.log(err)
        done()
      })
  })

  it('can validate a ticket object', (done) => {
    const testTicket = {
      cinema: 'Plaza Morelia',
      schedule: new Date(),
      movie: 'Assasins Creed',
      seats: ['35'],
      cinemaRoom: 1,
      orderId: '34jh1231ll'
    }

    validate(testTicket, 'ticket')
      .then(value => {
        console.log('validated')
        console.log(value)
        done()
      })
      .catch(err => {
        console.log(err)
        done()
      })
  })
})
```

然后，我们要看的代码文件是 `api/booking.js` ，我们将会遇到更多的麻烦了，**¿ 为什么呢 ?**，因为这里我们将会与**两个外部服务**进行交互：**支付服务**以及**通知服务**，而且这类交互会引发我们重新思考微服务的架构，并会牵扯到被称作**时间驱动数据管理**以及 **CQRS** 的课题，不过我们将把这些课题留到之后的章节再进行讨论，避免本章变得过于复杂冗长。所以，本章我们先与这些服务进行简单地交互。

```javascript
'use strict'
const status = require('http-status')

module.exports = ({repo}, app) => {
  app.post('/booking', (req, res, next) => {
    
    // we grab the dependencies need it for this route
    const validate = req.container.resolve('validate')
    const paymentService = req.container.resolve('paymentService')
    const notificationService = req.container.resolve('notificationService')

    Promise.all([
      validate(req.body.user, 'user'),
      validate(req.body.booking, 'booking')
    ])
    .then(([user, booking]) => {
      const payment = {
        userName: user.name + ' ' + user.lastName,
        currency: 'mxn',
        number: user.creditCard.number,
        cvc: user.creditCard.cvc,
        exp_month: user.creditCard.exp_month,
        exp_year: user.creditCard.exp_year,
        amount: booking.amount,
        description: `
          Tickect(s) for movie ${booking.movie},
          with seat(s) ${booking.seats.toString()}
          at time ${booking.schedule}`
      }

      return Promise.all([
        // we call the payment service
        paymentService(payment),
        Promise.resolve(user),
        Promise.resolve(booking)
      ])
    })
    .then(([paid, user, booking]) => {
      return Promise.all([
        repo.makeBooking(user, booking),
        repo.generateTicket(paid, booking)
      ])
    })
    .then(([booking, ticket]) => {
      // we call the notification service
      notificationService({booking, ticket})
      res.status(status.OK).json(ticket)
    })
    .catch(next)
  })

  app.get('/booking/verify/:orderId', (req, res, next) => {
    repo.getOrderById(req.params.orderId)
      .then(order => {
        res.status(status.OK).json(order)
      })
      .catch(next)
  })
}
```

你可以看到，这里我们使用到了 expressjs 的**中间件**：**container**，并将其作为我们所用到的依赖项的唯一真实来源。

不过包含这些依赖项的 **container** 是从何而来呢？

我们现在对项目结构做了一点调整，主要是对 `config` 目录的调整，如下：

```
. 
|-- config 
|   |-- db 
|   |   |-- index.js 
|   |   |-- mongo.js 
|   |   `-- mongo.spec.js 
|   |-- di 
|   |   |-- di.js 
|   |   `-- index.js 
|   |-- ssl
|   |   |-- certificates 
|   |   `-- index.js
|   |-- config.js
|   |-- index.spec.js 
|   `-- index.js
```

在 `config/index.js` 文件包含了几乎所有的配置文件，包括**依赖注入**服务：

```javascript
const {dbSettings, serverSettings} = require('./config')
const database = require('./db')
const {initDI} = require('./di')
const models = require('../models')
const services = require('../services')

const init = initDI.bind(null, {serverSettings, dbSettings, database, models, services})

module.exports = Object.assign({}, {init})
```

上面的代码中我们看到些不常见的东西，这里提出来给大家看看：

```javascript
initDI.bind(null, {serverSettings, dbSettings, database, models, services})
```

这行代码到底做了什么呢？之前我提到过我们要配置**依赖注入**，不过这里我们做的事情叫作**控制反转**，的确这种说法太过于技术化了，甚至有些夸张，不过一旦你理解了之后就很容易理解。

所以我们的**依赖注入**函数不需要知道依赖项来自哪里，它只要注册这些依赖项，使得应用能够使用即可，我们的 `di.js` 看起来如下：

```javascript
const { createContainer, asValue, asFunction, asClass } = require('awilix')

function initDI ({serverSettings, dbSettings, database, models, services}, mediator) {
  mediator.once('init', () => {
    mediator.on('db.ready', (db) => {
      const container = createContainer()
      
      // loading dependecies in a single source of truth
      container.register({
        database: asValue(db).singleton(),
        validate: asValue(models.validate),
        booking: asValue(models.booking),
        user: asValue(models.booking),
        ticket: asValue(models.booking),
        ObjectID: asClass(database.ObjectID),
        serverSettings: asValue(serverSettings),
        paymentService: asValue(services.paymentService),
        notificationService: asValue(services.notificationService)
      })
      
      // we emit the container to be able to use it in the API
      mediator.emit('di.ready', container)
    })

    mediator.on('db.error', (err) => {
      mediator.emit('di.error', err)
    })

    database.connect(dbSettings, mediator)

    mediator.emit('boot.ready')
  })
}

module.exports.initDI = initDI
```

如你所见，我们使用了一个名为 `awilix` 的 npm 包用作依赖注入，awilix 实现了 nodejs 中的依赖注入机制（我目前正在试用这个库，这里使用它是为了是例子看起来更加清晰），要安装它需要执行以下指令：

```shell
npm i -S awilix --silent
```

现在我们的主 `index.js` 文件看起来就像这样：

```javascript
'use strict'
const {EventEmitter} = require('events')
const server = require('./server/server')
const repository = require('./repository/repository')
const di = require('./config')
const mediator = new EventEmitter()

console.log('--- Booking Service ---')
console.log('Connecting to movies repository...')

process.on('uncaughtException', (err) => {
  console.error('Unhandled Exception', err)
})

process.on('uncaughtRejection', (err, promise) => {
  console.error('Unhandled Rejection', err)
})

mediator.on('di.ready', (container) => {
  repository.connect(container)
    .then(repo => {
      container.registerFunction({repo})
      return server.start(container)
    })
    .then(app => {
      app.on('close', () => {
        container.resolve('repo').disconnect()
      })
    })
})

di.init(mediator)

mediator.emit('init')
```

现在你能看到，我们使用的包含所有依赖项的真实唯一来源，可通过 request 的 container 属性访问，至于我们怎样通过 expressjs 的**中间件**进行设置的，如之前提到过的，其实只需要几行代码：

```javascript
const express = require('express')
const morgan = require('morgan')
const helmet = require('helmet')
const bodyparser = require('body-parser')
const cors = require('cors')
const spdy = require('spdy')
const _api = require('../api/booking')

const start = (container) => {
  return new Promise((resolve, reject) => {
    
    // here we grab our dependencies needed for the server
    const {repo, port, ssl} = container.resolve('serverSettings')

    if (!repo) {
      reject(new Error('The server must be started with a connected repository'))
    }
    if (!port) {
      reject(new Error('The server must be started with an available port'))
    }

    const app = express()
    app.use(morgan('dev'))
    app.use(bodyparser.json())
    app.use(cors())
    app.use(helmet())
    app.use((err, req, res, next) => {
      if (err) {
        reject(new Error('Something went wrong!, err:' + err))
        res.status(500).send('Something went wrong!')
      }
      next()
    })
    
    // here is where we register the container as middleware
    app.use((req, res, next) => {
      req.container = container.createScope()
      next()
    })
    
    // here we inject the repo to the API, since the repo is need it for all of our functions
    // and we are using inversion of control to make it available
    const api = _api.bind(null, {repo: container.resolve('repo')})
    api(app)

    if (process.env.NODE === 'test') {
      const server = app.listen(port, () => resolve(server))
    } else {
      const server = spdy.createServer(ssl, app)
        .listen(port, () => resolve(server))
    }
  })
}

module.exports = Object.assign({}, {start})
```

基本上，我们只是将 **container** 对象附加到了 expressjs 的 **req 对象**上，这样 expressjs 的所有路由上都能访问到它了。如果你想更深入地了解 expressjs 的中间件是如何工作的，你可以点击[这个链接查看 expressjs 的文档](http://expressjs.com/en/guide/using-middleware.html)。

常言道好事多磨，最后让我们来看看 `repository.js` 文件：

```javascript

'use strict'
const repository = (container) => {
  // we get the db object via the container
  const {db} = container.resolve('database')

  const makeBooking = (user, booking) => {
    return new Promise((resolve, reject) => {
      // payload to be insterted to the booking collection 
      const payload = {
        city: booking.city,
        cinema: booking.cinema,
        book: {
          userType: (user.membership) ? 'loyal' : 'normal',
          movie: {
            title: booking.movie.title,
            format: booking.movie.format,
            schedule: booking.schedule
          }
        }
      }

      db.collection('booking').insertOne(payload, (err, booked) => {
        if (err) {
          reject(new Error('An error occuered registring a user booking, err:' + err))
        }
        resolve(booked)
      })
    })
  }

  const generateTicket = (paid, booking) => {
    return new Promise((resolve, reject) => {
      // payload of ticket
      const payload = Object.assign({}, {booking, orderId: paid._id})
      db.collection('tickets').insertOne(payload, (err, ticket) => {
        if (err) {
          reject(new Error('an error occured registring a ticket, err:' + err))
        }
        resolve(ticket)
      })
    })
  }

  const getOrderById = (orderId) => {
    return new Promise((resolve, reject) => {
      const ObjectID = container.resolve('ObjectID')
      const query = {_id: new ObjectID(orderId)}
      const response = (err, order) => {
        if (err) {
          reject(new Error('An error occuered retrieving a order, err: ' + err))
        }
        resolve(order)
      }
      db.collection('booking').findOne(query, {}, response)
    })
  }

  const disconnect = () => {
    db.close()
  }

  return Object.create({
    makeBooking,
    getOrderById,
    generateTicket,
    disconnect
  })
}

const connect = (container) => {
  return new Promise((resolve, reject) => {
    if (!container.resolve('database')) {
      reject(new Error('connection db not supplied!'))
    }
    resolve(repository(container))
  })
}

module.exports = Object.assign({}, {connect})
```

 `repository.js` 文件并无特别之处，除了这可能是我们在系列文章中，第一次使用到 `insertOne()` 方法。不过我想指出一件事情，特别是在 `makeBooking()` 方法中，你可以看到 payload 使用了整个数据模型对象，为什么我们要这样用？这样是否会有太多的冗余信息？

没错，这样会造成数据冗余，同时也不是最好的实现方式，但是我们确实有理由这样做，不过得等到下回我才会告诉你为什么，因为一些有趣的事情即将发生。

如果你很好奇，那么我可以先给你留一点点提示😁

```
   ----------------------------------------
  |                                        |
  |                                        v
  |                       Jane  ------(went to)----------
  |                         |                            |
  |                         | (loyal vistor)             |
  |                         v                            v
 Joe --(normal visitor)--> Movie Name <--(displayed)-- Plaza Morelia
                             |                           |
                             |  (format)                 | (city)
                             v                           v
                            4DX                       Morelia
```

前面提到我们会与两个外部服务进行交互，先稍微看一下这些外部服务需要做什么。

```javascript
# for the payment service we will need to implement something like the following

module.exports = (paymentOrder) => {
  return new Promise((resolve, reject) => {
    supertest('url to the payment service')
      .get('/makePurchase')
      .send({paymentOrder})
      .end((err, res) => {
        if (err) {
          reject(new Error('An error occured with the payment service, err: ' + err))
        }
        resolve(res.body.payment)
      })
  })
}

# since we haven't made the payment service yet, let's make something simple to fulfill the article example, like the following    

module.exports = (paymentOrder) => {
  return new Promise((resolve, reject) => {
    resolve({orderId: Math.floor((Math.random() * 1000) + 1)})
  })
}

# for the notification service, at the moment we don't need any information from this service we will not implement it, this service will have the task for sending an email, sms or another notification, but we will make this service in the next chapter.
```

到这里我们就完成了本章微服务的构建工作，现在可以使用以下指令，执行代码库中的文件：

```shell
$ bash < start_service
```

以保证我们的微服务在 docker 容器中完整可用，并开始**集成测试**。

# 本章回顾

我们做了什么...

如果你看了之前的几篇文章，那么我们已经有了一个如下的系统架构图：

![](https://ws1.sinaimg.cn/large/7327fe71gy1fkiq8xdf42j212e0xijva.jpg)

你可以看到系统已经基本成形了，只是某些部分还是有点不对，那就是在 **worker1** 和 **worker2** 中我们没有运行任何微服务。那是因为我们还没有在这些 **docker-machines** 中创建任何的服务，不过以后会去做的。

在**影院微服务架构**中，我们基本上完成了下图中的部分：

![](https://ws1.sinaimg.cn/large/7327fe71ly1fkiqgvjnjhj20r909dt9w.jpg)

我们刚刚完成了**订票服务**，而且实现了简易版的**支付服务**和**通知服务**。

在本章中我们都做了什么事情¿ 🤔 ?，我们学习了**依赖注入**，接触了一点**SOLID原则**和**控制反转**的概念，还使用 **NodeJS** 完成了对微服务的第一个**POST 请求**，我们还学习到如何使用**joi**库进行对象和数据的校验。

我们已经学习了许多**NodeJS**开发的代码，不过我们仍然有很多可做和可学习的事情，这儿仅仅只是其中一小部分而已。希望它展示出了一些有趣而有用的东西，使你在工作中能够运用到 **Docker 和 NodeJS** 的相关技术。



翻译自：[Build a NodeJS cinema booking microservice and deploying it with docker (part 3)](https://medium.com/@cramirez92/build-a-nodejs-cinema-booking-microservice-and-deploying-it-with-docker-part-3-9c384e21fbe0)



[系列文章 - Part 1](https://log.zvz.im/2017/05/24/nodejs-cinema-microservice-part1/)
[系列文章 - Part 2](https://log.zvz.im/2017/07/17/nodejs-cinema-microservice-part2/)