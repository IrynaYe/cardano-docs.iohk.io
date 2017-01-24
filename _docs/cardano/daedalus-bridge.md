# `daedalus-bridge`

JavaScript based API of [`cardano-sl`](https://github.com/input-output-hk/cardano-sl/).


## Build `daedalus-bridge` locally

To run `daedalus-bridge` locally you have to start `wallet-api` of [`cardano-sl`](https://github.com/input-output-hk/cardano-sl/) as follow. Make sure that your are on root folder of [`cardano-sl`](https://github.com/input-output-hk/cardano-sl/).

```bash
# build app
stack build --flag cardano-sl:with-wallet --flag cardano-sl:with-web
# run tmux in another window
tmux
# generate PureScript types
stack exec -- cardano-wallet-hs2purs
# launch nodes
export WALLET_TEST=1; ./scripts/launch.sh
```

If you might run into some issues remove the following content first and build `wallet-api` again as described above.

```
rm -rf ./run/*
rm -rf wallet-db
rm node-*.*.key
```


With a running `wallet-api` you can run `daedalus-bridge` locally as follow. Please note that [yarn](https://yarnpkg.com/) and [Bower](https://bower.io/) are required to build `daedalus-bridge.

```bash
cd daedalus
yarn install
yarn start
open http://localhost:3080/
```

To try examples described in chapter ["Usage of API"](#Usage) just open console of your browser to put and run any example code there.


## Build `daedalus-bridge` for production

```bash
yarn build:prod
```

`daedalus-bridge` is build to `dist/Daedalus.js`.

Note: `daedalus-bridge` is not optimized / compressed. This is will be a job for Daedalus.


## Usage of API

API of `daedalus-bridge` does provide following (mostly Promise based) functions. All examples are written in ES5.

_getWallets_

```javascript
Daedalus.ClientApi.getWallets()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log('ERROR', reason);
  })
```


_getWallet_

```javascript
// XXX - any wallet id
Daedalus.ClientApi.getWallet('XXX')()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log('ERROR', reason);
  })
```


_newWallet_

```javascript
Daedalus.ClientApi.newWallet(
    'CWTPersonal'
  , 'ADA'
  , 'wallet name'
  , 'mnemonic words')()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log('ERROR', reason);
  })
```


_generateMnemonic_

```javascript
Daedalus.ClientApi.generateMnemonic()
```


_deleteWallet_

```javascript
// XXX - any wallet id
Daedalus.ClientApi.deleteWallet('XXX')()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log(reason);
  })
```


_send_

```javascript
// IdFrom - wallet id to send amount from
// IdTo - wallet id to send amount to
Daedalus.ClientApi.send('IdFrom', 'IdTo', 80)()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log(reason);
  })
```


_history_

```javascript
// XXX - wallet id
Daedalus.ClientApi.getHistory('XXX')()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log(reason);
  })
```


_isValidAddress_

```javascript
// XXX - wallet id
Daedalus.ClientApi.isValidAddress('XXX', 'ADA')()
  .then(function(value) {
    console.log('SUCCESS', value);
  }, function(reason) {
    console.log(reason);
  })
```

_notify_

```javascript
Daedalus.ClientApi.notify(
    // message callback
   function(msg) { return function() { console.log('msg ', msg) } },
   // error callback
   function(error) { return function() { console.log('error ', error) } }
 )();
```


## Run tests

First, make sure that you have all dependencies installed. Run (only once):
```bash
yarn install
```

Run tests:
```bash
yarn test
```
