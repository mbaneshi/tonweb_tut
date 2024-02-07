### How to send swap message to DEX (DeDust)?

<TabItem value="js-ton" label="JS (@ton)">
DEXes use different protocols for their work, we need to familiar ourself with key concepts and some vital component, and also knowing TL-B schema that involved to do our swap process correctly. In this tutorial we deal with DeDust, one of famous DEX implemented entirely in TON.

In DeDust , we have a abstract Asset concept that simply include any swapable asset types. Abstraction over asset types, simplify swap process because type of asset does not matters and extra-currency or even asset from other chains in this approach will be covered by ease.

Following is TL-B schema that DeDust introduced for Asset concept.

```tlb
native$0000 = Asset; // for ton

jetton$0001 workchain_id:int8 address:uint256 = Asset; // for any jetton,address refer to jetton master address

// Upcoming, not implemented yet.
extra_currency$0010 currency_id:int32 = Asset;
```

Next, DeDust introduced three components, Vault, Pool, and Factory.These components are contract or group of
contracts and responsible for parts of swap process. Factory act as finding other component addresses (factory, vault and pool)
and also building other components.
Vault is responsible for receiving transfer messages, holding assets and just informing corresponding pool that "user A wants to swap 100 X to Y".

Pool on the other hand, responsible for calculating swap amout based on predefined formula and inform other Vault that responsible for asset Y , and tell it to pay calculated amount to user.
Calculations of swap amount is based on mathematical formula,it means so far we have two different pool, one known as Volatile, that operates based on the commonly used "Constant Product" formula: x \* y = k, And the other known as Stable-Swap - Optimized for assets of near-equal value (e.g. USDT / USDC, TON / stTON). It uses the formula: x3 • y + y3 • x = k.
So for every swap we need corresponding Valut and it need just implemented specefic API tailored for interacting with a distinct asset type. DeDust have three implementations of Vault, Native Vault - Handles the native coin (Toncoin). Jetton Vault - Manages jettons and Extra-Currency Vault (upcoming) - Designed for TON extra-currencies.

DeDust provide special SDk to work with contract, component and API, it was written in typescript.
Enough theory, lets set up our environment to swap one jetton with TON.

```bash
npm install --save @ton/core @ton/ton @ton/crypt

```

we also need to bring DeDust SDK as well.

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
