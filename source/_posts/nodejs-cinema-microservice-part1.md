title: 用 NodeJS 打造影院微服务并部署到 docker 上 — Part 1
date: 2017-5-24 21:58:41
tags:
- Nodejs
---

![](https://img.zvz.im/imgs/2019/06/1212f2b3d00c4c80.png)

本文是「使用 NodeJS 构建影院微服务」系列的第一篇。此系列将会讲述了如何构建**NodeJS 微服务**并将它们部署到**Docker Swarm 集群**上。

本文将向你展示，如何构建**微服务**，以及如何将其部署到**Docker**容器中。我们会完成一个简单的 NodeJS 服务，并以 MongoDB 作为后端存储。

本文将使用到以下技术：

* NodeJS 7.2.0 版本
* MongoDB 3.4.1
* Docker for Mac 1.12.6

为了理解文中内容，你最好了解以下知识：

* NodeJS 的基础知识
* Docker 的基础知识（并且已安装了 docker）
* MongoDB 的基础知识（并且有可运行的数据库服务）

我强烈建议大家参考我以前的文章[「如何在 Docker 上部署 mongoDB 集群」](https://medium.com/@cramirez92/how-to-deploy-a-mongodb-replica-set-using-docker-6d0b9ac00e49)，部署好数据库服务并将其运行起来。
<!--more-->
## # 什么是微服务

> 微服务是一个独立可用的单元，它可以与其它微服务一起，构成一个大的应用系统。通过将一个应用划分为更小的单元，可使得这些小的单元更加独立、易于部署、且易于扩展，而且这些小单元可以由不同团队的开发，使用不同语言进行开发，并且单独进行测试。—— [Max Stoiber](https://medium.com/@cramirez92/how-to-deploy-a-mongodb-replica-set-using-docker-6d0b9ac00e49)
>
> 微服务架构意味着你的系统由许多小的、独立的应用组成。它们运行在自己的内存空间中，而且独立部署，能够扩展部署到多台机器上。—— [Eric Elliot](https://medium.com/javascript-scene/10-interview-questions-every-javascript-developer-should-know-6fa6bdf5ad95#.gl46xectt)

### 微服务的优点

* 应用启动更快，开发效率更高，部署也更加快速。
* 每个服务可以独立于其它服务进行部署——这样更容易部署新版本
* 更容易组织大规模开发，而且有性能优势。
* 可以避免与技术栈长期捆绑。当开发新服务时，你可以选用新的技术栈。
* 微服务一般都有更好的组织结构，因为每个微服务都有特定的使命，且与其它服务无关。
* 无耦合的服务更容易重新组合配置，可以服务于不同的应用系统（比如，同时服务于 web 客户端以及公共 API）。

### 微服务的缺点

* 开发者必须处理分布式系统带来的额外复杂性。
* 部署更加复杂。在生产环境中，部署并管理一个由许多不同服务组成的系统，操作起来更复杂。
* 在你构建微服务架构的过程中，可能会发现许多在设计初期未能预料到的功能点被切割分散到了各处。

## # 如何用 NodeJS 构建微服务

**微服务**可以利用许多**简单，目的单一，易于使用的**组件构建应用，使得软件质量更好，迭代速度更快。甚至已有的整体架构系统也可以采用[微服务模式](https://blog.risingstack.com/why-you-should-start-using-microservices/)进行转换，不过我们的问题是如何用 **NodeJS** 来构建一个微服务？

Javascript 是当前最流行编程语言，拥有丰富的开源模块生态系统。 对于一个微服务来说，我们想要构建一套 **API**，我应该用哪些模块，库，或者框架呢？我在 Quora 上搜索到了一个类似的问题：[构建 RESTful api 应该使用哪个 Node.js 框架？](https://www.quora.com/Which-Node-js-framework-is-better-for-building-a-RESTful-api)

问题的答案中有一位用户给出了一条非常有用的答案，他提供了一个 NB 的[网站](http://nodeframework.com/)，里面展示了我们可以用来构建 API 的所有框架和库，这样你就可以自己做选择了。本文中我们将使用 **ExpressJS** 来构建我们的 API 和微服务。

现在我们不再空谈，开始动手编码，学习如何解决这个实际问题吧 👨🏻‍💻👨🏼‍🎨🖥。

## # 我们的微服务架构

![](https://img.zvz.im/imgs/2019/06/3a80c5deba7da702.png)

假设我们在 **Cinépolis**（一个墨西哥电影院）的 IT 部门工作。他们派给我们一个任务，让我们重构票务和零售店系统，将原有的一体化系统改为微服务架构。

作为「使用 NodeJS 构建影院微服务」系列的第一部分，我们将关注点放在**电影目录服务(movies catalog service)**上。

在架构图中，我们可以看到有三种不同的设备使用到微服务。POS（售卖点），手机/平板，以及电脑。POS 和手机/平板有单独的应用（用 electron 开发），并且直接访问微服务。而电脑端则通过网页应用访问微服务。

## # 构建微服务

现在假设我们想在自己喜欢的电影院中去看某电影的首映。

首先，我们需要查看此影院中当前有哪些电影上映。下面这种图展示了微服务中是如何使用 REST 方式进行信息交流的。

![](https://img.zvz.im/imgs/2019/06/f142eea98b22201f.png)

我们的**电影服务（movies service）** API 规格定义如下：

```raml
#%RAML 1.0
title: cinema
version: v1
baseUri: /

types:
  Movie:
    properties:
      id: string
      title: string
      runtime: number
      format: string
      plot: string
      releaseYear: number
      releaseMonth: number
      releaseDay: number
    example:
      id: "123"
      title: "Assasins Creed"
      runtime: 115
      format: "IMAX"
      plot: "Lorem ipsum dolor sit amet"
      releaseYear : 2017
      releaseMonth: 1
      releaseDay: 6

  MoviePremieres:
    type: Movie []


resourceTypes:
  Collection:
    get:
      responses:
        200:
          body:
            application/json:
              type: <<item>>

/movies:
  /premieres:
    type:  { Collection: {item : MoviePremieres } }

  /{id}:
    type:  { Collection: {item : Movie } }
```

>  如果你不知道 **RAML** 是什么，这儿有一篇很好的[入门介绍](http://apiworkbench.com/docs/)。

此 API 项目的目录结构如下：

```yaml
- api/                  # our apis
- config/               # config for the app
- mock/                 # not necessary just for data examples
- repository/           # abstraction over our db
- server/               # server setup code
- package.json          # dependencies
- index.js              # main entrypoint of the app
```

让我们开始编码。首先看一下这个 `repository` 。这是我们进行数据库查询的地方。

```javascript
// repository.js
'use strict'
// factory function, that holds an open connection to the db,
// and exposes some functions for accessing the data.
const repository = (db) => {
  
  // since this is the movies-service, we already know
  // that we are going to query the `movies` collection
  // in all of our functions.
  const collection = db.collection('movies')

  const getMoviePremiers = () => {
    return new Promise((resolve, reject) => {
      const movies = []
      const currentDay = new Date()
      const query = {
        releaseYear: {
          $gt: currentDay.getFullYear() - 1,
          $lte: currentDay.getFullYear()
        },
        releaseMonth: {
          $gte: currentDay.getMonth() + 1,
          $lte: currentDay.getMonth() + 2
        },
        releaseDay: {
          $lte: currentDay.getDate()
        }
      }
      const cursor = collection.find(query)
      const addMovie = (movie) => {
        movies.push(movie)
      }
      const sendMovies = (err) => {
        if (err) {
          reject(new Error('An error occured fetching all movies, err:' + err))
        }
        resolve(movies)
      }
      cursor.forEach(addMovie, sendMovies)
    })
  }

  const getMovieById = (id) => {
    return new Promise((resolve, reject) => {
      const projection = { _id: 0, id: 1, title: 1, format: 1 }
      const sendMovie = (err, movie) => {
        if (err) {
          reject(new Error(`An error occured fetching a movie with id: ${id}, err: ${err}`))
        }
        resolve(movie)
      }
      // fetch a movie by id -- mongodb syntax
      collection.findOne({id: id}, projection, sendMovie)
    })
  }
  
  // this will close the database connection
  const disconnect = () => {
    db.close()
  }

  return Object.create({
    getAllMovies,
    getMoviePremiers,
    getMovieById,
    disconnect
  })
}

const connect = (connection) => {
  return new Promise((resolve, reject) => {
    if (!connection) {
      reject(new Error('connection db not supplied!'))
    }
    resolve(repository(connection))
  })
}
// this only exports a connected repo
module.exports = Object.assign({}, {connect})

// movie-service-repo.js
```

你可能注意到了，我们向唯一暴露的 **connect**(connection) 方法传入了一个 `connection` 对象，这儿你可以看到 javascript 最强大之处的**“闭包”**，这个仓库对象返回了一个闭包，其中所有的方法都能访问到 `db` 和 `collection` 对象， `db`  对象保持着数据库连接。这里我们对所连接数据库的类型进行了抽象，repository 对象并不知道使用的是哪一种数据库，本文中我们使用的是 **MongoDB**，它也不知道连接的数据库是单例还是集群，不过只要我们使用 mongodb 的语法，我们就能使用 repository 中的方法，我们还可以使用 [solid principles](https://medium.com/@cramirez92/s-o-l-i-d-the-first-5-priciples-of-object-oriented-design-with-javascript-790f6ac9b9fa#.ubs2g1ozf) 中的*依赖反转方式*，将 mongo 语法拆分到另一个文件中，而只调用数据库操作接口（比如使用 mongoose 模型）。

我们还有一个 `repository/repository.spec.js` 文件用于测试这个模块，以后我将会讲到测试的部分，你可以在 [github 仓库的 step-1 分支](https://github.com/Crizstian/cinema-microservice)中找到它。

接下来我们看一下 `server.js` 文件。

```javascript
'use strict'
const express = require('express')
const morgan = require('morgan')
const helmet = require('helmet')
const movieAPI = require('../api/movies')

const start = (options) => {
  return new Promise((resolve, reject) => {
    // we need to verify if we have a repository added and a server port
    if (!options.repo) {
      reject(new Error('The server must be started with a connected repository'))
    }
    if (!options.port) {
      reject(new Error('The server must be started with an available port'))
    }
    // let's init a express app, and add some middlewares
    const app = express()
    app.use(morgan('dev'))
    app.use(helmet())
    app.use((err, req, res, next) => {
      reject(new Error('Something went wrong!, err:' + err))
      res.status(500).send('Something went wrong!')
    })
    
    // we add our API's to the express app
    movieAPI(app, options)
    
    // finally we start the server, and return the newly created server 
    const server = app.listen(options.port, () => resolve(server))
  })
}

module.exports = Object.assign({}, {start})

// movie-service-server.js
```

这里我们实例化了一个新的 express 应用，验证我们是否提供了仓库对象以及和服务端口参数，然后使用了一些中间件，比如用于日志的 `morgan` ，用于安全的 `helmet` ，以及一个 `错误处理` 函数，最后对外提供了一个 start 方法，用于启动服务😎。

> Helmet 包括了多达 11 个模块全部是用于阻止针对用户的恶意攻击。

如果你想加强你的微服务的安全性，你可以看一下[这篇文章](https://nodesource.com/blog/nine-security-tips-to-keep-express-from-getting-pwned/?utm_source=nodeweekly&utm_medium=email)。

既然我们的 server 要使用到 movieAPI，那就让我们继续看一下 `movies.js` 。

```javascript
'use strict'
const status = require('http-status')

module.exports = (app, options) => {
  const {repo} = options
  
  // here we get all the movies 
  app.get('/movies', (req, res, next) => {
    repo.getAllMovies().then(movies => {
      res.status(status.OK).json(movies)
    }).catch(next)
  })
  
  // here we retrieve only the premieres
  app.get('/movies/premieres', (req, res, next) => {
    repo.getMoviePremiers().then(movies => {
      res.status(status.OK).json(movies)
    }).catch(next)
  })
  
  // here we get a movie by id
  app.get('/movies/:id', (req, res, next) => {
    repo.getMovieById(req.params.id).then(movie => {
      res.status(status.OK).json(movie)
    }).catch(next)
  })
}

// movies-service-movies.js
```

这里我们为 API 定义了路由，并根据路由调用仓库对象的不同方法。你可以注意到，这里是直接调用仓库对象的接口的，我们在实践着著名的「面向接口编程，而不是面向实现编程」箴言（coding for an interface not to an implementation），因为 express 路由并不知道数据库对象的存在，也不知道数据库查询的逻辑等等，它只是调用了仓库对象的方法用于处理所有的数据库相关业务。

我们所有的代码都有对应的单元测试，让我们看一下 `movies.js` 的测试代码。

> 你可以将测试代码当作你所构建应用的保护措施。它们不仅在你的本地机器上执行，还会在持续集成服务中执行，以保证失败的构建不会被推送到生成环境中。—— [Trace by RisingStack](https://medium.com/@RisingStack)

为了写单元测试，所有的依赖项都必须进行伪造，也就是说我们为要测试的模块提供伪造的依赖项。现在看下我们的 `标准测试文件` 长什么样子。

```javascript
/* eslint-env mocha */
const request = require('supertest')
const server = require('../server/server')

describe('Movies API', () => {
  let app = null
  let testMovies = [{
    'id': '3',
    'title': 'xXx: Reactivado',
    'format': 'IMAX',
    'releaseYear': 2017,
    'releaseMonth': 1,
    'releaseDay': 20
  }, {
    'id': '4',
    'title': 'Resident Evil: Capitulo Final',
    'format': 'IMAX',
    'releaseYear': 2017,
    'releaseMonth': 1,
    'releaseDay': 27
  }, {
    'id': '1',
    'title': 'Assasins Creed',
    'format': 'IMAX',
    'releaseYear': 2017,
    'releaseMonth': 1,
    'releaseDay': 6
  }]

  let testRepo = {
    getAllMovies () {
      return Promise.resolve(testMovies)
    },
    getMoviePremiers () {
      return Promise.resolve(testMovies.filter(movie => movie.releaseYear === 2017))
    },
    getMovieById (id) {
      return Promise.resolve(testMovies.find(movie => movie.id === id))
    }
  }

  beforeEach(() => {
    return server.start({
      port: 3000,
      repo: testRepo
    }).then(serv => {
      app = serv
    })
  })

  afterEach(() => {
    app.close()
    app = null
  })

  it('can return all movies', (done) => {
    request(app)
      .get('/movies')
      .expect((res) => {
        res.body.should.containEql({
          'id': '1',
          'title': 'Assasins Creed',
          'format': 'IMAX',
          'releaseYear': 2017,
          'releaseMonth': 1,
          'releaseDay': 6
        })
      })
      .expect(200, done)
  })

  it('can get movie premiers', (done) => {
    request(app)
    .get('/movies/premiers')
    .expect((res) => {
      res.body.should.containEql({
        'id': '1',
        'title': 'Assasins Creed',
        'format': 'IMAX',
        'releaseYear': 2017,
        'releaseMonth': 1,
        'releaseDay': 6
      })
    })
    .expect(200, done)
  })

  it('returns 200 for an known movie', (done) => {
    request(app)
      .get('/movies/1')
      .expect((res) => {
        res.body.should.containEql({
          'id': '1',
          'title': 'Assasins Creed',
          'format': 'IMAX',
          'releaseYear': 2017,
          'releaseMonth': 1,
          'releaseDay': 6
        })
      })
      .expect(200, done)
  })
})

//movie-service-movie.spec.js
```

```javascript
/* eslint-env mocha */
const server = require('./server')

describe('Server', () => {
  it('should require a port to start', () => {
    return server.start({
      repo: {}
    }).should.be.rejectedWith(/port/)
  })

  it('should require a repository to start', () => {
    return server.start({
      port: {}
    }).should.be.rejectedWith(/repository/)
  })
})

// movie-service-server.spec.js
```

如你所见，我们伪造了 `movies API` 的依赖项，我们验证了 server 对象需要服务端口和仓库对象。

你可在文本的 github 仓库中找到所有的测试文件。

让我们接下来看如何创建**仓库模块**中的 `数据库连接对象（db connection object）` ，在我们的规划中每个微服务都有自己的数据库，但是只会使用一个 **mongoDB 集群服务**，只是每个微服务都会使用自己的库。

下面是我们使用 **NodeJS** 连接 **MongoDB** 的数据库配置。

```javascript
const MongoClient = require('mongodb')

// here we create the url connection string that the driver needs
const getMongoURL = (options) => {
  const url = options.servers
    .reduce((prev, cur) => prev + `${cur.ip}:${cur.port},`, 'mongodb://')

  return `${url.substr(0, url.length - 1)}/${options.db}`
}

// mongoDB function to connect, open and authenticate
const connect = (options, mediator) => {
  mediator.once('boot.ready', () => {
    MongoClient.connect( getMongoURL(options), {
        db: options.dbParameters(),
        server: options.serverParameters(),
        replset: options.replsetParameters(options.repl)
      }, (err, db) => {
        if (err) {
          mediator.emit('db.error', err)
        }

        db.admin().authenticate(options.user, options.pass, (err, result) => {
          if (err) {
            mediator.emit('db.error', err)
          }
          mediator.emit('db.ready', db)
        })
      })
  })
}

module.exports = Object.assign({}, {connect})

// mongodb-relicaset.js
```

虽然可能还有其它更好的实现方式，不过我们基本上这样就可以与 mongoDB 集群建立连接了。

如你缩减，我们传入了一个 `options` 对象，它包含所有 mongo 连接需要的参数，我们还传入了一个 `event - mediator` 对象，当我们通过数据库连接验证时，它会喷出一个 `db` 对象。

> **注意\*** 这里我之所以使用了一个 event-emitter object 是因为使用 promise 方式时，当通过连接验证后，不知道什么原因它不能返回 db 对象，流程无法走下去——所以你可以挑战一下，尝试使用 promise 方式做一个实现，看看到底出了什么问题。

既然我们用到了 `options` 参数，那么让我们看下它来自哪里，所以下一个文件是 `config.js`

```javascript
// simple configuration file

// database parameters
const dbSettings = {
  db: process.env.DB || 'movies',
  user: process.env.DB_USER || 'cristian',
  pass: process.env.DB_PASS || 'cristianPassword2017',
  repl: process.env.DB_REPLS || 'rs1',
  servers: (process.env.DB_SERVERS) ? process.env.DB_SERVERS.split(' ') : [
    '192.168.99.100:27017',
    '192.168.99.101:27017',
    '192.168.99.102:27017'
  ],
  dbParameters: () => ({
    w: 'majority',
    wtimeout: 10000,
    j: true,
    readPreference: 'ReadPreference.SECONDARY_PREFERRED',
    native_parser: false
  }),
  serverParameters: () => ({
    autoReconnect: true,
    poolSize: 10,
    socketoptions: {
      keepAlive: 300,
      connectTimeoutMS: 30000,
      socketTimeoutMS: 30000
    }
  }),
  replsetParameters: (replset = 'rs1') => ({
    replicaSet: replset,
    ha: true,
    haInterval: 10000,
    poolSize: 10,
    socketoptions: {
      keepAlive: 300,
      connectTimeoutMS: 30000,
      socketTimeoutMS: 30000
    }
  })
}

// server parameters
const serverSettings = {
  port: process.env.PORT || 3000
}

module.exports = Object.assign({}, { dbSettings, serverSettings })

// movie-service-config.js
```

以上就是我们的配置文件，绝大多数时候，配置代码都是硬编码的，不过你可以看到有些属性使用了环境变量作为一个可选项。使用环境变量被认为是实践范式，因为这样在可以文件中隐藏数据库认证信息及服务器参数等。

最终，`movies-service`  API 的最后一步就是把所有这些模块集合到 `index.js` 中。

```javascript
'use strict'
// we load all the depencies we need
const {EventEmitter} = require('events')
const server = require('./server/server')
const repository = require('./repository/repository')
const config = require('./config/')
const mediator = new EventEmitter()

// verbose logging when we are starting the server
console.log('--- Movies Service ---')
console.log('Connecting to movies repository...')

// log unhandled execpetions
process.on('uncaughtException', (err) => {
  console.error('Unhandled Exception', err)
})
process.on('uncaughtRejection', (err, promise) => {
  console.error('Unhandled Rejection', err)
})

// event listener when the repository has been connected
mediator.on('db.ready', (db) => {
  let rep
  repository.connect(db)
    .then(repo => {
      console.log('Repository Connected. Starting Server')
      rep = repo
      return server.start({
        port: config.serverSettings.port,
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
mediator.on('db.error', (err) => {
  console.error(err)
})

// we load the connection to the repository
config.db.connect(config.dbSettings, mediator)
// init the repository connection, and the event listener will handle the rest
mediator.emit('boot.ready')

// movie-service-index.js
```

这里我们集成了所有的电影 API 服务，并且设置了一个简单的错误处理过程，之后我们加载配置文件，连接数据仓库，最终启动服务。

到此为止我们已经完成了所有 API 开发相关的工作，你可以在代码库的 [step-1 分支](https://github.com/Crizstian/cinema-microservice/tree/step-1) 找到源码。

如果你到 github 仓库去看一下，你会发现以下这些命令：

```bash
npm install          # setup node dependencies  
npm test             # unit test with mocha  
npm start            # starts the service  
npm run node-debug   # run the server in debug mode 
npm run chrome-debug # debug the node with chrome
npm run lint         # lint the code with standard
```

我们可以使用 `npm start` 命令启动我们的第一个微服务，并在本地运行，这可不是文章标题所声称的运行在 docker 中🤔。

不过首先我们要做的，是拥有 [“用 docker 创建一个 mongoDB 集群”](https://medium.com/@cramirez92/how-to-deploy-a-mongodb-replica-set-using-docker-6d0b9ac00e49#.xln0x0owy) 这篇文章中的 **Docker** 环境，如果你还没有，那么你需要一些额外的步骤为微服务创建一个数据库。这儿还有些命令是用来测试我们的电影服务的。

首先我们创建一个 **Dockerfile** 将我们的 **NodeJS 微服务** Docker 化。

```dockerfile
# Node v7 as the base image to support ES6
FROM node:7.2.0
# Create a new user to our new container and avoid the root user
RUN useradd --user-group --create-home --shell /bin/false nupp && \
    apt-get clean
ENV HOME=/home/nupp
COPY package.json npm-shrinkwrap.json $HOME/app/
COPY src/ $HOME/app/src
RUN chown -R nupp:nupp $HOME/* /usr/local/
WORKDIR $HOME/app
RUN npm cache clean && \
    npm install --silent --progress=false --production
RUN chown -R nupp:nupp $HOME/*
USER nupp

EXPOSE 3000

CMD ["npm", "start"]
```

我们用 NodeJS 镜像作为我们的基础镜像，然后创建一个用户避免使用 root 账户，之后我们把源码拷贝到镜像中再安装依赖库，对外暴露一个端口号，最后实例化我们的电影服务。

接下来使用下面的命令构建我们的 docker 镜像：

```shell
$ docker build -t moives-service .
```

先分析一下这条构建指令。

1. `docker build` 告诉 docker 引擎我们要创建一个新的镜像。
2. `-t moives-service` 将这个镜像打上 `movies-service` 标签。以后我们就可以用标签引用这个镜像。
3. `.` 表示在当前目录寻找 `Dockerfile` 。

在看到一些命令行输出之后，包含我们的 NodeJS 应用的镜像就准备好了，现在我们就可以用下面的命令运行这个镜像了：

```shell
$ docker run --name movie-service -p 3000:3000 -e DB_SERVERS="192.168.99.100:27017 192.168.99.101:27017 192.168.99.100:27017" -d movies-service
```

命令中我们传入了一个环境变量，用来提供 mongoDB 集群的服务器地址，这儿只是做一个演示，实际可以有更好操作的方式，比如从 env 文件中读取配置。

现在我们的 docker 容器已经运行起来了，我们可以执行 `docker-machine ip machine-name` 获取微服务的 ip 地址，这样我们微服务就已经准备就绪，可以对其进行集成测试了。

以下是我们的测试代码，它会进行一个接口调用检测。

```javascript
/* eslint-env mocha */
const supertest = require('supertest')

describe('movies-service', () => {

  const api = supertest('http://192.168.99.100:3000')

  it('returns a 200 for a collection of movies', (done) => {

    api.get('/movies/premiers')
      .expect(200, done)
  })
})

// movies-service-integration.test.js
```

## 总结时间

最后回顾一下我们都做了什么事情。

![](https://img.zvz.im/imgs/2019/06/f142eea98b22201f.png)

我们已经完成了以上信息交互中的第一个部分，构建了**电影服务**使得我们可以在电影院查询电影首映时间，使用 NodeJS 实现了**电影服务**的 API，我们先用一个 **RAML** 文件进行 **API 设计**，然后开始构建我们的 **API**，并且进行了相应的**单元测试**，最后把所有的代码整合成我们的**微服务**，使我们的电影服务可以启动起来。

之后我们把我们的**微服务**打包进 **Docker** 容器，使我们能对其进行一些**集成测试。**

虽然我们可能见过许多 **NodeJS** 开发的项目，不过仍然还有许多东西值得我们去学习和实践。本文只是一个小小的展示，我希望它展示出了一些有趣而有用的东西，使你在工作中能够运用 **Docker 和 NodeJS** 的相关技术。

## Github 上的完整代码

你可以在 Github 上找到本文的完整代码 [Crizstian/cinema-microservice](https://github.com/Crizstian/cinema-microservice)。


翻译自：[Build a NodeJS cinema microservice and deploying it with docker — part 1](https://medium.com/@cramirez92/build-a-nodejs-cinema-microservice-and-deploying-it-with-docker-part-1-7e28e25bfa8b)



[系列文章 - Part 2](https://zvz.im/2017/05/24/nodejs-cinema-microservice-part2/)