---
layout: default
title: Building Cardano SL and Daedalus from Source
permalink: /for-contributors/building-from-source/
group: for-contributors
---
# Building Cardano SL and Daedalus from Source

Cardano SL consists of a collection of binaries that constitutes
backend, a PureScript API for the Electron-based wallet and the
Electron-based wallet called “Daedalus”. You can read more about design
and architecture of Cardano SL in the [Implementation
Guide](/for-contributors/implementation.md).

## Cardano SL and Daedalus Bridge

Source code for both Cardano SL and Daedalus Bridge can be obtained
(here)[https://github.com/input-output-hk/pos-haskell-prototype]

We strongly suggest using [Nix package
manager](https://nixos.org/nix/download.html) to get the right
dependencies for building Cardano SL. It will fetch the correct
`openssl` version, but won't override the system-installed version. Same
goes for dependencies such as `rocksdb`, with which many developers
report having problems. The rest of documentation assumes that the reader
has Nix package manager installed on their machine.

To build the project, first clone the source base and navigate to the root
directory of it:

```
git clone https://github.com/input-output-hk/pos-haskell-prototype.git
cd pos-haskell-prototype
```

Then enter `nix-shell` and, if it's the first project in Haskell
language on the machine, run `stack setup`.

```
Tue Jan 10 sweater@chill ~/github/pos-haskell-prototype (master)
λ nix-shell
[nix-shell:~/github/pos-haskell-prototype]$ stack setup
* snip *
```

After the relevant Haskell compiler version is obtained, let's enable
Nix for all `stack` builds. To do so, make sure that
`~/.stack/config.yaml` has the following option:

```
nix:
  enable: true
```

Now in order to build Cardano SL with wallet capabilities, run the
following command:

```
[nix-shell:~/github/pos-haskell-prototype]
$ stack build --flag cardano-sl:with-wallet --flag cardano-sl:with-web
```

Here is the [asciinema
cast](http://asciinema.org/a/47vbdch8srbhf3j5kta7j9bov) of the project building.

It is suggested having at least 8GB of RAM and some swap space for the build
process. As the project is fairly large and GHC parallelizes builds very
effectively, memory and CPU consumption during the build process is
demanding.

After the project is built, the built binaries can be launched
using the `stack exec` command. Let's discuss important binaries briefly
before proceeding to next steps.

### cardano-node

Binary `cardano-node` is the most important binary of the system. It
launches nodes. In order to attach to a network, Hardened Kadmelia DHT
peer information has to be supplied. Peer discovery will follow if
initial DHT peer is available. The syntax for communicating initial
DHT peer is the following: `--peer HOST:PORT/HOST_ID`, for example
`discover.memorici.de:21989/MHdtsP-oPf7UWly7QuXnLK5RDB8=`.

[//]: # (TODO: Actually put a small dev-only net with a discoverable)
[//]: # (peers which will send a recruitment propsal message to people)
[//]: # (who bothered to build the system from scratch in the early days)
[//]: # (of testnet release)

Before providing an example of running the node, the trickiest command
line arguments are noted.

When a testnet is bootstrapped, stake is distributed across several
addresses in the genesis block. The distributions that are supported
are flat distribution and Bitcoin distribution. Spending and VSS keys
are generated for genesis block. If a node has access to a genesis key
mapping, the index of the secret key in this mapping can be provided
using `--vss-genesis N` and `--spending-genesis N`, where `N` is index
in this mapping.

Example of a local invoaction connecting to HostID
`MHdtsP-oPf7UWly007QuXnLK5RD=`:

```
stack exec -- cardano-node \
  --port 3002 \
  --db-path run/node-db2 \
  --vss-genesis 2 --spending-genesis 2 \
  --peer 127.0.0.1:3000/MHdtsP-oPf7UWly007QuXnLK5RD= \
  --json-log=logs/2017-01-10_035413/node2.json \
  --logs-prefix logs/2017-01-10_035413 \
  --log-config logs/2017-01-10_035413/conf/node2.log.yaml \
  --flat-distr "(3, 100000)"
```

### cardano-smart-generator

A tool which tests if transactions are accordingly sent by a node in
a network; a stress-test can be invocated by `cardano-smart-generator`.
This tool is designed to provide reasonable and reliable measurements of
transactions per second (or TPS).

It works in few rounds, each split in few phases.

At start, the initial transaction is submited. This transaction uses unspent
output from the genesis address, index of which is supplied as `--index`
argument and creates `(k + P) * slotDuration * maxTPS` outputs, which
would be used for further transactions. `k` (block depth to treat
transaction as stable) and `P` (approximate amount of slots needed for a
transaction to be successfully published in a block) are parameters
here. During our benchmarks we were using `k=6` and `P=2` with hundred
nodes.

Each round of `cardano-smart-generator` tests that the system is capable of
handling a concrete TPS value. It goes from value provided by `--tps`
CLI argument and adjusts it every step by `--tps-step` value. Both can
be fractional (double precision floats). The process will continue for
at most `--round-number` (`N`) rounds.

This way, `maxTps = initTps + tpsStep * N`.

All further transactions are `(in, 1, A)`, where

 + `A` is the owner of `in`;
 + 1 is the amount of coins transferred;
 + `in` is the output of previous transaction;  

Each subsequent transaction is being sent only if the parent is included
into block of depth `k` (i.e stable). This way there's no possibility
of producing more transactions than a node can include into blocks.

Each round is split to `(R + 2)` phases. On each phase transactions are
sent with current TPS rate. On the first phase no measurements are taken.gq
On the last phase no new transactions get emmitted, only confirmations
for sent ones being collected.

Each phase takes `(k + P) * slotDuration` seconds, where `P` is
approximate amount of slots needed for a successful transaction to be
published in block.

Transactions are sent in a few threads. Each thread uses its own
transaction pool derived from its own index in genesis block.

Here is an example of invocation of `cardano-smart-generator` sending
transactions from node number zero:

```
stack exec -- cardano-smart-generator \
  --json-log=txgen.json \
  --index 0 \
  --round-period-rate 60 \
  --round-number 10 \
  --tps 50 \
  --propagate-threshold 4 \
  --tps-sleep 20 \
  --init-money 100000 \
  --peer 127.0.0.1:3000/MHdtsP-oPf7UWly007QuXnLK5RD=
```

The purpose of the most important binaries and basic
operations with said binaries is clarified, let's proceed with building
the wallet. On the Haskell side of things, just two matters have to be
addressed —

 1. Generate types for `daedalus-bridge`
 2. Build Daedalus Bridge

### Generating Types for Daedalus Bridge

To generate types, run

```
stack exec -- cardano-wallet-hs2purs
```

A warning message will be printed that is safe to ignore. Once the types
are generated, proceed to the next step.

### Building Daedalus Bridge

To build Daedalus Bridge, some JavaScript heavy-lifting must be done.
Currently Nix expressions don't install Node.js and NPM, so
those have to be installed manually. To do that, consult the repositories
of the package manager of according OS or download binaries from [the
net](https://nodejs.org/en/download/) you also will need the following
npm packages: `bower`, `pulp`, `purescript`. Those can be installed with

```
npm install -g bower pulp purescript
```

To build Daedalus Bridge with npm installed, run the following commands:

```
cd daedalus
npm install
```

Running `npm install` will register `daedalus-bridge` in local npm
package repository. This way, at any time, `daedalus-bridge`
dependency can be satisfied in any project that depends on it by
manually running `npm link daedalus-bridge`.

## Building Daedalus

If the instructions of building Cardano SL and the Bridge have been followed,
building Daedalus wallet is as simple as cloning Daedalus' repository:

```
git clone https://github.com/input-output-hk/daedalus.git
cd daedalus
```

Then execute the following command:

```
npm install
```

Now to run the wallet connected to the Cardano SL in dev-mode, call

```
npm run dev
```
