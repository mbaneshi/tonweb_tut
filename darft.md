[//]: # "// FIXME: "

how we can use // HACK: and other cool in md files

In dedust , we have a abstract Asset concept that refer to specific asset types.
abstraction over asset types, simplify swap process because type of asset does not matters and extra-currency or
even asset from other chains in this approach will be covered.
to adopt new asset type that future may come to existance,it just need to handle two action, recieving and transferring it.

Giving asyncronous nature and message passing nature in TON, dedust have some group of contract, responsible for
specefic part of swaping process.

First, it is factory, it is responsible for locating other contract and also creating necessary contract.

we also we two other class of contracts , vault and pool. vault is responsible for recieving message of swap ,
hold its value and inform pool that this user want to swap this value.Pool on other hand , base on provided
formula just calculates equal asset and inform another vault that pay to one who ask for swap.
we have, two type of pool ,Volatile Pool and Stable-Swap Pool ,one for normal asset that has fix ratio with
other like x \* y = k and the other used for stable coin that have close value with each others.

dedust provide special SDk to work with contract, component and API,it was written in typescript.
enough theory, lets set up our environment to swap one jetton with TON.

```bash
npm install --save @ton/core @ton/ton @ton/crypt

```

we also need to bring dedust SDK as well.

```bash
npm install --save @dedust/sdk
```

Now we need to initialize some objects.

```typescript
import { Factory, MAINNET_FACTORY_ADDR } from "@dedust/sdk";
import { Address, TonClient4 } from "@ton/ton";

const tonClient = new TonClient4({
  endpoint: "https://mainnet-v4.tonhubapi.com",
});
const factory = tonClient.open(Factory.createFromAddress(MAINNET_FACTORY_ADDR));
//The Factory contract serves used to  locate other contracts.
```

To swap Token T with J, the proces is same ,for instance we send amout of T token to vault T , vault T
receives our asset, hold it and inform pool of (T,J) that this address ask for swap ,now pool base on
calculation inform another vault, here vault J to release equivalent J to user who request swap.
The diffrent between asset is just about transfer method for example, for jettons, we transfer them to the Vault using transfer message and attach a specific forward_payload, but for the native coin, we send a swap message to the Vault, attaching the corresponding amount of TON.
So every vault and corresponding pool designed for specefic swap and has special API tailored to special asset.

Process of swapping has some steps, for example to swaping some TON with jetton we first need to find corresponding vault and pool
then make sure they are deployed.For our example TON and SCALE, code is as follows :

```typescript
import { Asset, VaultNative } from "@dedust/sdk";

//Native vault is for TON
const tonVault = tonClient.open(await factory.getNativeVault());
// we use factory to find our vault
```

next step is to find corresponding pool, here (TON and SCALE)

```typescript
import { PoolType } from "@dedust/sdk";

const SCALE_ADDRESS = Address.parse(
  "EQBlqsm144Dq6SjbPI4jjZvA1hqTIP3CvHovbIfW_t-SCALE",
);
// master address of jetton
const TON = Asset.native();
const SCALE = Asset.jetton(SCALE_ADDRESS);

const pool = tonClient.open(
  await factory.getPool(PoolType.VOLATILE, [TON, SCALE]),
);
```

Now we should ensure that these contract exis since sending funds to an inactive contract could result in irretrievable loss.

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

that is it.

This was swaping TON with jetton SCALE. The process for swapping jetton with jetton is same, the only diffrent
is we should provide payload that was described in TL-B schema.

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

Another facility that dedust SDK provided is multi hop transfer ,it means we can transfer asset even if its
pool does not exist but it is possible to swap through intermediate pool.
