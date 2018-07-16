# Urkel Tree

An optimized and cryptographically provable key-value store.

## Design

The urkel tree was created for the [Handshake protocol][1], and is implemented
as a base-2 merkelized trie. It was created as an alternative to [Ethereum's
base-16 trie][2] (which was the initial choice for Handshake name proofs). The
design was inspired by Amaury Séchet's [Merklix tree][3] and shares many
similarities to earlier work done by [Bram Cohen][4].

Urkel stores nodes in a series of append-only files for snapshotting and crash
consistency capabilities. Due to these presence of these features, Urkel has
the ability to expose a fully transactional database.

The primary advantages in using an urkel tree over something like Ethereum's
trie are:

- __Performance__ - Stores nodes in flat files instead of an existing key-value
  store like LevelDB. Urkel is its _own_ database. In benchmarks, this results
  in a 100x+ speedup.
- __Simplicity__ - Maintains only two types of nodes: internal nodes and leaf
  nodes.
- __Storage__ - Internal nodes are small (a constant size of 77 bytes on disk).
  This is important as internal nodes are frequently rewritten during updates
  to the tree.
- __Proof Size__ - Sibling nodes required for proofs are a constant size of 32
  bytes, similar to a typical merkle tree proof. This results in an extremely
  compact proof size.

The final benefit was the primary focus of the Handshake protocol. As name
resolutions are a frequently requested operation, Handshake required proof
sizes less than 1kb even after hundreds of millions of leaves are present in
the tree.

History independence and non-destruction are also inherent properties of the
urkel tree, just the same as the Ethereum trie. Note that urkel should only be
used with uniformally distributed keys (i.e. hashed).

Compaction, while available, is currently inefficient and requires user
intervention. This will be optimized in a future C implementation of the urkel
tree. In the meantime, we don't see this as a problem as long as frequent
commissions are avoided in consensus applications of the tree (i.e. avoid
committing the tree on every block).

A more in-depth description is available in the [Handshake Whitepaper][5].

## Usage

``` js
const assert = require('assert');
const crypto = require('bcrypto');
const urkel = require('urkel');
const {Tree} = urkel;
const {Blake2b, randomBytes} = crypto;

const tree = new Tree(Blake2b, 256, '/path/to/my/db');

await tree.open();

let key;

const txn = tree.txn();

for (let i = 0; i < 500; i++) {
  const k = randomBytes(32);
  const v = randomBytes(300);

  await txn.insert(k, v);

  key = key;
}

await txn.commit();

const snapshot = tree.snapshot();
const root = snapshot.rootHash();
const proof = await snapshot.prove(key);
const [code, value] = proof.verify(root, key, Blake2b, 256);

assert(code === 0);

if (value) {
  console.log('Valid proof for %s: %s',
    key.toString('hex'), value.toString('hex'));
} else {
  console.log('Absence proof for %s.', key.toString('hex'));
}

await snapshot.range((key, value) => {
  console.log('Iterated over item:');
  console.log('%s: %s', key.toString('hex'), value.toString('hex'));
});

await tree.close();
```

## Contribution and License Agreement

If you contribute code to this project, you are implicitly allowing your code
to be distributed under the MIT license. You are also implicitly verifying that
all code is your original work. `</legalese>`

## License

- Copyright (c) 2018, Christopher Jeffrey (MIT License).

See LICENSE for more info.

[1]: https://handshake.org
[2]: https://github.com/ethereum/wiki/wiki/Patricia-Tree
[3]: https://www.deadalnix.me/2016/09/24/introducing-merklix-tree-as-an-unordered-merkle-tree-on-steroid/
[4]: https://github.com/bramcohen/MerkleSet
[5]: https://handshake.org/paper
