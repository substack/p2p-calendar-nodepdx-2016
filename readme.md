# p2p calendar

James Halliday

https://substack.neocities.com

---
# google calendar

at most a few kilobytes of text data

and yet:

```
$ du -sh Google\ Calendar\ -\ Month\ of\ Jun\ 2016*
2.9M  Google Calendar - Month of Jun 2016_files
556K  Google Calendar - Month of Jun 2016.html
```

---
# privacy

now an advertising company (google)
has your calendar

---
# services that nobody can own

instead of using somebody else's computers (the cloud),

we can use our own computers

---
# p2p

make the web great again

---
# everything is a client

Computer programs talk to each other
directly without mediation.

(counterantidisintermediationism)

---
# tips for building a distributed system

Avoid applying centralized thinking to distributed systems:

* global concensus
* "conflicts" - why can't we all get along?

---
# tips for building a distributed system

Instead of solving a problem,
try not having that problem in the first place.

---
# anarchitecture

kappa architecture

* an append-only log serves as a source of truth
* materialized views create derived information about the log

---
# some things I've built with this architecture:

* p2p offline map database (osm-p2p)

---
# some libraries for a p2p calendar

* hyperlog
* hyperlog-index
* hyperkv
* hyperlog-calendar-index
* parse-messy-time
* parse-messy-schedule
* hyperlog-calendar-index
* calendar-db
* calendar-month-string
* text-layers

---
# hyperlog

append-only log

where each document can point at the hash of prior documents

like git! (a merkle DAG)

---
# hyperlog

appending a document:

``` js
var hyperlog = require('hyperlog')
var level = require('level')
var db = level('./data.db')
var log = hyperlog(db, { valueEncoding: 'json' })

var value = JSON.parse(process.argv[2])
var ancestorKeys = process.argv.slice(3)

log.add(ancestorKeys, value, function (err, node) {
 if (err) console.error(err)
 else console.log(node)
})
```

---
# hyperlog

listing all documents:

``` js
var hyperlog = require('hyperlog')
var level = require('level')
var db = level('./data.db')
var log = hyperlog(db, { valueEncoding: 'json' })

log.createReadStream().on('data', console.log)
```

---
# hyperlog-index

tail the log to generate materialized views

``` js
var hindex = require('hyperlog-index')

hindex({
 log: log,
 db: db,
 map: function (row, next) {
  // `row` is a document from the log
  // create some derived values here
  // then call next()
 })
})
```

---
# hyperkv

a key/value store derived from a hyperlog

multi-value register conflict strategy

---
# hyperkv

``` js
var hyperkv = require('hyperkv')
var kv = hyperkv({
 db: level('./index.db'),
 log: log
})

kv.put('hello', { msg: 'world' }, function (err) {
 if (err) return console.error(err)
 kv.get('hello', function (err, values) {
  if (err) return console.error(err)
  console.log(values) // note, "values" not "value"
 })
})
```

---
# multi-value register

for any given key, there may be multiple values

plurality!

Now you don't need to solve consensus problems.

Make the UI deal with it.

---
# where I'm going with this...

* write calendar events into hyperlog
* with hyperkv so that events are updateable with an event ID
* which uses hyperlog-index

---
# some time parsing

* parse-messy-time
* parse-messy-schedule

DEMO

---
# where I'm going with this...

* write calendar events into hyperlog
* with hyperkv so that events are updateable with an event ID
* which uses hyperlog-index

* encapsulate parse-messy-{time,schedule} into calendar-db
* make hyperlog-calendar-index to write to calendar-db
* which uses hyperlog-index to read from hyperkv

---
# where I'm going with this...

* and calendar-month-string renders a text calendar
* then text-layers positions blocks of text

---
# norcal

does all of those things

in 63 sloc! (because modularity)

```
$ grep \\S index.js | wc -l
63
```

---
# norcal

DEMO

also let's add replication to the source code!

---
# sharing a calendar

Here are some options:

* share your whole calendar publically
* encrypt your calendar and share the key
* encrypt individual properties and share those keys

---
# capability system: unix file system

```
(u)ser
 |  (g)roup
 |   |  (o)ther
 |   |   |
[ ] [ ] [ ]
rwx rwx rwx
```

* r - able to read a file
* w - able to write to a file
* x - able to execute a file or list a directory

---
# capability system

Q: How can we make a distributed capability system where
other networked computers help us host our data?

A: cryptography!

---
# hmac capability system

We can use HMACs to implement a distributed capability system.

``` js
var crypto = require('crypto')

var rootkey = crypto.randomBytes(32)
var propkey1 = hmac(rootkey, 'propname1')
var propkey2 = hmac(rootkey, 'propname2')
var propkey3 = hmac(rootkey, 'propname3')

function hmac (root, name) {
  return crypto.createHmac('sha256', root).write(name).digest()
}
```

---
# hmac encryption

encrypt each property with the propkey:

``` js
var crypto = require('crypto')
var sodium = require('chloride')

var nonce1 = crypto.randomBytes(24)
var plainText = Buffer('hello pdx')
var cipherText1 = sodium.crypto_secretbox(plainText, nonce1, propkey1)
```

---
# hmac storage

Save the nonce value and the cipher text:

```
var fs = require('fs')
fs.writeFile('encrypted.txt', JSON.stringify({
  propname1: { nonce: nonce1, cipherText: cipherText1 },
  propname2: { nonce: nonce2, cipherText: cipherText2 },
  propname3: { nonce: nonce3, cipherText: cipherText3 }
})
```

It's safe to distribute these values publically.

---
# hmac storage

For extra privacy, you can also encrypt the list of property names.

Here we'll use a special `_keys` property.

``` js
var listKey = hmac(rootkey, '_keys')

var listNonce = crypto.randomBytes(24)
var plainText = Buffer(JSON.stringify(['propname1','propname2','propname3']))
var listCipherText = sodium.crypto_secretbox(plainText, listNonce, listkey)
```

---
# hmac storage

Now our storage payload becomes:

``` js
var fs = require('fs')
fs.writeFile('encrypted.txt', JSON.stringify([
  { nonce: listNonce, cipherText: listCipherText },
  { nonce: nonce1, cipherText: cipherText1 },
  { nonce: nonce2, cipherText: cipherText2 },
  { nonce: nonce3, cipherText: cipherText3 }
])
```

---
# hmac capabilities overview

* r - individual prop keys
* w - access to the root key plus a signed message authorizing the key
* x - `_keys` property

---
# data payload

What if we want to share several months or a whole year's worth of information?

That could be a lot of data.

---
# recursive hmac capabilities

We can derive a sub-rootkey from a rootkey:

``` js
var rootkey = crypto.randomBytes(32)
var subrootkey = hmac(rootkey, 'subobjname')
```

---
# recursive hmac capabilities

We can derive a sub-rootkey from a rootkey:

``` js
var rootkey = crypto.randomBytes(32)
var subrootkey = hmac(rootkey, 'subobjname')
```

and that sub-rootkey can have its own properties:


``` js
var subpropkey1 = hmac(subrootkey, 'propname1')
```

...and so on, recursively

---
# your day, your week, your month, and even your year

``` js
var year = crypto.randomBytes(32)
var month = hmac(year, 'june')
var week = hmac(month, '19-25')
var day = hmac(week, '21')
```

---
# your day, your week, your month, and even your year

``` js
var year = crypto.randomBytes(32)
var month = hmac(year, 'june')
var week = hmac(month, '19-25')
var day = hmac(week, '21')
```

---
# saving an event

and we can store event data at the highest resolution (day):

``` js
var values = {
  time: '2016-06-21 9:30',
  location: '722 E burnside portland oregon usa',
  event: 'nodepdx',
  title: 'a p2p calendar talk'
}
var enc = {}, keys = {}

Object.keys(values).forEach(function (key) {
  var nonce = crypto.randomBytes(24)
  var propkey = hmac(day, key)
  var plainText = values[key]

  enc[key] = {
    nonce: nonce,
    cipherText: sodium.crypto_secretbox(plainText, nonce, propkey)
  }
  keys[key] = propkey
})
```

---
# and now

Now, if we want to share our whole week or month or year,
we only need to send one key!

If we want to be exclusive about properties,
we need to send those property keys individually,
but that is less information.

---
# partial decryption: single property

``` js
var keys = JSON.parse(sodium.crypto_secretbox_open(
  listCipherText, listNonce, listKey))
var propIndex = keys.indexOf('propname1') + 1
var rec = values[propIndex]
var prop1 = sodium.crypto_secretbox_open(rec.cipherText, rec.nonce, propkey1)
```

---
# partial decryption: whole event

``` js
var listKey = hmac(evkey, '_keys')
var keys = JSON.parse(sodium.crypto_secretbox_open(
  listCipherText, listNonce, listKey))

keys.forEach(function (key, i) {
  var rec = values[i+1]
  var propkey = hmac(evkey, key)
  var prop = sodium.crypto_secretbox_open(
    rec.cipherText, rec.nonce, propkey)
  console.log(propkey, '=>', prop)
})
```

---
# partial decryption: whole day of events

``` js
var dayListKey = hmac(day, '_keys')
var dayKeys = JSON.parse(sodium.crypto_secretbox_open(
  dayListCipherText, dayListNonce, dayListKey))

dayKeys.forEach(function (evProp) {
  var evkey = hmac(day, evProp)
  var listKey = hmac(evkey, '_keys')
  var keys = JSON.parse(sodium.crypto_secretbox_open(
    listCipherText, listNonce, listKey))

  keys.forEach(function (key, i) {
    var rec = values[i+1]
    var propkey = hmac(evkey, key)
    var prop = sodium.crypto_secretbox_open(
      rec.cipherText, rec.nonce, propkey)
    console.log(propkey, '=>', prop)
  })
})
```

---
# nested derived hmac encryption module

That was a lot of code.

Module coming soon.

---
# and now

let's imagine using that capability system in our p2p calendar

I bet it would be nice.

---

EOF
