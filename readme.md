[Streaming](https://nodejs.org/api/stream.html) [BSON](http://bsonspec.org/) parser with [rudimentary](#issues) MongoDB [archive](https://www.mongodb.com/blog/post/archiving-and-compression-in-mongodb-tools) support.

- [`bson`](https://www.npmjs.com/package/bson) is used for parsing, it is faster than [`bson-ext`](https://www.npmjs.com/package/bson-ext) according to some tests.

# Install

```sh
$ npm i stream-bson
# or
$ yarn add stream-bson
```

# Usage example

Counting records and different values ([`trace.js`](./trace.js))

```js
const { createReadStream, writeFileSync } = require('fs')
const StreamBSON = require('./index.js')

const Trace = q => {
  const ks = Array.isArray(q) ? q : q.split('/')
  const r = {}
  return {
    get value() {
      return r
    },
    push(x) {
      const k = ks.map(k => x[k]).join('/')
      r[k] = r[k] || 0
      ++r[k]
    }
  }
}

let trace = null
let i = 0

createReadStream(process.argv[2])
  .pipe(new StreamBSON({ archive: true }))
  .on('data', doc => {
    ++i
    trace = trace || Trace(process.env.q || Object.keys(doc).filter(k => !k.startsWith('_')))
    trace.push(doc)
  })
  .on('finish', () => {
    console.log(`${i} records processed`)
    console.log(`writing trace in ${process.argv[2]}.json`)
    const r = {}
    Object.keys(trace.value).sort().forEach(k => {
      if (trace.value[k] >= (process.env.n ? parseInt(process.env.n) : 1)) {
        r[k] = trace.value[k]
      }
    })
    writeFileSync(`${process.argv[2]}.json`, JSON.stringify(r, null, 2))
  })
  .on('error', err => {
    console.log(err)
  })
```

# Issues

MongoDB [archives](https://github.com/mongodb/mongo-tools/blob/master/common/archive/archive.go) are not parsed properly. Assuming there is a BSON file in the archive, setting `archive` in `StreamBSON` options will skip the archive header (until `\x07_id`) and ignore the footer (via parsing error).
