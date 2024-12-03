# pumpfun-buy-sell-bot
An automated Solana Pump.fun bot designed for seamless buy-and-sell execution, maximizing efficiency and profitability in real-time trading.


## Contact Info
If you need more technical support and development inquires, you can contact below.

Telegram: [@diasibt](https://t.me/@diasibt)

X: [@DiasIbt101](https://x.com/DiasIbt101)

Discord: [@dias_ishbulatov](https://discordapp.com/users/1213745904599961631)



## **Installation:**

```bash
# Clone the repository 
git clone https://github.com/solguru310/pumpfun-buy-sell-bot.git

# Install dependencies 
cd pumpfun-buy-sell-bot
npm install 
```

### Core Functionality 

**BondingCurveAccount:**
```javascript
  getBuyPrice(amount: bigint): bigint {
    if (this.complete) {
      throw new Error("Curve is complete");
    }

    if (amount <= 0n) {
      return 0n;
    }

    // Calculate the product of virtual reserves
    let n = this.virtualSolReserves * this.virtualTokenReserves;

    // Calculate the new virtual sol reserves after the purchase
    let i = this.virtualSolReserves + amount;

    // Calculate the new virtual token reserves after the purchase
    let r = n / i + 1n;

    // Calculate the amount of tokens to be purchased
    let s = this.virtualTokenReserves - r;

    // Return the minimum of the calculated tokens and real token reserves
    return s < this.realTokenReserves ? s : this.realTokenReserves;
  }

  getSellPrice(amount: bigint, feeBasisPoints: bigint): bigint {
    if (this.complete) {
      throw new Error("Curve is complete");
    }

    if (amount <= 0n) {
      return 0n;
    }

    // Calculate the proportional amount of virtual sol reserves to be received
    let n =
      (amount * this.virtualSolReserves) / (this.virtualTokenReserves + amount);

    // Calculate the fee amount in the same units
    let a = (n * feeBasisPoints) / 10000n;

    // Return the net amount after deducting the fee
    return n - a;
  }
```


**Events:**
```javascript
export function toCreateEvent(event: CreateEvent): CreateEvent {
  return {
    name: event.name,
    symbol: event.symbol,
    uri: event.uri,
    mint: new PublicKey(event.mint),
    bondingCurve: new PublicKey(event.bondingCurve),
    user: new PublicKey(event.user),
  };
}

export function toCompleteEvent(event: CompleteEvent): CompleteEvent {
  return {
    user: new PublicKey(event.user),
    mint: new PublicKey(event.mint),
    bondingCurve: new PublicKey(event.bondingCurve),
    timestamp: event.timestamp,
  };
}

export function toTradeEvent(event: TradeEvent): TradeEvent {
  return {
    mint: new PublicKey(event.mint),
    solAmount: BigInt(event.solAmount),
    tokenAmount: BigInt(event.tokenAmount),
    isBuy: event.isBuy,
    user: new PublicKey(event.user),
    timestamp: Number(event.timestamp),
    virtualSolReserves: BigInt(event.virtualSolReserves),
    virtualTokenReserves: BigInt(event.virtualTokenReserves),
    realSolReserves: BigInt(event.realSolReserves),
    realTokenReserves: BigInt(event.realTokenReserves),
  };
}

export function toSetParamsEvent(event: SetParamsEvent): SetParamsEvent {
  return {
    feeRecipient: new PublicKey(event.feeRecipient),
    initialVirtualTokenReserves: BigInt(event.initialVirtualTokenReserves),
    initialVirtualSolReserves: BigInt(event.initialVirtualSolReserves),
    initialRealTokenReserves: BigInt(event.initialRealTokenReserves),
    tokenTotalSupply: BigInt(event.tokenTotalSupply),
    feeBasisPoints: BigInt(event.feeBasisPoints),
  };
}
```

**Global Account:**
```javascript
export class GlobalAccount {
  public discriminator: bigint;
  public initialized: boolean = false;
  public authority: PublicKey;
  public feeRecipient: PublicKey;
  public initialVirtualTokenReserves: bigint;
  public initialVirtualSolReserves: bigint;
  public initialRealTokenReserves: bigint;
  public tokenTotalSupply: bigint;
  public feeBasisPoints: bigint;

  constructor(
    discriminator: bigint,
    initialized: boolean,
    authority: PublicKey,
    feeRecipient: PublicKey,
    initialVirtualTokenReserves: bigint,
    initialVirtualSolReserves: bigint,
    initialRealTokenReserves: bigint,
    tokenTotalSupply: bigint,
    feeBasisPoints: bigint
  ) {
    this.discriminator = discriminator;
    this.initialized = initialized;
    this.authority = authority;
    this.feeRecipient = feeRecipient;
    this.initialVirtualTokenReserves = initialVirtualTokenReserves;
    this.initialVirtualSolReserves = initialVirtualSolReserves;
    this.initialRealTokenReserves = initialRealTokenReserves;
    this.tokenTotalSupply = tokenTotalSupply;
    this.feeBasisPoints = feeBasisPoints;
  }

  getInitialBuyPrice(amount: bigint): bigint {
    if (amount <= 0n) {
      return 0n;
    }

    let n = this.initialVirtualSolReserves * this.initialVirtualTokenReserves;
    let i = this.initialVirtualSolReserves + amount;
    let r = n / i + 1n;
    let s = this.initialVirtualTokenReserves - r;
    return s < this.initialRealTokenReserves
      ? s
      : this.initialRealTokenReserves;
  }

  public static fromBuffer(buffer: Buffer): GlobalAccount {
    const structure: Layout<GlobalAccount> = struct([
      u64("discriminator"),
      bool("initialized"),
      publicKey("authority"),
      publicKey("feeRecipient"),
      u64("initialVirtualTokenReserves"),
      u64("initialVirtualSolReserves"),
      u64("initialRealTokenReserves"),
      u64("tokenTotalSupply"),
      u64("feeBasisPoints"),
    ]);

    let value = structure.decode(buffer);
    return new GlobalAccount(
      BigInt(value.discriminator),
      value.initialized,
      value.authority,
      value.feeRecipient,
      BigInt(value.initialVirtualTokenReserves),
      BigInt(value.initialVirtualSolReserves),
      BigInt(value.initialRealTokenReserves),
      BigInt(value.tokenTotalSupply),
      BigInt(value.feeBasisPoints)
    );
  }
}
```

...........

Also There are many other part of the project!!!

## Main Part of Project

### Index.ts

```javascript
const main = async () => {
  try {
    console.log(
      (await connection.getBalance(mainKp.publicKey)) / 10 ** 9,
      "SOL in main keypair"
    );

    console.log(mintAddress);

    try {
      const tokenBuyix = await makeBuyIx(
        mainKp,
        Math.floor(SWAP_AMOUNT * 10 ** 9)
      );

      if (!tokenBuyix) {
        console.log("Token buy instruction not retrieved");
        return;
      }
      const tx = new Transaction().add(
        ComputeBudgetProgram.setComputeUnitPrice({
          microLamports: 100_000,
        }),
        ComputeBudgetProgram.setComputeUnitLimit({
          units: 200_000,
        }),
        ...tokenBuyix
      );

      tx.feePayer = mainKp.publicKey;
      tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

      console.log(await connection.simulateTransaction(tx));

      const signature = await sendAndConfirmTransaction(
        connection,
        tx,
        [mainKp],
        { skipPreflight: true, commitment: commitment }
      );

      console.log(`Buy Tokens : https://solscan.io/tx/${signature}`);
    } catch (error) {
    }

    try {
      const tokenAccount = await getAssociatedTokenAddress(
        mintAddress,
        mainKp.publicKey
      );

      const tokenBalance = (
        await connection.getTokenAccountBalance(tokenAccount)
      ).value.amount;

      if (tokenBalance) {
        const tokenSellix = await makeSellIx(mainKp, Number(tokenBalance));
        console.log(tokenSellix);
        if (!tokenSellix) {
          console.log("Token buy instruction not retrieved");
          return;
        }

        const tx = new Transaction().add(
          ComputeBudgetProgram.setComputeUnitPrice({
            microLamports: 100_000,
          }),
          ComputeBudgetProgram.setComputeUnitLimit({
            units: 200_000,
          }),
          tokenSellix
        );

        tx.feePayer = mainKp.publicKey;
        tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

        console.log(await connection.simulateTransaction(tx));

        const signature = await sendAndConfirmTransaction(
          connection,
          tx,
          [mainKp],
          { skipPreflight: true, commitment: commitment }
        );

        console.log(`Sell Tokens : https://solscan.io/tx/${signature}`);
      }
    } catch (error) {
    }
  } catch (error) {
    console.log("Token trading error");
  }
};
```

