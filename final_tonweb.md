
</TabItem>

### How to send swap message to DEX (DeDust)?

<TabItem value="js-tonweb" label="JS (@tonweb)">
In this tutorial, we use tonweb, one of the JS SDKs of TON to send swap messages to DeDust.All theory and
  concept that was introduced in the previous part (@ton tutorial) also applies here.

After initializing the node project, we need to bring some libraries.

```bash
npm install --save tonweb @dedust/sdk
```

Now it is time to bring the necessary objects to scope.

```typescript
const TonWeb = require("tonweb");

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
const factory = tonClient.open(Factory.createFromAddress(MAINNET_FACTORY_ADDR));
//The Factory contract serves used to  locate other contracts.
```

```typescript
import { Address, TonClient4, WalletContractV3R2 } from "@ton/ton";
import { mnemonicToPrivateKey } from "@ton/crypto";
import { Asset, Factory, MAINNET_FACTORY_ADDR } from "@dedust/sdk";

async function main() {
  if (!process.env.MNEMONIC) {
    throw new Error("Environment variable MNEMONIC is required.");
  }

  const mnemonic = process.env.MNEMONIC.split(" ");

  if (!process.env.JETTON_ADDRESS) {
    throw new Error("Environment variable JETTON_ADDRESS is required.");
  }

  const jettonAddress = Address.parse(process.env.JETTON_ADDRESS);

  const tonClient = new TonClient4({
    endpoint: "https://mainnet-v4.tonhubapi.com",
  });

  const factory = tonClient.open(
    Factory.createFromAddress(MAINNET_FACTORY_ADDR),
  );

  const keys = await mnemonicToPrivateKey(mnemonic);
  const wallet = tonClient.open(
    WalletContractV3R2.create({
      workchain: 0,
      publicKey: keys.publicKey,
    }),
  );

  const sender = wallet.sender(keys.secretKey);

  await factory.sendCreateVault(sender, {
    asset: Asset.jetton(jettonAddress),
  });
}

main();
// HACK: this is dedust refrence
```

```typescript
// FIXME: we bring some tutorial from cookbook here .
//    How to construct a message for a jetton transfer with a comment?
const TonWeb = require("tonweb");
const { mnemonicToKeyPair } = require("tonweb-mnemonic");

async function main() {
  const tonweb = new TonWeb(
    new TonWeb.HttpProvider("https://toncenter.com/api/v2/jsonRPC", {
      apiKey: "put your api key",
    }),
  );
  const destinationAddress = new TonWeb.Address(
    "put destination wallet address",
  );

  const forwardPayload = new TonWeb.boc.Cell();
  forwardPayload.bits.writeUint(0, 32); // 0 opcode means we have a comment
  forwardPayload.bits.writeString("Hello, TON!");

  /*
        Tonweb has a built-in class for interacting with jettons, which has
        a method for creating a transfer. However, it has disadvantages, so
        we manually create the message body. Additionally, this way we have a
        better understanding of what is stored and how it functions.
     */

  const jettonTransferBody = new TonWeb.boc.Cell();
  jettonTransferBody.bits.writeUint(0xf8a7ea5, 32); // opcode for jetton transfer
  jettonTransferBody.bits.writeUint(0, 64); // query id
  jettonTransferBody.bits.writeCoins(new TonWeb.utils.BN("5")); // jetton amount, amount * 10^9
  jettonTransferBody.bits.writeAddress(destinationAddress);
  jettonTransferBody.bits.writeAddress(destinationAddress); // response destination
  jettonTransferBody.bits.writeBit(false); // no custom payload
  jettonTransferBody.bits.writeCoins(TonWeb.utils.toNano("0.02")); // forward amount
  jettonTransferBody.bits.writeBit(true); // we store forwardPayload as a reference
  jettonTransferBody.refs.push(forwardPayload);

  const keyPair = await mnemonicToKeyPair("put your mnemonic".split(" "));
  const jettonWallet = new TonWeb.token.ft.JettonWallet(tonweb.provider, {
    address: "put your jetton wallet address",
  });

  // available wallet types: simpleR1, simpleR2, simpleR3,
  // v2R1, v2R2, v3R1, v3R2, v4R1, v4R2
  const wallet = new tonweb.wallet.all["v4R2"](tonweb.provider, {
    publicKey: keyPair.publicKey,
    wc: 0, // workchain
  });

  await wallet.methods
    .transfer({
      secretKey: keyPair.secretKey,
      toAddress: jettonWallet.address,
      amount: tonweb.utils.toNano("0.1"),
      seqno: await wallet.methods.seqno().call(),
      payload: jettonTransferBody,
      sendMode: 3,
    })
    .send(); // create transfer and send it
}

main().finally(() => console.log("Exiting..."));
```

```typescript
//FIXME: How to change the owner of a collection's smart contract?
const TonWeb = require("tonweb");
const { mnemonicToKeyPair } = require("tonweb-mnemonic");

async function main() {
  const tonweb = new TonWeb(
    new TonWeb.HttpProvider("https://toncenter.com/api/v2/jsonRPC", {
      apiKey: "put your api key",
    }),
  );
  const collectionAddress = new TonWeb.Address("put your collection address");
  const newOwnerAddress = new TonWeb.Address("put new owner wallet address");

  const messageBody = new TonWeb.boc.Cell();
  messageBody.bits.writeUint(3, 32); // opcode for changing owner
  messageBody.bits.writeUint(0, 64); // query id
  messageBody.bits.writeAddress(newOwnerAddress);

  // available wallet types: simpleR1, simpleR2, simpleR3,
  // v2R1, v2R2, v3R1, v3R2, v4R1, v4R2
  const keyPair = await mnemonicToKeyPair("put your mnemonic".split(" "));
  const wallet = new tonweb.wallet.all["v4R2"](tonweb.provider, {
    publicKey: keyPair.publicKey,
    wc: 0, // workchain
  });

  await wallet.methods
    .transfer({
      secretKey: keyPair.secretKey,
      toAddress: collectionAddress,
      amount: tonweb.utils.toNano("0.05"),
      seqno: await wallet.methods.seqno().call(),
      payload: messageBody,
      sendMode: 3,
    })
    .send(); // create transfer and send it
}

main().finally(() => console.log("Exiting..."));
```

<!---->
<!-- DEXes use different protocols for their work, we need to familiar ourself with key concepts and some vital component, and also knowing TL-B schema that involved to do our swap process correctly. In this tutorial we deal with DeDust, one of famous DEX implemented entirely in TON. -->
<!---->
<!-- In DeDust , we have a abstract Asset concept that simply include any swapable asset types. Abstraction over asset types, simplify swap process because type of asset does not matters and extra-currency or even asset from other chains in this approach will be covered by ease. -->
<!---->
<!-- Following is TL-B schema that DeDust introduced for Asset concept. -->
<!---->
<!-- ```tlb -->
<!-- native$0000 = Asset; // for ton -->
<!---->
<!-- jetton$0001 workchain_id:int8 address:uint256 = Asset; // for any jetton,address refer to jetton master address -->
<!---->
<!-- // Upcoming, not implemented yet. -->
<!-- extra_currency$0010 currency_id:int32 = Asset; -->
<!-- ``` -->
<!---->
<!-- Next, DeDust introduced three components, Vault, Pool, and Factory.These components are contract or group of -->
<!-- contracts and responsible for parts of swap process. Factory act as finding other component addresses (factory, vault and pool) -->
<!-- and also building other components. -->
<!-- Vault is responsible for receiving transfer messages, holding assets and just informing corresponding pool that "user A wants to swap 100 X to Y". -->
<!---->
<!-- Pool on the other hand, responsible for calculating swap amout based on predefined formula and inform other Vault that responsible for asset Y , and tell it to pay calculated amount to user. -->
<!-- Calculations of swap amount is based on mathematical formula,it means so far we have two different pool, one known as Volatile, that operates based on the commonly used "Constant Product" formula: x \* y = k, And the other known as Stable-Swap - Optimized for assets of near-equal value (e.g. USDT / USDC, TON / stTON). It uses the formula: x3 • y + y3 • x = k. -->
<!-- So for every swap we need corresponding Valut and it need just implemented specefic API tailored for interacting with a distinct asset type. DeDust have three implementations of Vault, Native Vault - Handles the native coin (Toncoin). Jetton Vault - Manages jettons and Extra-Currency Vault (upcoming) - Designed for TON extra-currencies. -->
<!---->
<!-- DeDust provide special SDk to work with contract, component and API, it was written in typescript. -->
<!-- Enough theory, lets set up our environment to swap one jetton with TON. -->
<!---->
<!-- ```bash -->
<!-- npm install --save @ton/core @ton/ton @ton/crypt -->
<!---->
<!-- ``` -->
<!---->
<!-- we also need to bring DeDust SDK as well. -->
<!---->
<!-- ```bash -->
<!-- npm install --save @dedust/sdk -->
<!-- ``` -->
<!---->
<!-- Now we need to initialize some objects. -->
<!---->

```typescript
import { Factory, MAINNET_FACTORY_ADDR } from "@dedust/sdk";
import { Address, TonClient4 } from "@ton/ton";

const tonClient = new TonClient4({
  endpoint: "https://mainnet-v4.tonhubapi.com",
});
const factory = tonClient.open(Factory.createFromAddress(MAINNET_FACTORY_ADDR));
//The Factory contract  used to  locate other contracts.
```

Process of swapping has some steps, for example to swaping some TON with jetton we first need to find corresponding Vault and Pool
then make sure they are deployed.For our example TON and SCALE, code is as follows :

```typescript
import { Asset, VaultNative } from "@dedust/sdk";

//Native vault is for TON
const tonVault = tonClient.open(await factory.getNativeVault());
// we use factory to find our native coin (Toncoin) Vault.
```

Next step is to find corresponding Pool, here (TON and SCALE)

```typescript
import { PoolType } from "@dedust/sdk";

const SCALE_ADDRESS = Address.parse(
  "EQBlqsm144Dq6SjbPI4jjZvA1hqTIP3CvHovbIfW_t-SCALE",
);
// master address of SCALE jetton
const TON = Asset.native();
const SCALE = Asset.jetton(SCALE_ADDRESS);

const pool = tonClient.open(
  await factory.getPool(PoolType.VOLATILE, [TON, SCALE]),
);
```

Now we should ensure that these contract exist, since sending funds to an inactive contract could result in irretrievable loss.

```typescript
import { ReadinessStatus } from "@dedust/sdk";

// Check if pool exists:
if ((await pool.getReadinessStatus()) !== ReadinessStatus.READY) {
  throw new Error("Pool (TON, SCALE) does not exist.");
}

// Check if vault exits:
if ((await tonVault.getReadinessStatus()) !== ReadinessStatus.READY) {
  throw new Error("Vault (TON) does not exist.");
}
```

After that we can send transfer messages with amount of TON

```typescript
import { toNano } from "@ton/core";

const amountIn = toNano("5"); // 5 TON

await tonVault.sendSwap(sender, {
  poolAddress: pool.address,
  amount: amountIn,
  gasAmount: toNano("0.25"),
});
```

To swap Token X with Y, the process is same ,for instance we send amout of X token to vault X , vault X
receives our asset, hold it and inform Pool of (X,Y) that this address ask for swap, now Pool base on
calculation inform another Vault, here Vault Y to release equivalent Y to user who request swap.

The difference between asset is just about transfer method for example, for jettons, we transfer them to the Vault using transfer message and attach a specific forward_payload, but for the native coin, we send a swap message to the Vault, attaching the corresponding amount of TON.

This is schema for TON and jetton :

```tlb
swap#ea06185d query_id:uint64 amount:Coins _:SwapStep swap_params:^SwapParams = InMsgBody;
```

So every vault and corresponding Pool designed for specefic swap and has special API tailored to special asset.

This was swaping TON with jetton SCALE. The process for swapping jetton with jetton is same, the only difference is we should provide payload that was described in TL-B schema.

```TL-B
swap#e3a0d482 _:SwapStep swap_params:^SwapParams = ForwardPayload;
```

```typescript
//find Vault
const scaleVault = tonClient.open(await factory.getJettonVault(SCALE_ADDRESS));
```

```typescript
//find jetton address
import { JettonRoot, JettonWallet } from '@dedust/sdk';

const scaleRoot = tonClient.open(JettonRoot.createFromAddress(SCALE_ADDRESS));
const scaleWallet = tonClient.open(await scaleRoot.getWallet(sender.address);

// Transfer jettons to the Vault (SCALE) with corresponding payload

const amountIn = toNano('50'); // 50 SCALE

await scaleWallet.sendTransfer(sender, toNano("0.3"), {
  amount: amountIn,
  destination: scaleVault.address,
  responseAddress: sender.address, // return gas to user
  forwardAmount: toNano("0.25"),
  forwardPayload: VaultJetton.createSwapPayload({ poolAddress }),
});
```

</TabItem>
