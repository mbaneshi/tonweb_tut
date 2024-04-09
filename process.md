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
```typescript
const TonWeb = require("tonweb");
const {mnemonicToKeyPair} = require("tonweb-mnemonic");

  const mnemonic = await tonMnemonic.generateMnemonic();
    // -> ["vintage", "nice", "initial", ... ]  24 words by default

 const keyPair = await tonMnemonic.mnemonicToKeyPair(mnemonic);
    // -> {publicKey: Uint8Array(32), secretKey: Uint8Array(64)}


import { Factory, MAINNET_FACTORY_ADDR } from "@dedust/sdk";

/* By default, mainnet toncenter.com API is used. Please note that without the API key there will be a request rate limit.

You can start your own TON HTTP API instance as it is open source.

Use mainnet TonCenter API with high ratelimit API key 
*/
const tonweb = new TonWeb(
  new TonWeb.HttpProvider("https://toncenter.com/api/v2/jsonRPC", {
    apiKey: "YOUR_MAINNET_TONCENTER_API_KEY",
  }),
);

```

After that we need use some address and our wallet :

```typescript

//The Factory contract serves used to  locate other contracts.
const address1 = new TonWeb.utils.Address('EQDKbjIcfM6ezt8KjKJJLshZJJSqX7XOA4ff-W72r5gqPrHF');
  // available wallet types: simpleR1, simpleR2, simpleR3,
    // v2R1, v2R2, v3R1, v3R2, v4R1, v4R2
    const wallet = new tonweb.wallet.all['v4R2'](tonweb.provider, {
        publicKey: keyPair.publicKey,
        wc: 0 // workchain
    });

```

