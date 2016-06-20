# p2p calendar

---


---
# services that nobody can own

---
# 

computer programs that talk to each other

everything is a client

---
# hyperlog

---
# kappa architecture

---
# 

some things I've built with this architecture:

* p2p offline map database

---
# a calendar

* parse-messy-time
* parse-messy-schedule
* 

---
# sharing a calendar

Here are some options:

* share your whole calendar - bad for privacy
* 

---
# capability system

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
