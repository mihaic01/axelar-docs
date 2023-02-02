# GMP Express (Beta)

import Callout from 'nextra-theme-docs/callout'

GMP Express is the fastest way to send messages and tokens, building on the foundation of Axelar's General Message Passing (GMP) for [messages](./gmp-messages) and [tokens](./gmp-tokens-with-messages). Full end to end GMP messages can sometimes take many minutes for execution on the destination chain because they must wait for full finality and network resolution, but with GMP Express you can take advantage of faster message and token delivery.

Transactions sent with GMP Express will still pass through the Axelar network, but while this is happening the GMP Express Service will call the target contract on the destination chain. GMP Express will temporarily lend any sent tokens, to be paid back upon resolution by the full network.

By leveraging GMP Express, a theoretical Ethereum to Moonbeam GMP transaction time can be reduced from around 17 minutes down to around 1 minute.

## How to use GMP Express
<Callout>
For a full code example, check out our [examples repository](https://github.com/axelarnetwork/axelar-local-gmp-examples/tree/main/examples/call-contract).
</Callout>

Using GMP Express closely matches the APIs and mental model of [using GMP](./gmp-tokens-with-messages). There are three steps to take to upgrade any normal GMP calls into GMP Express calls.

1. When paying the gas for your message, call `payNativeGasForExpressCallWithToken` instead of `payNativeGasForContractCallWithToken`.
1. On the destination contract that is being called, you’ll inherit from `ExpressExecutable` instead of `AxelarExecutable`
1. You'll need to create a proxy  - (having a standardized proxy that is responsible for repayment, allows us to trivilaly verify express call endpoints) (Squid may not like this)
1. You'll need to register your contract on the [Axelar Services Portal](https://axelar.network) to configure the GMP Express service.

Inheriting from AxelarExpressExecutable instead of AxelarExecutable will be able to handle the express execution and the standard execution from the Axelar Network, including automatic repayment of the GMP Express loan.

<!-- TODO Add guidance for running your own Express service, or let users express their own calls 

eg
gasReceiver.payNativeGasForContractCallWithToken{ value: msg.value }( …);

And then expressExecuteWithToken on the destination chain with your own microservice or use some other third party Express service instead. Users also could express execute their transactions as of permissionless nature of the protocol.
-->




## Behind the scenes
Behind the scenes GMP Express messages follow two paths in parallel.

* The Axelar GMP Express service will pick up transactions and execute them on the destination chain, supplying any sent tokens as a loan. This bypasses the standard Relayer for initial delivery. 

* The transaction will be picked up by Axelar Network as usual and sent to the relayer on the destination chain as usual. Funds will be minted to your contract, and automatically used to repay the GMP Express loan.

--- *Graphic Designer insert visual flow here* ---

### Standard GMP Process
For usual GMP calls from an `AxelarExecutable`, we have multiple steps that take some time:
1. `NativeGasPaidForContractCallWithToken` event is emitted by the GasService on the source chain
1. `ContractCallWithToken` event is emitted by the Gateway on the source chain
1. After required confirmation height, event is detected by Axelar sentinels and delivered to Axelar network: 
1. Event confirmed by validators and voted on. 
1. A unique command ID is generated for the transaction
1. Command IDs are batched and then broadcasted to the destination chain
1. GMP approval command is sent to the gateway on the destination chain.
1. Destination contract `executeWithToken` function is called and the execution is validated with the gateway.

### GMP Express Process
For the faster GMP Express call we skip some validation steps to speed things up:
1. `NativeGasPaidForExpressContractCallWithToken` event is emitted by the GasService on the source chain.
1. `ContractCallWithToken` event is emitted by the Gateway on the source chain.
1. The GMP Express service will detect the event and verify several conditions:
    1. The microservice knows to invoke `expressExecuteWithToken` before  `executeWithToken` because the right method is called at the gas service and `NativeGasPaidForExpressContractCallWithToken` was emitted (instead of `NativeGasPaidForContractCallWithToken`) .
    1. The amount is below a configured maximum (ex $1,000).
    1. The amount is under of what currently available at AxelarGMPExpressService (all funds might be already used by other executions).
    1. The contract was configured in the Axelar Service Portal.
1. After a configured minimum amount of confirmations on the source chain our microservice initiates the execution. 
    1. The microservice will call the `AxelarGMPExpressService` contract on the destination chain that will call the dApp’s destination contract (which must implement an `AxelarExpressExecutable` interface). 
    1. AxelarGMPExpressService will lend the necessary token to the `AxelarExpressExecutable` contract and will call `executeWithToken` method.
1. After some time, the Axelar network will finish processing the transaction and GMP approval will arrives at the destination gateway.
    1. `executeWithToken` is called by the standarard relayer but no action is performed as execution is already complete.
    1. The lent token is returned back to the `AxelarGMPExpressService` automatically by code in your `AxelarExpressExecutable` contract.

## Cost
GMP Express calls will cost more than standard GMP calls because they will still follow the full GMP path in addition to the express execution. While heavily dependent on the involved chains, your contract complexity and payload size, we expect express messages to generally cost XX% more than standard calls.

## Security
Because GMP Express will be unable to validate the message using the Axelar network, you should verify that incoming execution requests are originated by the Axelar GMP Express service.

```solidity
code code code
```

## Status
The GMP Express service is currently in beta and only supported on these chains:

1. a
1. b
1. c