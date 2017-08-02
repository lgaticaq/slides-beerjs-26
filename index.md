title: Beerjs #26
author:
  name: Leonardo Gatica
  twitter: lgaticaq
  url: https://about.me/lgatica
  email: lgatica@protonmail.com
output: index.html
theme: juanbrujo/cleaver-beerjs
style: style.css
controls: true

--

<h1>Demo de API con Koa</h1>

--

# Dependencias

```shell
npm i koa koa-bodyparser koa-logger koa-router mongoose
```

--

# app.js

```javascript
const Koa = require('koa')
const BodyParser = require('koa-bodyparser')
const logger = require('koa-logger')
const mongoose = require('mongoose')
const router = require('./router')

mongoose.Promise = global.Promise
const db = async function () {
  try {
    mongoose.connect('mongodb://localhost/beerjs')
  } catch (err) {
    process.exit(1)
  }
}
db()
const app = new Koa()
app.use(logger())
app.use(BodyParser())
app.use(router.routes()).use(router.allowedMethods())
module.exports = app
```

--

# router.js

```javascript
const Router = require('koa-router')
const controllers = require('./controllers')

const router = new Router()

router.post('/', controllers.create)
router.get('/', controllers.list)
router.get('/:event', controllers.get)
router.put('/:event', controllers.update)
router.delete('/:event', controllers.remove)

module.exports = router
```

--

# controller.js (create)

```javascript
const Beerjs = require('./model')

exports.create = async ctx => {
  try {
    const doc = Beerjs()
    doc.event = ctx.request.body.event
    doc.datetime = ctx.request.body.datetime
    doc.place = ctx.request.body.place
    doc.address = ctx.request.body.address
    doc.theme = ctx.request.body.theme
    doc.requirement = ctx.request.body.requirement
    await doc.save()
    ctx.body = doc
  } catch (err) {
    ctx.throw(500, err.message)
  }
}
```

--

# controller.js (list)

```javascript
const Beerjs = require('./model')

exports.list = async ctx => {
  try {
    const docs = await Beerjs.find()
    ctx.body = docs.map(doc => doc.asJson)
  } catch (err) {
    ctx.throw(500, err.message)
  }
}
```

--

# controller.js (get)

```javascript
const Beerjs = require('./model')

exports.get = async ctx => {
  try {
    const doc = await Beerjs.findOne({ event: ctx.params.event })
    if (doc) {
      ctx.body = doc.asJson
    } else {
      ctx.throw(404)
    }
  } catch (err) {
    ctx.throw(500, err.message)
  }
}
```

--

# controller.js (update)

```javascript
const Beerjs = require('./model')

exports.update = async ctx => {
  try {
    const doc = await Beerjs.findOne({ event: ctx.params.event })
    if (doc) {
      if (ctx.request.body.datetime) {
        doc.datetime = ctx.request.body.datetime
      }
      ...
      await doc.save()
      ctx.body = doc.asJson
    } else {
      ctx.throw(404)
    }
  } catch (err) {
    ctx.throw(500, err.message)
  }
}
```

--

# controller.js (delete)

```javascript
const Beerjs = require('./model')

exports.remove = async ctx => {
  try {
    const doc = await Beerjs.findOne({ event: ctx.params.event })
    if (doc) {
      await Beerjs.findByIdAndRemove(doc._id)
      ctx.body = doc.asJson
    } else {
      ctx.throw(404)
    }
  } catch (err) {
    ctx.throw(500, err.message)
  }
}
```

--

# model.js

```javascript
const mongoose = require('mongoose')

const Schema = new mongoose.Schema({
  event: {
    type: Number,
    unique: true
  },
  datetime: Date,
  place: String,
  address: String,
  theme: String,
  requirement: {
    type: String,
    default: 'Traer rica y helada cerveza'
  },
  registry: String
})
```

--

# model.js (pre)

```javascript
Schema.pre('save', function (next) {
  const id = parseInt(Math.random() * (500000 - 470000) + 470000, 10)
  this.registry = `https://guestlist.co/events/${id}`
  next()
})
```

--

# model.js (virtual)

```javascript
Schema.virtual('asJson').get(function () {
  const doc = this.toJSON()
  delete doc._id
  delete doc.__v
  return doc
})

module.exports = mongoose.model('Beerjs', Schema)
```

--

# Dockerfile (production)

```Dockerfile
FROM lgatica/node-krb5:8-onbuild
```

--

# Dockerfile (development)

```Dockerfile
FROM lgatica/node-krb5:8

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

COPY package.json package-lock.json /usr/src/app/
RUN npm i
COPY . /usr/src/app

CMD [ "npm", "run", "dev" ]
```

--

# Dockerfile (lgatica/node-krb5:8)

```Dockerfile
FROM node:8.1.2-alpine

RUN apk add --no-cache --virtual build-dependencies make gcc g++ python && \
  apk add --no-cache krb5-dev

CMD [ "node" ]
```

--

# Dockerfile (lgatica/node-krb5:8-onbuild)

```Dockerfile
FROM lgatica/node-krb5:8.1.2

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

ONBUILD ARG NODE_ENV
ONBUILD ENV NODE_ENV $NODE_ENV
ONBUILD COPY package.json package-lock.* yarn.* /usr/src/app/
ONBUILD RUN if [ -e yarn.lock ]; \
  then yarn && yarn cache clean; \
  else npm i && npm cache clean --force; fi && \
  apk del build-dependencies && \
  rm -rf ~/.node-gyp /tmp/*
ONBUILD COPY . /usr/src/app

CMD [ "npm", "start" ]
```

--

# docker-compose.yml (production)

```yaml
version: '2'
services:
  app:
    build: .
    depends_on:
      - mongo
    environment:
      - MONGODB_URI=mongo://mongo/beerjs
    ports:
      - "3000:3000"
  mongo:
    image: mvertes/alpine-mongo:3.4.4-0
    volumes:
      - mongo:/data/db
volumes:
  mongo:
```

--

# docker-compose.yml (development)

```yaml
version: '2'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    depends_on:
      - mongo
    environment:
      - MONGODB_URI=mongo://mongo/beerjs
    ports:
      - "3000:3000"
      - "9229:9229"
    volumes:
      - .:/usr/src/app
  mongo:
    image: mvertes/alpine-mongo:3.4.4-0
    volumes:
      - mongo:/data/db
volumes:
  mongo:
```
