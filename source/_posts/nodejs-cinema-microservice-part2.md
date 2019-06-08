title: 用 NodeJS 打造影院微服务并部署到 docker 上 — Part 2
date: 2017-7-17 14:18:23
tags:
- Nodejs
---

> 本文是「使用 NodeJS 构建影院微服务」系列的第二篇文章。

### ## 第一章快速回顾

* 我们讲了**什么是微服务**
* 我们对比了**微服务**的**利**与**弊**
* 我们设计了一个**影院微服务的架构**
* 我们使用 **RAML** 定义了**电影服务 API** 的技术规格
* 我们使用 **NodeJS 和 ExpressJS** 开发了 **电影服务 API**
* 我们针对 API 做了**单元测试**
* 我们将 API 组成一个服务，并使我们的**电影服务**运行在 **Docker** 容器中
* 我们对运行在 **Docker** 中的**电影服务**做了**集成测试**

如果你还没有阅读过第一篇文章，你可以[去这里看看](https://log.zvz.im/2017/05/24/nodejs-cinema-microservice-part1/)👀。

在本篇中我们将继续构建我们的**影院微服务**，这次我们将进行**影院目录服务**的开发，以完成下图中的功能。

![](https://img.zvz.im/imgs/2019/06/f142eea98b22201f.png)
<!-- more -->
我们将使用到以下技术：

* NodeJS version 7.2.0
* MongoDB 3.4.1
* Docker for Mac 1.13

要跟上本文的进度有以下要求：

* 已经完成[上一篇文章](https://log.zvz.im/2017/05/24/nodejs-cinema-microservice-part1/)中的例子代码

如果你还没有完成这些代码，我已经将代码传到了 github 上，你可以直接使用[代码库](https://github.com/Crizstian/cinema-microservice/tree/step-1)分支 **step-1**。

## # 微服务安全以及 ¿ HTTP/2 ?

在第一章中，我们实现了一个简单的微服务，它使用的是 HTTP/1.1 协议。对于已有 15 年历史的 HTTP 协议来说，**HTTP/2 是第一次大升级**，它被高度优化且效率更高。**HTTP/2 是最新的网络标准**，它起源于谷歌的 SPDY 协议。现在已有许多热门站点在使用它，并且几乎所有主流浏览器都支持该协议。

**要实现 HTTP/2，只需满足以下一些规则。**

* 它仅支持 HTTPS 协议（我们需要一个有效的 SSL 安全证书）
* 它是向后兼容的。当浏览器或设备不支持 **HTTP/2** 时，它能回退到 HTTP/1.1
* 它会带来极大的性能提升
* 它不需要客户端做任何改动，只需要服务端做一个基础实现
* 一些有趣的新特性能提高站点的加载速度，在 HTTP 1.1 中这种方式是无法想象的

### 一个启用了 HTTP/2 的微服务网络架构

*这意味着我们要在客户端与服务端之间启用一个单独的连接，然后在“网络中”利用类似 Y 轴分片（ Y-axis sharding 会在本系列文章的动态扩容篇讲到更多）的方式享受 HTTP/2 带来的性能提升，同时还能享受微服务架构在运营和开发上带来的好处。*

为什么我们要使用最新的 **HTTP/2 协议**？因为作为一名好的开发者，我们不仅要保障应用，基础设施和通信的安全，尽最大努力来防止恶意攻击；而且作为一名好的开发者，我们还应该采用对我们有利的最佳实践方式，比如使用 HTTP/2 。

微服务安全的一些好的实践方式如下：

> 安全，是微服务应用生产环境部署中的一个重要组成部分。根据 451 Research 的一项名为[2016软件定义设施展望](https://451research.com/blog/64-451-research-presents-2016-software-defined-infrastructure-sdi-outlook)的调查，在未来的 12 月内，近 45% 的企业已经或计划实现基于容器的应用部署。由于 DevOps 实践在企业中取得了稳固地位，容器应用变得更加流行，安全顾问们需要知道如何保障这些应用的安全。—— [@Ranga Rajagopalan](http://www.darkreading.com/endpoint/rethinking-application-security-with-microservices-architectures-/a/d-id/1325155)

* 发现并监控服务间通信
* 分段并隔离应用与服务
* 对传输中的数据以及存储的数据进行加密

我们将对微服务通信进行加密以实现安全通信，并提升公网流量的安全性，这是我们使用 **HTTP/2** 的原因之一，为了更好的性能以及安全提升。

## # 微服务的 HTTP/2 实现

首先，让我们更新下上一章的**电影服务**，以支持 **HTTP/2** 协议，在此之前我们在 **config** 目录下新建一个 **ssl** 目录。

```shell
movies-service/config:/ $ mkdir ssl
movies-service/config:/ $ cd ssl
```

进入 **ssl** 目录后，我们创建一个自签名的 SSL 证书，用于实现 **HTTP/2** 协议。

```bash
# Let's generate the server pass key

ssl/: $ openssl genrsa -des3 -passout pass:x -out server.pass.key 2048 
# now generate the server key from the pass key

ssl/: $ openssl rsa -passin pass:x -in server.pass.key -out server.key
# we remove the pass key

ssl/: $ rm server.pass.key
# now let's create the .csr file
ssl/: $ openssl req -new -key server.key -out server.csr 
...
Country Name (2 letter code) [AU]:MX
State or Province Name (full name) [Some-State]:Michoacan 
... 
A challenge password []: 
...
# now let's create the .crt file
ssl/: $ openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
```

然后我们使用下面的命令安装 **SPDY** 包：

```shell
cinema-catalog-service/: $ npm i -S spdy --silent
```

首先我们在 `ssl/` 目录下创建一个 `index.js` 文件，代码如下，我们在代码中加载 key 和 cert 文件，这里是我们为数不多地使用 `fs.readFileSync()` 的地方之一：

```javascript
const fs = require('fs')
module.exports = {
  key: fs.readFileSync(`${__dirname}/server.key`),
  cert: fs.readFileSync(`${__dirname}/server.crt`)
}
```

然后我们还要修改一些文件，第一个文件是 `config.js` ：

```javascript
const dbSettings = { ... }
// The first modification is adding the ssl certificates to the 
// serverSettings
const serverSettings = {
  port: process.env.PORT || 3000,
  ssl: require('./ssl')
}
module.exports = Object.assign({}, { dbSettings, serverSettings })
```

接下来修改 `server.js` ，如下：

```javascript
...
const spdy = require('spdy')
const api = require('../api/movies')
const start = (options) => {
...
 const app = express()
 app.use(morgan('dev'))
 app.use(helmet())
 app.use((err, req, res, next) => {
   reject(new Error('Something went wrong!, err:' + err))
   res.status(500).send('Something went wrong!')
 })
 api(app, options)
 // here is where we made the modifications, we create a spdy
 // server, then we pass the ssl certs, and the express app
 const server = spdy.createServer(options.ssl, app)
      .listen(options.port, () => resolve(server))
  })
}
module.exports = Object.assign({}, {start})
```

最后修改 `index.js` 这个主文件：

```javascript
'use strict'
const {EventEmitter} = require('events')
const server = require('./server/server')
const repository = require('./repository/repository')
const config = require('./config/')
const mediator = new EventEmitter()
...
mediator.on('db.ready', (db) => {
  let rep
  repository.connect(db)
    .then(repo => {
      console.log('Connected. Starting Server')
      rep = repo
      return server.start({
        port: config.serverSettings.port,
        // here we pass the ssl options to the server.js file
        ssl: config.serverSettings.ssl,
        repo
      })
    })
    .then(app => { ... })
})
...
```

现在我们使用下面的指令重建 docker 镜像：

```shell
$ docker build -t movies-service .
```

再使用以下参数运行我们的 docker 镜像 `moives-service` ：

```shell
$ docker run --name movies-service -p 443:3000 -d movies-service
```

最终我们用 chrome 浏览器测试一下，可以确认我们的 **HTTP/2** 协议已经完全生效了。

![](https://img.zvz.im/imgs/2019/06/6858ab638d364e2d.png)

而且我们还可以使用 **wireshark** 之类的抓包工具确认 **ssl** 确实生效了。

![](https://img.zvz.im/imgs/2019/06/0d98f20c7725bd61.png)

### # 在微服务上实现 JWT

另外一种加密并保障我们的微服务通信安全的方式是使用 `json web token` 协议，我们将在本系列后面的文章中讲到🚉。

## # 构建微服务

到此为止我们已经知道了如何实现 **HTTP/2** 协议，接下来让我们继续构建**影院目录服务** 。我们会使用与**电影服务**一样的项目结构，让我们少讲话🗣多编码👨🏻‍💻👩🏻‍💻这就干起来。

在开始设计 API 之前，这次我们要先为数据库设计 **Mongo Schemas**，我们将使用到以下数据库：

* Locations（国家，州县和城市）
* Cinemas（影院，放映计划，电影）

### # 模型数据设计

本文重点关注于构建微服务，故不会在“模型数据设计”上花太多的时间，只会强调某些部分。

```
# posible collections for the cinemas db.
# For locations 
- countries
- states
- cities
# For cinemas
- cinemas
- cinemaRooms
- schedules
```

对于我们的 **Locations** 库来说，一个国家可以有多个州而一个州则对应一个国家，所以这是**一对多**的关系，对于州和城市来说也是一样的关系，让我们看一下关系图：

![](https://img.zvz.im/imgs/2019/06/5c43c08b82f70617.png)

![](https://img.zvz.im/imgs/2019/06/7de1fafcdbfff1a2.png)

这种关系还适用于很多情况。一个城市有多个电影院，一个电影院属于一个城市；一个放映室有许多放映计划，一个放映计划属于一个放映室，我们看一下它们之间的关系。

![](https://img.zvz.im/imgs/2019/06/3f1a1b0ba2e94f39.png)

如果电影院数组或者放映计划数据增长比较有限，我们可以采用上图中的引用关系。假设一个放映室每天最多只有 5 个放映计划，我们也可以将放映计划文档直接嵌入到影院文档中去。

> 内嵌数据模型允许应用将相关性数据片断存放在同一条数据库记录中。这样做可使得应用的常用操作需要更少的查询和更新。—— MongoDB Docs

![](https://img.zvz.im/imgs/2019/06/213cd97d135f56e1.png)

这就是我们数据库结构设计的最终结果。

### # 将数据导入数据库

我已经为刚才设计的数据库结构准备了一些样例数据，在 github 仓库的 `cinema-catalog-service/src/mock` 目录下有 4 个 JOSN 文件，你可以将它们导入到影院数据库中，不过首先我们找到数据库主服务器，之后执行以下命令：

```shell
# first we need to copy the files one by one or we can zip it and pass the zip file
$ docker cp countries.json mongoNodeContainer:/tmp
$ docker cp state.json mongoNodeContainer:/tmp
$ docker cp city.json mongoNodeContainer:/tmp
$ docker cp cinemas.json mongoNodeContainer:/tmp
```

执行完上面的命令后，我们就可以把数据导入到数据库中了：

```shell
$ docker exec mongoNode{number} bash -c 'mongoimport --db cinemas --collection countries --file /tmp/countries.json --jsonArray -u $MONGO_USER_ADMIN -p $MONGO_PASS_ADMIN --authenticationDatabase "admin"'

$ docker exec mongoNode{number} bash -c 'mongoimport --db cinemas --collection states --file /tmp/states.json --jsonArray -u $MONGO_USER_ADMIN -p $MONGO_PASS_ADMIN --authenticationDatabase "admin"'

$ docker exec mongoNode{number} bash -c 'mongoimport --db cinemas --collection cities --file /tmp/cities.json --jsonArray -u $MONGO_USER_ADMIN -p $MONGO_PASS_ADMIN --authenticationDatabase "admin"'

$ docker exec mongoNode{number} bash -c 'mongoimport --db cinemas --collection cinemas --file /tmp/cinemas.json --jsonArray -u $MONGO_USER_ADMIN -p $MONGO_PASS_ADMIN --authenticationDatabase "admin"'
```

现在我们已经设计好了数据库结构，而且数据也已导入可以查询了，那么我们现在开始为**影院目录服务**设计 API，有一种定义路由的方法是用一句话把它们表达出来，如下：

* 我们需要一个城市能够显示所有可用的电影院。
* 我们需要这些电影院展示电影上映信息。
* 我们需要电影的首映信息并展示放映计划。
* 我们需要放映计划并查看是否有座位可供订票。

我们假设 **Cinépolis** IT 部门的其他开发小组在做其他的增删改查操作，而交给我们的任务只是查询读取数据，我们还假设一些 **Cinépolis** 影院的操作人员已经为影院排好了放映计划，这样一来我们的任务只是取获取这些计划而已。

**影院目录服务**只专注于处理影院和放映计划两方面的数据，没有其它功能，之前我们提到的 **locations** 数据集合，是另一个微服务负责处理的，不过显示影院和放映计划时还是要依赖位置信息的。

现在我们已经确认了需求边界，可以创建我们的 RAML 文件了，如下：

```yaml
#%RAML 1.0
title: Cinema Catalog Service
version: v1
baseUri: /
uses:
  object: types.raml
  stack: ../movies-service/api.raml

types:
  Cinemas: object.Cinema []
  Movies: stack.MoviePremieres
  Schedules: object.Schedule []

traits:
  FilterByLocation:
    queryParameters:
      city:
        type: string

resourceTypes:
  GET:
    get:
      responses:
        200:
          body:
            application/json:
              type: <<item>>


/cinemas:
  type:  { GET: {item : Cinemas } }
  get:
    is: [FilterByLocation]
  description: we already have the location defined to display the cinemas

  /cinemas/{cinema_id}:
    type:  { GET: {item : Movies } }
    description: we have selected the cinema to display the movie premieres

    /cinemas/{cinema_id}/{movie_id}:
      type:  { GET: {item : Schedules } }
      description: we have selceted a movie to display the schedules
```

我们已经实现了上面四句话需求中的三句，第 4 条需求是在电影院订票，不过它是**订票服务**负责的，所以请继续关注「用 NodeJS 打造影院微服务」系列的后续文章。

现在我们可以继续开发**影院目录服务**了，它的结构与配置和**电影服务**基本一样，所以先看一下 API 的 **repository.js** 文件。

```javascript
// more code above

const getCinemasByCity = (cityId) => {
    return new Promise((resolve, reject) => {
      const cinemas = []
      const query = {city_id: cityId}
      const projection = {_id: 1, name: 1}
      // example of making a find query to mongoDB, 
      // passign a query and projection objects.
      const cursor = db.collection('cinemas').find(query, projection)
      const addCinema = (cinema) => {
        cinemas.push(cinema)
      }
      const sendCinemas = (err) => {
        if (err) {
          reject(new Error('An error occured fetching cinemas, err: ' + err))
        }
        resolve(cinemas)
      }
      cursor.forEach(addCinema, sendCinemas)
    })
  }

  const getCinemaById = (cinemaId) => {
    return new Promise((resolve, reject) => {
      const query = {_id: new ObjectID(cinemaId)}
      const projection = {_id: 1, name: 1, cinemaPremieres: 1}
      const response = (err, cinema) => {
        if (err) {
          reject(new Error('An error occuered retrieving a cinema, err: ' + err))
        }
        resolve(cinema)
      }
      // example of using findOne method from mongodb, 
      // we do this because we only need one record.
      db.collection('cinemas').findOne(query, projection, response)
    })
  }

  const getCinemaScheduleByMovie = (options) => {
    return new Promise((resolve, reject) => {
      const match = { $match: {
        'city_id': options.cityId,
        'cinemaRooms.schedules.movie_id': options.movieId
      }}
      const project = { $project: {
        'name': 1,
        'cinemaRooms.schedules.time': 1,
        'cinemaRooms.name': 1,
        'cinemaRooms.format': 1
      }}
      const unwind = [{ $unwind: '$cinemaRooms' }, { $unwind: '$cinemaRooms.schedules' }]
      const group = [{ $group: {
        _id: {
          name: '$name',
          room: '$cinemaRooms.name'
        },
        schedules: { $addToSet: '$cinemaRooms.schedules.time' }
      }}, { $group: {
        _id: '$_id.name',
        schedules: {
          $addToSet: {
            room: '$_id.room',
            schedules: '$schedules'
          }
        }
      }}]
      const sendSchedules = (err, result) => {
        if (err) {
          reject('An error has occured fetching schedules by movie, err: ' + err)
        }
        resolve(result)
      }
      // example of using a aggregation method from mongoDB
      // we difine our pipline above, we are using also ES6 spread operator
      db.collection('cinemas').aggregate([match, project, ...unwind, ...group], sendSchedules)
    })
  }
```

想要查看完整版 **repository.js** 文件，你可以去 github 仓库的 *step-2* 分支里去找。

此文件中定义了三个函数：

* **getCinemasByCity**: 通过此函数我们可以获取到城市中所有可用的电影院，我们传入 city_id 参数以获取电影院信息，可用来调用下一个函数。
* **getCinemaById**: 此函数可以通过 cinema_id 查询到电影院的名称，id，以及正在上映的电影，返回结果可以帮助我们最终获取到电影的放映计划。
* **getCinemaScheduleByMovie**: 这个函数可以为我们提供城中所有影院关于某电影的放映计划。

为了展示当前影院的放映计划，我们可能还需要一个函数，或者我们可以修改一下 **getCinemaById** 方法，如果你乐意做个小练习，这是个不错的挑战，由于我已经提供了所有需要的信息，所以这应该不会太难。

下面我们要看 API 文件 **cinema-catalog.js**。

```javascript
'use strict'
const status = require('http-status')

module.exports = (app, options) => {
  const {repo} = options

  app.get('/cinemas', (req, res, next) => {
    repo.getCinemasByCity(req.query.cityId)
      .then(cinemas => {
        res.status(status.OK).json(cinemas)
      })
      .catch(next)
  })

  app.get('/cinemas/:cinemaId', (req, res, next) => {
    repo.getCinemaById(req.params.cinemaId)
      .then(cinema => {
        res.status(status.OK).json(cinema)
      })
      .catch(next)
  })

  app.get('/cinemas/:cityId/:movieId', (req, res, next) => {
    const params = {
      cityId: req.params.cityId,
      movieId: req.params.movieId
    }
    repo.getCinemaScheduleByMovie(params)
      .then(schedules => {
        res.status(status.OK).json(schedules)
      })
      .catch(next)
  })
}
```

如你所见，这里我们实现了服务的入口，和我们在 **RAML** 文件中定义的一样，根据路由调用 `repository.js` 中的不同方法。

在第一个路由中，我们使用 `req.query.cityId` 的值作为城市的 `city_id` 查询数据库获取到影院信息，另一个路由中，我们通过 `req.params` 得到 `cinemaId` 和 `movieId` 的值，来查询放映计划 `schedules` 😎。

最后，我们看一下 `cinema-catalog.spec.js` 这个测试文件：

```javascript

/* eslint-env mocha */
const request = require('supertest')
const server = require('../server/server')
process.env.NODE = 'test'

describe('Movies API', () => {
  let app = null
  const testCinemasCity = [{
    '_id': '588ac3a02d029a6d15d0b5c4',
    'name': 'Plaza Morelia'
  }, {
    '_id': '588ac3a02d029a6d15d0b5c5',
    'name': 'Las Americas'
  }]

  const testCinemaId = {
    '_id': '588ac3a02d029a6d15d0b5c4',
    'name': 'Plaza Morelia',
    'cinemaPremieres': [
      {
        'id': '1',
        'title': 'Assasins Creed',
        'runtime': 115,
        'plot': 'Lorem ipsum dolor sit amet',
        'poster': 'link to poster...'
      },
      {
        'id': '2',
        'title': 'Aliados',
        'runtime': 124,
        'plot': 'Lorem ipsum dolor sit amet',
        'poster': 'link to poster...'
      },
      {
        'id': '3',
        'title': 'xXx: Reactivado',
        'runtime': 107,
        'plot': 'Lorem ipsum dolor sit amet',
        'poster': 'link to poster...'
      }
    ]
  }

  const testSchedulesMovie = [{
    '_id': 'Plaza Morelia',
    'schedules': [{
      'room': 2.0,
      'schedules': [ '10:15' ]
    }, {
      'room': 1.0,
      'schedules': [ '6:55', '4:35', '10:15' ]
    }, {
      'room': 3.0,
      'schedules': [ '10:15' ]
    }]
  }, {
    '_id': 'Las Americas',
    'schedules': [ {
      'room': 2.0,
      'schedules': [ '3:25', '10:15' ]
    }, {
      'room': 1.0,
      'schedules': [ '12:15', '10:15' ]
    }]
  }]

  let testRepo = {
    getCinemasByCity (location) {
      console.log(location)
      return Promise.resolve(testCinemasCity)
    },
    getCinemaById (cinemaId) {
      console.log(cinemaId)
      return Promise.resolve(testCinemaId)
    },
    getCinemaScheduleByMovie (cinemaId, movieId) {
      console.log(cinemaId, movieId)
      return Promise.resolve(testSchedulesMovie)
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

  it('can return cinemas by location', (done) => {
    const location = {
      city: '588ababf2d029a6d15d0b5bf'
    }
    request(app)
      .get(`/cinemas?cityId=${location.city}`)
      .expect((res) => {
        res.body.should.containEql(testCinemasCity[0])
        res.body.should.containEql(testCinemasCity[1])
      })
      .expect(200, done)
  })

  it('can get movie premiers by cinema', (done) => {
    request(app)
    .get('/cinemas/588ac3a02d029a6d15d0b5c4')
    .expect((res) => {
      res.body.should.containEql(testCinemaId)
    })
    .expect(200, done)
  })

  it('can get schedules by cinema and movie', (done) => {
    request(app)
      .get('/cinemas/588ababf2d029a6d15d0b5bf/1')
      .expect((res) => {
        res.body.should.containEql(testSchedulesMovie[0])
        res.body.should.containEql(testSchedulesMovie[1])
      })
      .expect(200, done)
  })
})
```

最终我们可以构建一个 `cinema-catalog-service` 的 docker 镜像，并使其在容器中运行。我们将使用和 **movies service** 一样的 `dockerfile` ，为了使操作过程更加自动化一些，我们创建了一个 bash 脚本 `start-service.sh` 如下：

```bash
#!/usr/bin/env bash
eval `docker-machine env manager1`
docker build -t catalog-service .
docker run --name catalog-service -p 3000:3000 --env-file env -d catalog-service
```

由于我们的服务会不断增加，所以需要注意可供服务使用的端口，这次我们使用 3000 端口，另外我会使用一个**环境文件**使我们的 **NodeJS** 服务开始使用 **process.env** 配置，我们的环境文件看起来如下：

```
DB=cinemas
DB_USER=cristian
DB_PASS=cristianPassword2017
DB_REPLS=rs1
DB_SERVERS='192.168.99.100:27017 192.168.99.101:27017 192.168.99.102:27017'
PORT=3000
```

在 devOps 的世界中，这被认为是最好的做法。

最后只要执行我们的 bash 小脚本：

```shell
# execute our script
$ bash < start-service.sh
# check running docker contianers
$ docker ps
```

应该可以看到类似下图的情况：

![](https://img.zvz.im/imgs/2019/06/9306e91bf244845c.png)

我们还可以在 chrome 浏览器中测一下我们的服务，验证 **HTTP/2** 协议能正常工作，而且服务也是正常运行的。

![](https://img.zvz.im/imgs/2019/06/80065700ac28226e.png)

我们还可以找点乐子，使用 **JMeter** 进行压力测试，压测文件也在 github 仓库中的 **integration-test/** 目录下。

![](https://img.zvz.im/imgs/2019/06/15b430ee5d0b7082.png)

![](https://img.zvz.im/imgs/2019/06/89fcb6d89df7cf8a.gif)

## # 总结时间

我们都做了什么。。。

![](https://img.zvz.im/imgs/2019/06/f142eea98b22201f.png)

我们已经完成了图中的这些微服务，你可能会说在**影院目录服务**中我们还没有调用**电影服务**。的确如此，我们至此只实现了服务的 **GET** 请求处理。而在**影院目录服务**中调用**电影服务**时使用 **POST** 请求，是为了给影院输入上映的电影，以便制作放映计划，由于我们本次的任务只是 **CRUD** 操作中的 **R** 读操作，所以我们还没有看到这个交互。不过在本系列的后续文章中，我们会在微服务中实现更多的 CRUD 操作，请保持耐心和好奇心 :D 。

那么本章中我们做了什么 ¿ 🤔 ?，我们学习了 **HTTP/2** 协议并且在**微服务**中实现了它。我们还看到了如何**设计一个 MongoDB 模型**，虽然并不深入但是我突出了这个部分，并详细描绘了**影院目录服务**设计中发生了什么，然后我们用 **RAML** 设计了 API 接口，接着实现 API 服务，之后做了相应的**单元测试**，最后我们把这些都组装起来完成了微服务。

其次我们使用了与之前的**微服务**一样的 **dockerfile**，并构建了一个脚本来自动化此过程，我们还介绍了如何使用环境文件，并且在 **Docker** 容器中使用 `environment variables`，在完成这些之后，作为**集成测试**的补充，我还向你们展示了如何使用 JMeter 进行**压力测试**。

虽然我们可能见过许多 **NodeJS** 开发的项目，不过仍然还有许多东西值得我们去学习和实践。本文只是一个小小的展示，我希望它展示出了一些有趣而有用的东西，使你在工作中能够运用 **Docker 和 NodeJS** 的相关技术。



翻译自：[Build a NodeJS cinema microservice and deploying it with docker (part 2)](https://medium.com/@cramirez92/build-a-nodejs-cinema-microservice-and-deploying-it-with-docker-part-2-e05cc7b126e0)



[系列文章 - Part 1](https://log.zvz.im/2017/05/24/nodejs-cinema-microservice-part1/)