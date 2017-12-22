# @nrk/doublecache-as-promised

> Fast and resilient cache for NodeJs targeting high-volume sites

- [Features](#features)
- [Installing](#installing)
- [Basic usage](#examples)
- [Distributed capabilites](#distributed-capabilites)
- [Local development](#local-development)
- [Building and committing](#building-and-committing)

# Motivation
Sometimes Node.js needs to do some heavy lifting, performing CPU or network intensive tasks and yet respond quickly on incoming requests. For repetitive tasks like Server side rendering of markup or parsing big JSON responses caching can give the application a great performance boost. In addition - serving stale content when a backend resource is down may save your day! The intention of `doublecache-as-promised` is to give you a fairly simple, yet powerful application cache, with fine-grained control over caching behaviour.

`doublecache-as-promised` is inspired by how [Varnish](https://varnish-cache.org/) works. It is not intended to replace Varnish (but works great in combination). In general Varnish works great as an edge/burst/failover cache, in addition to reverse proxying and loadbalancing. There exists several other cache solutions on NPM, but they're often too basic or too attached to a combination of perquisites that does not fit all needs of an application cache.

## Features
- __In-memory cache__ is used as primary storage since it will always be faster than parsing and fetching data over network. An [LRU-cache](https://www.npmjs.com/package/lru-cache) is enabled to constrain the amount of memory used.
- __Persistent cache__ is used as secondary storage to avoid high back-pressure on backend resources when caches are cleared after server restarts. This is achieved storing cache-misses in Redis depending on a [ioredis](https://www.npmjs.com/package/ioredis)-factory
- __Caches are filled using (worker) promises__ since cached objects often are depending on async operations. (RxJs)[https://www.npmjs.com/package/rxjs] is used to queue concurrent requests for the same key; thus ensuring that only __one__ worker is performed when cached content is missing/stale.
- __Caching of custom class instances, functions and native objects__ such as Date, RegExp and redux stores are supported through in-memory caching. Non-serializable (using JSON.stringify) objects are filtered out in persistent caches though.
- __On demand expiry__ is supported using Redis pub/sub, so that new content may be available published before cache-TTL is reached.
- __Grace mode__ is used if a worker promise fails (eg. caused by failing backends), ie.  stale cache is returned instead.
- __Avoidance of spamming backend resources__ using a configurable retry-wait parameter, serving either a stale object or rejection.

## Performance testing

Parsing a json-file at around 47kb (file contents are cached at startup). Using a Macbook pro, mid 2015, 16gb ram, i7 CPU.

<p align="left">
  <img src="./test/linear-perftest-nocache.jpeg?raw=true" width="50%"/>
</p>

The image shows graph from running the test script `perf:nocache-cache-file -- --type=linear`. At around 1300 iterations the event loop starts lagging, and at around 1500 iterations the process stops responding. It displays that even extremely optimized JSON.parse could be a bottleneck when fetching remote data for rendring. (`React.render` would be even slower)

<p align="left">
  <img src="./test/linear-perftest-cache.jpeg?raw=true" width="50%"/>
</p>

The second image is a graph from running test script `perf:cache -- --type=linear`. At around 3.1 million iterations the event loop starts lagging, and at around 3.4 million iterations the process runs out of memory and crashes. The graph has no relation to how fast JSON.parse is, but what speed is achievable by skipping it altogether (ie. `Promise`-processing)

## Installing

```
npm install @nrk/doublecache-as-promised --save
```

# Examples
*Note! These examples are written using ES2015 syntax. The lib is exported as CJS modules though, using Babel*

## Basic usage
```js
import inMemoryCache from '@nrk/doublecache-as-promised'
const cache = inMemoryCache({ /* options */})

// imiplicit set cache on miss, or use cached value
cache.get('key', {/* options */}, Promise.resolve({hello: 'world'}))
  .then((data) => {
    console.log(data)
    // {
    //   value: {
    //     hello: 'world'
    //   },
    //   created: 123456789,
    //   cache: 'miss',
    //   TTL: 86400000
    // }
  })
```

## Basic usage with options
```js
import inMemoryCache from '@nrk/doublecache-as-promised';

const cache = inMemoryCache({
  initial: {                    // initial state
    foo: 'bar'
  },                            
  maxLength: 1000,              // LRU max object count
  maxAge: 24 * 60 * 60 * 1000   // LRU max age in ms
})
// set/overwrite cache key
cache.set('key', {hello: 'world'})
// imiplicit set cache on miss, or use cached value
cache.get('anotherkey', {
  ttl: 60 * 1000,               // TTL for cached object, in ms
  workerTimeout: 5 * 1000,      // worker timeout, in ms
  deltaWait: 5 * 1000           // wait time, if worker fails
}, Promise.resolve({hello: 'world'}))
  .then((data) => {
    console.log(data)
    // {
    //   value: {
    //     hello: 'world'
    //   },
    //   created: 123456789,
    //   cache: 'miss',
    //   TTL: 86400000
    // }
  })
```

## Distributed capabilites
Distributed expire and persisting of cache misses to Redis are provided as
plugins using __function composition__, ie. wrapping the in-memory cache with a factory that intercepts function calls. It should therefore be easy to write your own plugins using pub/sub from rabbitMQ, zeroMQ, persisting to a NAS, hit/miss-ratio to external measurments systems and more.

#### Distributed expire
```js
import inMemoryCache, {distCache} from '@nrk/doublecache-as-promised'
import Redis from 'ioredis'

// a factory function that returns a redisClient
const redisFactory = () => new Redis(/* options */)
const cache = distCache(
  inMemoryCache({initial: {fooKey: 'bar'}}),
  redisFactory,
  'namespace'
)
// publish to redis (using wildcard)
cache.expire(['foo*'])
setTimeout(() => {
  cache.get('fooKey').then(console.log)
  // expired in server # 1 + 2
  // {value: {fooKey: 'bar'}, created: 123456789, cache: 'stale', TTL: 86400000}
}, 1000)
```

#### Persisting cache misses
```js
import inMemoryCache, {persistentCache} from '@nrk/doublecache-as-promised'
import Redis from 'ioredis'

const redisFactory = () => new Redis(/* options */)
const cache = persistentCache(
  inMemoryCache({/* options */}),
  redisFactory,
  {
    keySpace: 'myCache',   // key prefix used when storing in redis
    grace: 60 * 60         // auto expire unused keys in Redis after TTL + grace seconds
  }
)

cache.get('key', {/* options */}, Promise.resolve('hello'))
// will store a key in redis, using key: myCache-<key>
// {value: 'hello', created: 123456789, cache: 'hit', TTL: 60000}
```

#### Persisting cache misses __and__ distributed expire
```js
import inMemoryCache, {distCache, persistenCache} from '@nrk/doublecache-as-promised'
import Redis from 'ioredis'

const redisFactory = () => new Redis(/* options */)
const imc = inMemoryCache({/* options */})
const dc = distCache(imc, redisFactory, 'namespace')
const cache = persistenCache(
  dc,
  redisFactory,
  {
    keySpace: 'myCache',   // key prefix used when storing in redis
    grace: 60 * 60         // auto expire unused keys in Redis after TTL + grace seconds
  }
)

cache.expire(['foo*'])  // distributed expire of all keys starting with foo
cache.get('key', {
  ttl: 60000,                       // in ms
  workerTimeout: 5000,
  deltaWait: 5000
}, Promise.resolve('hello').then(console.log)
// will store a key in redis, using key: myCache-<key>
// {value: 'hello', created: 123456789, cache: 'miss', TTL: 60000}
```

---

## Local development
First clone the repo and install its dependencies:

```bash
git clone git@github.com:nrkno/doublecache-as-promised.git
cd doublecache-as-promised
npm install && npm run build && npm run test
```

## Building and committing
After having applied changes, remember to build and run tests before pushing the changes upstream.

```bash
git checkout -b feature/my-changes
# update the source code
npm run build
git commit -am "Add my changes"
git push origin feature/my-changes
# then make a PR to the master branch,
# and assign another developer to review your code
```

> NOTE! Please also make sure to keep commits small and clean (that the commit message actually refers to the updated files).  
> Stylistically, make sure the commit message is **Capitalized** and **starts with a verb in the present tense** (for example `Add minification support`).
