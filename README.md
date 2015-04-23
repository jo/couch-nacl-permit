# couch-nacl-permit
Handle database permits, which encrypts a database key with a session key. For
use with [couch-box](https://github.com/jo/couch-box).

[![Build Status](https://travis-ci.org/jo/couch-nacl-permit.svg?branch=master)](https://travis-ci.org/jo/couch-nacl-permit)

couch-nacl-permit uses [TweetNaCl.js](https://github.com/dchest/tweetnacl-js), a port of
[TweetNaCl](http://tweetnacl.cr.yp.to/) / [NaCl](http://nacl.cr.yp.to/) to
JavaScript for modern browsers and Node.js by Dmitry Chestnykh
([@dchest](https://github.com/dchest)).

The use of this widely ported cryptography library makes it possible to
implement this encryption schema in other, possibly more secure platforms, for
example with Python and CouchDB.

**:warning: Only to play around! Not yet ready for production use.**

## Installation
couch-nacl-permit is [hosted on npm](https://www.npmjs.com/package/couch-nacl-permit).

### Node
Install via `npm install couch-nacl-permit` 

## Usage
```js
var sessionKeyPair = require('tweetnacl').box.keyPair()
// a keyPair consists of a `publicKey` and a `secretKey` (here base64 encoded):
// {
//   secretKey: 'smsDNnqeT40IfAwDw0+6x5WzDRYFv0492O/JW/s8tT0=',
//   publicKey: 'sAUGULAT5q2g6gzNMuBX1tkY/FsnoiLA/tv2XmmU2Dg='
// }

// create a permit
var permit = require('couch-nacl-permit')(sessionKeyPair)

// build new permit
permit.build()
// parse permit doc
permit.parse()
// return json representation
permit.toJSON()
// return base64 encoded public database key
permit.receiver()
```

## Details
### Database Key
A database can have one or more database keys. The private database key is
stored encrypted in permit documents.

```json
{
  "secretKey": "LCvM/keRcO00AguI5aBX+tY0UfIb7n5w294JJZ2i1XU=",
  "publicKey": "2XiwPX1U6pKPitmhyeubV9g4YYxtIxNfMNE6B5keEmg="
}
```
The keys are both 32 bytes long. They are based on Curve25519 and created with
the [NaCl crypto_box](http://nacl.cr.yp.to/box.html) `crypto_box_keypair`
function.

The keypair above was created similar to session key generation by the following
statements:
```js
var nacl = require('tweetnacl')

var databaseKeyPair = nacl.box.keyPair()

{
  secretKey: nacl.util.encodeBase64(databaseKeyPair.secretKey),
  publicKey: nacl.util.encodeBase64(databaseKeyPair.publicKey)
}
```


### Permit: `permit/<permit-id>`
A permit contains the database secret key, encrypted with the public session
key. Decoupling of session and database keys allows the account holder to change
its keys and also enables the use of different public key algorithms for session
keys.

The schema of the permit varies between the chosen algorithms. By now only
`curve25519-xsalsa20-poly1305` is supported. This may change in the future.

The permit id is the base64 encoded public session key, prefixed with
`permit/`.

In order to create the database permit we

1. Create an ephemeral key pair
2. Create a nonce
3. Encrypt the database secret key with nonce, public session key and ephemeral
secret key

```json
{
  "_id": "permit/sAUGULAT5q2g6gzNMuBX1tkY/FsnoiLA/tv2XmmU2Dg=",
  "type": "curve25519-xsalsa20-poly1305",
  "createdAt": "2015-03-18T01:29:46.764Z",
  "ephemeral": "RxYOHMWUri/8+aVUDodcocOhTBakV5BckIU9kdeFwSo=",
  "nonce": "Le1AvMAvjkVy3mn1knnmovY36lYk028K",
  "encryptedKey": "7aX5QPyU7OABMFD6YZJF8akTqiNnP4LN9CetpsW/37LgRdl5DPuPtoUiahCMGbCq"
}
```

```js
var nacl = require('tweetnacl')

var sessionKeyPair = nacl.box.keyPair.fromSecretKey(
  nacl.util.decodeBase64('smsDNnqeT40IfAwDw0+6x5WzDRYFv0492O/JW/s8tT0=')
)
var databaseKeyPair = nacl.box.keyPair.fromSecretKey(
  nacl.util.decodeBase64('LCvM/keRcO00AguI5aBX+tY0UfIb7n5w294JJZ2i1XU=')
)
var permitEphemeralKeyPair = nacl.box.keyPair()
var permitNonce = nacl.randomBytes(nacl.box.nonceLength)

{
  _id: 'permit/' + nacl.util.encodeBase64(sessionKeyPair.publicKey),
  type: 'curve25519-xsalsa20-poly1305',
  createdAt: new Date(),
  ephemeral: nacl.util.encodeBase64(permitEphemeralKeyPair.publicKey),
  nonce: nacl.util.encodeBase64(permitNonce),
  encryptedKey: nacl.util.encodeBase64(nacl.box(
    databaseKeyPair.secretKey,
    permitNonce,
    sessionKeyPair.publicKey,
    permitEphemeralKeyPair.secretKey
  ))
}
```

## Testing
```sh
npm test
```
