# Make payments to/from your account

**To make payments, both the sender and the receiver need to have a conditional deposit address (CDA). The sender needs an expired CDA that contains IOTA tokens, and the receiver needs an active CDA to send tokens to.**

## Conditions of a CDA

When you create a CDA, you specify the `timeoutAt` field to define whether it's active or expired. You can deposit IOTA tokens into an active address. But, you can't withdraw tokens from an active address. You can withdraw tokens from an expired address. But, you can't deposit tokens into an expired address.

You can also specify one of the following recommended fields:

* **multiUse (recommended):** A boolean that specifies if the address may receive more than one deposit.
* **expectedAmount (recommended):** The amount of IOTA tokens that the address is expected to receive. When the address contains this amount, it's considered expired. We recommend specifying this condition.

:::info:
You can't specify the `expected_amount` and `multi_use` fields in the same CDA. Please refer to the [FAQ](../references/cda-advice.md) for more advice about CDA conditions.
:::

|  **Combination of fields** | **Withdrawal conditions**
| :----------| :----------|
|`timeoutAt` |The CDA can used in withdrawals as long as it contains IOTA tokens|
|`timeoutAt` and `multiUse` (recommended) |The CDA can be used in withdrawals as soon as it expires, regardless of how many deposits were made to it. See the [CDA FAQ](../references/cda-advice.md) on when to use addresses with the `multiUse` field set. |
|`timeoutAt` and `expectedAmount` (recommended) | The CDA can be used in withdrawals as soon as it contain the expected amount. See the [CDA FAQ](../references/cda-advice.md) on when to use addresses with the `multi_use` field set.|

:::warning:Warning
If a CDA was created with only the `timeoutAt` field, it can be used in withdrawals as soon as it has a non-zero balance even if it hasn't expired. 

To avoid withdrawing from a spent address, we recommend creating CDAs with either the `multiUse` field or with the `expectedAmount` field whenever possible.
:::

:::info:
CDAs can be used only in an account and not in the generic client library methods. As a result, both you and the sender must have an account to be able to use CDAs.
:::

---

1. Create a CDA and set it to expire tomorrow

    ```js
    const cda = account
        .generateCDA({
            timeoutAt: Date.now() + 24 * 60 * 60 * 1000,
            expectedAmount: 1000
        });
    ```

    :::info:
    If you created an account with a `timeSource()` method, you can call that method in the `timeoutAt` field.
    :::

2. After making sure that the CDA is still active, send a deposit to it

    ```js
    cda
        .tap(cda => console.log('Sending to:', cda.address))
        .then(cda =>
            account.sendToCDA({
                ...cda,
                value: 1000
            })
            .then((trytes) => {
            console.log('Successfully prepared transaction trytes:', trytes)
        })
        )
        .catch(err => {
            // Handle errors here...
        });
    ```

    :::info:
    If you want to test this sample code with free test tokens, [request some from the Devnet faucet](root://getting-started/0.1/tutorials/receive-test-tokens.md).
    :::

    :::info:
    The last 9 characters of a CDA are the checksum, which is a hash of the address and all of its conditions. This checksum is not compatible with Trinity or the Devent faucet because they don't yet support CDAs.
    
    Remove the checksum before pasting your address into the input field of either of these applications.
    :::

In the output, you should see an address and a tail transaction hash when the transaction is pending, and the same address and tail transaction hash when the transaction is confirmed.

The account's attachment routine will keep attaching unconfirmed transactions until they are confirmed.

You can start or stop the attachment routine by calling the `startAttaching()` and
`stopAttaching()` methods.

```js
account.startAttaching({
    depth: 3,
    minWeightMagnitude: 9,
    delay: 30 * 1000

    // How far back in the Tangle to start the tip selection
    depth: 3,

    // The minimum weight magnitude is 9 on the Devnet
    minWeightMagnitude: 9,

    // How long to wait before the next attachment round
    delay: 1000 * 30,

    // The depth at which transactions are no longer promotable
    // Those transactions are automatically re-attached
    maxDepth: 6
});

account.stopAttaching();
```

## Transfer your entire available balance to one CDA

You may want to keep the majority of your balance on as few CDAs as possible. This way, making payments is faster and requires fewer transactions. To do so, you can transfer you available balance to a new CDA.

:::info:
Available balance is the total balance of all expired CDAs. This balance is safe to withdraw.

Your account's total balance includes CDAs that are still active as well as expired. This balance is unsafe to withdraw.
:::

1. Create a CDA that has your account's total available balance as its expected amount, then transfer your total available balance to that CDA

    ```js
    account.getAvailableBalance()
    .then(balance => {
        const cda = account.generateCDA({
                timeoutAt: Date.now() + 24 * 60 * 60 * 1000,
                expectedAmount: balance
            })
            .then(cda => {
                account.sendToCDA({
                ...cda,
                value: balance
            })
            .then((trytes) => {
                console.log('Successfully prepared transaction trytes:', trytes)
            })
        })
    })
    .catch(err => {
            // Handle errors here...
    });
    ```

## Send someone your CDA

If you want a depositer to transfer IOTA tokens to your account, you need to send them your CDA.

Because CDAs are descriptive objects, you can serialize them into any format before sending them. The `generateCDA()` method returns a CDA object with the following fields. You can serialize a CDA into any format before distributing it to senders.

```js
{
   address, // The last 9 trytes are the checksum
   timeoutAt,
   multiUse,
   expectedAmount
}
```

For example, you can serialize these fields to create a magnet link.

### Serialize a CDA into a magnet link

The built-in method for serializing a CDA is to create a magnet link.

1. To serialize a CDA into a magnet link, use the `serializeCDAMagnet()` method in the [`@iota/cda` module](https://github.com/iotaledger/iota.js/tree/next/packages/cda)

    ```js
    const CDA = require('@iota/cda');
    
    const magnetLink = CDA.serializeCDAMagnet(cda);
    // iota://MBREWACWIPRFJRDYYHAAME…AMOIDZCYKW/?timeout_at=1548337187&multi_use=1
    ```

2. To send a transaction to a CDA that's been serialized into a magnet link, pass the magnet link to the `parseCDAMagnet()` method, then and pass the result to the`sendToCDA()` method

    ```js
     const magnetLink = 'iota://MBREWACWIPRFJRDYYHAAME…AMOIDZCYKW/?timeout_at=1548337187&multi_use=1&expected_amount=0';
     const { address, timeoutAt, multiUse, expectedAmount } = parseCDAMagnet(
        magnetLink
    );

    account.sendToCDA({
        address,
        timeoutAt,
        multiUse,
        expectedAmount,
        value: 1000
    })
        .then((trytes) => {
            console.log('Successfully prepared the transaction:', trytes)
        })
        .catch(err => {
            // Handle errors here...
        });

    account.startAttaching({
        depth: 3,
        minWeightMagnitude : 9,
        delay: 30 * 1000 // 30 second delay
    });
    ```


