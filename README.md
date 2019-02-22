# libsigner

Manage watch only wallets with bcoin

## Features

- Node.js `Hardware` Class with Ledger Support
- CLI tooling for end to end work with `bcoin`
- Pull extended public keys, create watch only wallets/accounts
- Sign transactions, broadcast to the network
- Manage multisignature wallets

## Library Usage

`libsigner` helps to manage watch only wallets using `bcoin`.

### Exposed Classes/Functions

##### Hardware

A class to manage signing. Currently only supports Hardware devices,
but will be generalized into an abstract `Signer` in the future.

```javascript

const {Hardware,Path} = require('libsigner');

(async () => {

  // create bip44 xpub path
  const path = Path.fromList([44,0,0], true);

  const hardware = Hardware.fromOptions({
    vendor: 'ledger',    // supports ledger
    network: 'regtest',  // main, testnet, regtest, or simnet
  });
  
  const hdpubkey = await hardware.getPublicKey(path);

})().catch(e => {
  console.log(e.stack);
  process.exit(1);
});

```

Use in conjunction with [bcoin](https://github.com/bcoin-org/bcoin/)
to sign transactions using the hardware wallet device.


```javascript
const {WalletClient} = require('bclient');
const {Newtork} = require('bcoin');
const {Path,prepareSign,Hardware} = require('libsigner');

const network = Network.get('regtest');

const client = new WalletClient({
  port: network.walletPort,
  network: network.type,
});

const wallet = client.wallet('mywallet');

const hardware = Hardware.fromOptions({
  vendor: 'ledger',
  network: 'regtest',
});

const wallet = client.wallet('primary');
const path = Path.fromList([44,0,0], true);

const tx = await wallet.createTX({
  account: 'default',
  rate: 1e3,
  outputs: [{ value: 1e4, address: REaoV1gcgqDSQCkdZpjFZptGnutGEat4DR }],
  sign: false,
});

const {coins,inputTXs,paths,mtx} = await prepareSign({
  tx: tx,
  wallet: walletClient.wallet(walletId),
  path: path.clone(),
});

const signed = await hardware.signTransaction(mtx, {
  paths,
  inputTXs,
  coins,
});

console.log(signed.verify());
// true
```

Also use in conjunction with [bmultisig](https://github.com/bcoin-org/bmultisig)
to manage signing multisignature transactions

```javascript
const {WalletClient} = require('bclient');
const {Newtork} = require('bcoin');
const {Path,prepareSignMultisig,Hardware} = require('libsigner');

const network = Network.get('regtest');

const client = new WalletClient({
  port: network.walletPort,
  network: network.type,
});

const wallet = client.wallet('primary');

const proposalId = 0;
const path = Path.fromList([44,0,0], true);

const pmtx = await wallet.getProposalMTX(proposalId, {
  paths: true,
  scripts: true,
  txs: true,
});

const {paths,inputTXs,coins,scripts,mtx} = prepareSignMultisig({
  pmtx,
  path: path.clone(),
});

const signatures = await hardware.getSignature(mtx, {
  paths,
  inputTXs,
  coins,
  scripts,
  enc: 'hex',
});

const approval = await wallet.approveProposal(proposalId, signatures);

```

##### Path

A class to manage [bip44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
wallets. This class can be used in conjunction with the `Hardware` class
to make deriving keys on the device more simple.

- Create abstractions over hardened indices (no more manual bitwise or)
- Represent as string or list of uint256
- Throw errors in "strict" mode, when path depth exceeds 5
- Infer path from extended [public key](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#serialization-format)

Commonly seen notation for a hardened index includes `0'` or `0h`. Under the hood,
the hardened index is not `0`, its representations are shown below:

- `0 | 0x80000000`
- `1 << 31`
- `2147483648`
- `0b10000000000000000000000000000000`

Different paths correspond to different coins. See [slip44](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)
to learn the mapping between coin types and coins.

Create a `Path` that represents the path to the
keypair that locks a particular utxo

```javascript
const {Path} = require('libsigner');

// create a Path instance for bitcoin mainnet
const path = Path.fromList([44,0,0], true);

console.log(path.toString());
// 'm\'/44\'/0\'/0\''

console.log(path.toList());
// [ 2147483692, 2147483648, 2147483648 ]

// clone path to reuse the same
// account depth path for another tx
// from same account
let myTXPath = path.clone();

// fetch branch and index from someplace
const branch = 0;
const index = 0;

myTXPath = myTXPath.push(branch).push(index);

console.log(myTXPath.toString());
// 'm\'/44\'/0\'/0\'/0/0'

console.log(myTXPath.toList());
// [ 2147483692, 2147483648, 2147483648, 0, 0 ]

```

## CLI Usage

### pubkeys.js

Quickly pull [bip 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#serialization-format)
extended public keys from your hardware devices

```bash
$ ./bin/pubkeys.js -v ledger -n regtest -i 0h
{
  "message": "success",
  "network": "regtest",
  "path": "m'/44'/1'/0'",
  "xkey": "tpubDC2Q4xK4XH72GzfN8XLxwqyAmc71fJvWYQiwUGpTShBKRv5F6E9NtWev2ogdxB4zggRJ9BzpmJhWHsDn6vD5tJDDYHa1yoHY7JeCgenb32D",
  "publicKey": "028b42cd4776376c82791b494155151f56c2d7b471e0c7a526a7ce60dd872e3867",
  "receive": {
    "legacy": "mkpZhYtJu2r87Js3pDiWJDmPte2NRZ8bJV",
    "segwit": "bcrt1q8gk5z3dy7zv9ywe7synlrk58elz4hrnegvpv6m"
  }
}
```

### sign.js

Keep private keys on a hardware security module instead of any old machine.
Sign transactions that the bcoin wallet assmebles for watch only wallets.
Then broadcast them to the network.

##### Flags

Use the `--help` flag to see in depth details.

- `-i` - bip44 account index, must specify hardened with either `h` or `'`
- `-n` - bitcoin network, one of main, testnet, regtest, simnet
- `-v` - signing vendor, one of ledger or trezor

Lets start by verifying. Grab the first receive address of the first account.

```bash
$ receive=$(./bin/pubkeys.js -v ledger -i 0h -n regtest | jq -r .receive.legacy)
```

Now we can create a wallet using the extended public key 44h/0h/0h

```bash
$ ./bin/pubkeys.js -v ledger -i 0h -n regtest -w foo --create-wallet
```

Now lets compare the receive address that bcoin created against
the one that we derived locally

```bash
$ bwallet-cli --id foo account get default | jq -r .receiveAddress
REaoV1gcgqDSQCkdZpjFZptGnutGEat4DR
```

Sanity check to make sure they match

```bash
$ echo $receive
REaoV1gcgqDSQCkdZpjFZptGnutGEat4DR
```

If you don't already have a ton of BTC at that address, mine some real quick

```bash
$ bcoin-cli rpc generatetoaddress 300 REaoV1gcgqDSQCkdZpjFZptGnutGEat4DR
```

Now create a transaction and sign it. It will broadcast it to the network
automatically.

```bash
$ ./bin/sign.js -v ledger -w foo -n regtest --value 10000 --recipient REaoV1gcgqDSQCkdZpjFZptGnutGEat4DR
{
  "vendor": "ledger",
  "network": "regtest",
  "wallet": "foo",
  "account": "default",
  "valid": true,
  "broadcast": true,
  "hex": "010000000174fa5c5c4d870b04c47dea06f97422962d80be048413d5e787bfdc6e12c07bd0000000006b483045022100a16f35b7e7a414e5c100f362bcdc02e526ac6a962a03b955098ca5949caddd4a0220256d1d3aef9e7ea89402e3519d97d7e99c0a38022288e66363ab8a4715089c5e012102a7451395735369f2ecdfc829c0f774e88ef1303dfe5b2f04dbaab30a535dfdd6ffffffff022d260000000000001976a9143a2d4145a4f098523b3e8127f1da87cfc55b8e7988ace2de2901000000001976a914033e299551bd538711fb536beb7f99a726f24cb988ac00000000",
  "response": {
    "success": true
  }
}
```

### multisig.js

Docs coming soon

## Notes

Signing transactions with both legacy and segwit
inputs will not work on ledger and trezor hardware 
devices due to their firmware. It is possible
to craft such transactions with bcoin, so please
be careful not to do so.

## TODO

- Separate tests so that they can more easily run
- prepackage `trezor.js` post babelified, so that we do not need to include `babel-runtime` as a dependency.

## Disclaimer

This is experimental software for an experimental protocol.
Please do your own research and understand the code if you
decide to use it with real money.
