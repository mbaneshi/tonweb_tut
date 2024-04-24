
</TabItem>

### How to send a swap message to DEX (DeDust)?

<TabItem value="js-tonweb" label="JS (@tonweb)">
In this tutorial, we use tonweb, one of the JS SDKs of TON to send swap messages to DeDust.All theory and
  concept that was introduced in the previous part (@ton tutorial) also applies here.

After initializing the node project, we need to bring some libraries.

```bash
npm install --save tonweb @dedust/sdk tonweb-mnemonic

```

Now it is time to bring the necessary objects to scope.

from tonweb, we need main class to interact with TON blockchain, from dedust SDK, we need bring factory address, so we can find and access to
other entity.
So : 

```typescript
const TonWeb = require("tonweb");



/* By default, mainnet toncenter.com API is used. Please note that without the API key there will be a request rate limit.

You can start your own TON HTTP API instance as it is open source.

Use main net TonCenter API with high rate-limit API key 
*/
const tonweb = new TonWeb(
  new TonWeb.HttpProvider("https://toncenter.com/api/v2/jsonRPC", {
    apiKey: "YOUR_MAINNET_TONCENTER_API_KEY",
  }),
);
```

After that, we need to use some addresses and our wallet :
```typescript

import { Factory, MAINNET_FACTORY_ADDR } from "@dedust/sdk";

//The Factory contract is used to  locate other contracts.
const factory_address = new TonWeb.Address(Factory.createFromAddress(MAINNET_FACTORY_ADDR));
// or 
 const factory_address = new TonWeb.utils.Address(Factory.createFromAddress(MAINNET_FACTORY_ADDR));
// both are correct.
```

Our wallet is needed to send messages to DEX, to use our wallet we need access to its address which can be obtained from our public key, and
also, we need access to our secret key hence we need to sign our message.
So we can use mnemonic words if we already have a wallet, or create new wallet from scratch and use their mnemonic words .
We suppose in either case we charge our wallet with some coin because without a coin we can not send any message.

```typescript

const TonWeb = require("tonweb");
const {mnemonicToKeyPair} = require("tonweb-mnemonic");
//Scenario 1, we have 24 words ( mnemonic words)

 if (!process.env.MNEMONIC) {
    throw new Error("Environment variable MNEMONIC is required.");
  }

  const mnemonic = process.env.MNEMONIC.split(" ");
// we suppose we add our words in env variable (best practices)
```
or if we have not 24 words, and we want to create it :

```typescript

const TonWeb = require("tonweb");
const {mnemonicToKeyPair} = require("tonweb-mnemonic");
import tonMnemonic from "tonweb-mnemonic";


  const mnemonic = await tonMnemonic.generateMnemonic();
    // -> ["vintage", "nice", "initial", ... ]  24 words by default


```


by accessing to mnemonic (till now we have it on hand ) we can continue by : 


```typescript

 const keyPair = await tonMnemonic.mnemonicToKeyPair(mnemonic);
    // -> {publicKey: Uint8Array(32), secretKey: Uint8Array(64)}

```
Now that we have access to our key pairs, we can access our wallet by  :

 ```typescript

  // available wallet types: simpleR1, simpleR2, simpleR3,
    // v2R1, v2R2, v3R1, v3R2, v4R1, v4R2
    const wallet = new tonweb.wallet.all['v4R2'](tonweb.provider, {
        publicKey: keyPair.publicKey,
        wc: 0 // workchain
    });

```
From here, we can use some method associated with our wallet object like transfer to send a message to corresponding vault.
to achieve this, we need to prepare our message payload. according to the dedust schema that was brought in the preface of this tutorial section we have two schema,
one for jetton to jetton and another tonconin to jetton.
scheme of toncoin to jetton swap:
```tbl
swap#ea06185d query_id:uint64 amount:Coins _:SwapStep swap_params:^SwapParams = InMsgBody;
              step#_ pool_addr:MsgAddressInt params:SwapStepParams = SwapStep;
              step_params#_ kind:SwapKind limit:Coins next:(Maybe ^SwapStep) = SwapStepParams;
              swap_params#_ deadline:Timestamp recipient_addr:MsgAddressInt referral_addr:MsgAddress
                    fulfill_payload:(Maybe ^Cell) reject_payload:(Maybe ^Cell) = SwapParams;

```
and swapping jetton to jetton or jetton to toncoin :
```tlb
swap#e3a0d482 _:SwapStep swap_params:^SwapParams = ForwardPayload;
              step#_ pool_addr:MsgAddressInt params:SwapStepParams = SwapStep;
              step_params#_ kind:SwapKind limit:Coins next:(Maybe ^SwapStep) = SwapStepParams;
              swap_params#_ deadline:Timestamp recipient_addr:MsgAddressInt referral_addr:MsgAddress
                    fulfill_payload:(Maybe ^Cell) reject_payload:(Maybe ^Cell) = SwapParams;

```
On the other hand, in tonweb sdk we can use :
```typescript
 const payload = new TonWeb.boc.Cell();
```
to prepare payload objects, that have some method to add schema :
```typescript

```
and lastly, 
```typescript
 wallet.methods
    .transfer({
      secretKey: keyPair.secretKey,
      toAddress: jettonWallet.address,
      amount: tonweb.utils.toNano("0.1"),
      seqno: await wallet.methods.seqno().call(),
      payload: jettonTransferBody,
      sendMode: 3,
    })
    .send(); // create transfer and send it
```

</TabItem>
