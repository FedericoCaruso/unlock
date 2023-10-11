# Cross Chain Governance

To simplify the maintenance and management of the protocol contracts, we use a cross-chain governance that allows for a DAO proposal on mainnet to reach contracts on other networks.

# How it works

To reach other chains, calls emitted from the mainnet DAO go though the [Connext bridge](https://www.connext.network/) and are executed on the other side of the bridge, after a period of cooldown.

```
(mainnet)                       (destination chain)    (cooldown period)
DAO proposal >  Connext Bridge >   Safe multisig    >  wait for 7 days   >
```

The workflow is as follow

1. a DAO proposal is created, containing 1 call per chain
2. if the vote succeeds, the DAO proposal is executed. All calls are sent to the bridge(s).
3. each call crosses the bridge seperately towards its destination on a specific chain
4. the call is received on the destination chain by Unlock multisig
5. once received, the call is held in the multisig for a period of X days during which it can be cancelled.
6. once the cooldown period ends, the call is executed

NB: The cooldown period is useful to prevent malicious or errored calls from being executed

The Safes still own Unlock contracts and Proxy Admins on destination networks (no changes in governance here).

# Supported Networks

(as Oct 10, 2023)

|                    | Unlock Protocol | Safe | Connext |
| ------------------ | --------------- | ---- | ------- |
| Ethereum Mainnet   | x               | x    | x       |
| Gnosis             | x               | x    | x       |
| Polygon            | x               | x    | x       |
| Optimism           | x               | x    | x       |
| Arbitrum One       | x               | x    | x       |
| BNB Chain          | x               | x    | x       |
| Avalanche          | x               | x    |         |
| Base               | x               | x    |         |
| Celo               | x               | x    |         |
| Palm               | x               |      |         |
| Polygon zkEVM      |                 | x    |         |
| zkSync Era mainnet |                 | x    |         |
| Aurora             |                 | x    |         |

# How to setup the multisig

On every receiving chain, we need a Gnosis Safe configured with two Zodiac plugins

- one Connext bridge receiver
- one Delay

1. In Gnosis web UI, go to Apps > Zodiac
2. add the Delay module (cooldown : 7 days, expiration: 90 days)
3. add the Connext module (Origin sender address : DAO address, Origin domain ID: 6648936 - see [list of Connext domains](https://docs.connext.network/resources/deployments#ethereum))
4. Go to Connext module > write > `setTarget` with the Delay address
5. Get the Connext module address then remove the Connext module
6. Go to Delay > write > `enableModule` with the Connext module address

As code it looks like

```jsx
safe.disableModule(connextMod.address)
safe.enableModule(delayMod.address)
delayMod.enableModule(connextMod.address)
connextMod.setTarget(delayMod.address)
```

Make sure that `delayMod.target` is set to `safe.address` and `connextMod.target` is set to `delayMod.address`

Now the multisig can receive cals from the bridge through the `connextMod` that will pass the call to the `delayMod` which will put it in cooldown.

## Write a multichain DAO proposal

We have a parser for DAO proposals that relies on object formatted as follow :

```jsx
return {
  proposalName, // title of the proposal
  calls: [
    {
      contractAddress: bridgeAddress, // the address of the Connext bridge
      contractNameOrAbi: abiIConnext, // Connext Abi of `xcall`
      functionName: 'xcall', // standard Connext function for bridge call
      functionArgs: [
        destDomainId, // the Domain ID of the receiving chain
        destMultisigAddress, // the safe address on the receiving chain
        ADDRESS_ZERO, // asset
        ADDRESS_ZERO, // delegate
        0, // amount
        30, // slippage
        moduleData, // the calldata to be executed on the receiving chain
      ],
    },
  ],
}
```

Once the proposal is correctly parsed, we can send it to the DAO

```jsx
yarn hardhat gov:submit --gov-address <governor> --proposal <proposal-filepath> --network gnosis <arguments>
```

Here the `<arguments>` passed to the cli as POSIT args are passed to the proposal script. See an example in `proposals/006-cross-bridge-proposal.js`

```jsx
yarn hardhat gov:submit --gov-address 0xE85696a3419162452e6925816D8073374e4190b7 --proposal proposals/006-cross-bridge-proposal.js --network gnosis 137 0xfa2709Aa98F051c4190d70dE38F7c7A330c60ab7 0x2411336105D4451713d23B5156038A48569EcE3a
```

## Executing a call

Once a call has pass the cooldown delay, it can be executed via the `executeNextTx` in the Delay mod.

The arguments to pass to the function are the one that were pass in the original DAO proposal. You can also find here in the `TransactionAdded` event that was fire by the bridge transaction (example [https://polygonscan.com/tx/0xc7bb22753a6c9fccb9c389cd2c3108361f6ed9d3e069134844842d63cdd24ca9#eventlog](https://polygonscan.com/tx/0xc7bb22753a6c9fccb9c389cd2c3108361f6ed9d3e069134844842d63cdd24ca9#eventlog) )

There is a script that allows to execute a call stored in Safe Zodiac Delay module

```jsx
yarn hardhat safe:bridge:execute --bridge-tx <tx hash> --delay-mod <contract address> --network <network name>
```

`bridge-tx` : the tx sent by the bridge to the multisig. This is used to unpack and parse args from the original call

(ex: [https://connextscan.io/tx/0xb24ccc8a014be54acfe764f86c04ca9ec7669a8aaafb38610b1cc029a37379a2](https://connextscan.io/tx/0xb24ccc8a014be54acfe764f86c04ca9ec7669a8aaafb38610b1cc029a37379a2) on the “receive” end)

`delay-mod` : the address of the Zodiac Delay module of the Safe that contains the tx

## Cancel a call

During the cooldown period, a call can be cancelled. That can prevent malicious or errored calls if from being executed.

To cancel a call, you need to call `setTxNonce` from the multisig with the value returned by `queueNonce()`. That way, the nonce will increase past the malicious tx, making the queue appear empty.