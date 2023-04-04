## WARNING

For REMP tests only!

## EVER Wallet contract

The simplest yet powerful ABI 2.3 compliant wallet.

* Doesn't require deployment.
* The code takes only ~232 bytes and 6 cells.
* Supports transfers with state init.
* Can send up to 4 messages at the same time.

<details><summary><b>Code</b></summary>
<p>

`te6cckEBBgEA/AABFP8A9KQT9LzyyAsBAgEgAgMABNIwAubycdcBAcAA8nqDCNcY7UTQgwfXAdcLP8j4KM8WI88WyfkAA3HXAQHDAJqDB9cBURO68uBk3oBA1wGAINcBgCDXAVQWdfkQ8qj4I7vyeWa++COBBwiggQPoqFIgvLHydAIgghBM7mRsuuMPAcjL/8s/ye1UBAUAmDAC10zQ+kCDBtcBcdcBeNcB10z4AHCAEASqAhSxyMsFUAXPFlAD+gLLaSLQIc8xIddJoIQJuZgzcAHLAFjPFpcwcQHLABLM4skB+wAAPoIQFp4+EbqOEfgAApMg10qXeNcB1AL7AOjRkzLyPOI+zYS/`

</p>
</details>

<details><summary><b>ABI</b></summary>
<p>

```
{
  "ABI version": 2,
  "version": "2.3",
  "header": ["pubkey", "time", "expire"],
  "functions": [
    {
      "name": "sendTransaction",
      "inputs": [
        { "name": "dest", "type": "address" },
        { "name": "value", "type": "uint128" },
        { "name": "bounce", "type": "bool" },
        { "name": "flags", "type": "uint8" },
        { "name": "payload", "type": "cell" }
      ],
      "outputs": []
    },
    {
      "name": "sendTransactionRaw",
      "inputs": [
        { "name": "flags", "type": "uint8" },
        { "name": "message", "type": "cell" }
      ],
      "outputs": []
    }
  ],
  "data": [],
  "events": [],
  "fields": [
    { "name": "_pubkey", "type": "uint256" },
    { "name": "_timestamp", "type": "uint64" }
  ]
}
```

</p>
</details>

**Code hash:** `3ba6528ab2694c118180aa3bd10dd19ff400b909ab4dcf58fc69925b2c7b12a6`

### How to build and test

> Requires `func` and `fift` binaries to be visible in `$PATH`

```bash
npm install

export FIFTPATH="/path/to/fift/lib"
npm run build
npm run test
```

### How to use

#### 1. Compute state init and address

```typescript
const WalletCode = 'te6cckEBBgEA/AABFP8A9KQT9LzyyAsBAgEgAgMABNIwAubycdcBAcAA8nqDCNcY7UTQgwfXAdcLP8j4KM8WI88WyfkAA3HXAQHDAJqDB9cBURO68uBk3oBA1wGAINcBgCDXAVQWdfkQ8qj4I7vyeWa++COBBwiggQPoqFIgvLHydAIgghBM7mRsuuMPAcjL/8s/ye1UBAUAmDAC10zQ+kCDBtcBcdcBeNcB10z4AHCAEASqAhSxyMsFUAXPFlAD+gLLaSLQIc8xIddJoIQJuZgzcAHLAFjPFpcwcQHLABLM4skB+wAAPoIQFp4+EbqOEfgAApMg10qXeNcB1AL7AOjRkzLyPOI+zYS/';

const {boc: data} = await ever.packIntoCell({
  structure: [
    {name: 'publicKey', type: 'uint256'},
    {name: 'timestamp', type: 'uint64'},
  ] as const,
  data: {
    publicKey: `0x${walletPublicKey}`,
    timestamp: 0,
  },
});

const {tvc} = await ever.mergeTvc({data, code: WalletCode});
const tvcHash = await ever.getBocHash(tvc);

const stateInit = tvc;
const address = `0:${tvcHash}`;
```

> NOTE: initial data must be constructed manually because the linker uses mapping for this, which will require a separate action to deploy.

#### 2. Transfer

```typescript
const WalletAbi = {
  "ABI version": 2,
  "version": "2.3",
  "header": ["pubkey", "time", "expire"],
  "functions": [
    {
      "name": "sendTransaction",
      "inputs": [
        {"name": "dest", "type": "address"},
        {"name": "value", "type": "uint128"},
        {"name": "bounce", "type": "bool"},
        {"name": "flags", "type": "uint8"},
        {"name": "payload", "type": "cell"}
      ],
      "outputs": []
    }
  ],
  "events": [],
} as const;

const wallet = new ever.Contract(WalletAbi, walletAddress);

const walletState = await ever.getFullContractState({address: wallet.address});
const stateInit = !walletState?.isDeployed ? stateInit : undefined;

const {transaction} = await wallet.methods
  .sendTransaction({
    dest: recipientAddress,
    value: '1000000000', // amount in nano EVER
    bounce: false,
    flags: 3,
    payload: ''
  }).sendExternal({
    publicKey: walletPublicKey,
    stateInit,
  });

console.log(transaction);
```

### Message flags

`flags` param modifies message behavior. It is just a logical OR of different parts:

* 1 - sender wants to pay transfer fees separately (from account balance instead of message balance)
* 2 - if there are some errors during the action phase it should be ignored (don't fail transaction e.g. if message balance
  is greater than remaining balance, or it has invalid address)
* 32 - current account must be destroyed if its resulting balance is zero (usually combined with flag 128)
* 64 - used for messages that carry all the remaining value of the inbound message in addition to the value initially
  indicated in the new message
* 128 - message will carry all the remaining balance

Common combinations:

* Regular transfer: **`3`** (pay fees separately and ignore action phase errors)
* Send all balance: **`130`** (attach all the remaining balance and ignore action phase errors)
* Send all balance and destroy account: **`162`** (attach all the remaining balance, destroy account and ignore action phase errors)

### Error codes

* 40 - External inbound message has an invalid signature.
* 52 - Replay protection exception.
* 57 - External inbound message is expired.
* 58 - External inbound message has no signature but has public key.
* 60 - Inbound message has wrong function id.
* 100 - Message sender is not a custodian.
