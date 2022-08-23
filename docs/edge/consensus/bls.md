---
id: bls
title: BLS
description: "hoge"
keywords:
  - docs
  - polygon
  - edge
  - bls
---

## Overview

BLS is a signing algorithm that can aggregate multiple signatures into a single bytes array signature. In Polygon Edge, BLS can be used to generate `CommittedSeals` and `ParentCommittedSeals` in IBFT Extra of Header. As default, these data are array of signatures signed by ECDSA. BLS can aggregate these data into one bytes array and reduce block header size. Each chain can choose whether to use BLS or not. ECDSA key is used regardless of whether BLS mode or not because `ProposerSeal` which is signed by Block Proposer is signed by ECDSA in the both mode.

All validators must sign `CommittedSeal` in BLS mode. On the other hand, all validators must sign `CommittedSeal` by ECDSA in the chain that doesn't use BLS.

## How to setup new chain using BLS

This section describes how to setup a new chain using BLS step by step.
Refer the [Local Setup](/docs/edge/get-started/set-up-ibft-locally)
/ [Cloud Setup](/docs/edge/get-started/set-up-ibft-on-the-cloud) sections for detailed setup

The **only difference** between setting up ECDSA mode and BLS mode is to generate keys and `genesis.json`.

### 1. Create keys

**When creating new keys for the chain of BLS mode, an additional flag is needed `--bls`**:

`secrets init` command prints the BLS Public Key in addition to Validator Key (Address) and Node ID if `--bls` flag is given.

```bash
polygon-edge secrets init --bls --data-dir test-chain-1

[SECRETS INIT]
Public key (address) = 0x...
BLS Public key       = 0x...
Node ID              = 16...
```

### 2. Generate genesis.json

**When creating genesis.json for the chain using BLS, an additional flag is needed `--ibft-validator-type`**

`--ibft-validators-prefix-path` can be used to specify validators if the data directories that contains validator keys are located locally.

```bash
polygon-edge genesis --consensus ibft --ibft-validator-type bls --ibft-validators-prefix-path test-chain- --bootnode /ip4/127.0.0.1/tcp/10001/p2p/[NODE_ID_1] 
```

If the data directories are not in locally, you can specify validator address and BLS Public Key for `--ibft-validator` flag. The format of the value is `[ADDRESS_HEX]:[BLS_PUBLIC_KEY_HEX]` whose values are displayed in `secrets init` command's result.

```bash
polygon-edge genesis --consensus ibft --ibft-validator-type bls --ibft-validator [VALIDATOR_ADDRESS]:[VALIDATOR_BLS_PUBLIC_KEY] test-chain- --bootnode /ip4/127.0.0.1/tcp/10001/p2p/[NODE_ID]
```

### 3. Start server

Once `genesis.json` is created, you can start the server as usual.

```bash
polygon-edge server --data-dir ./test-chain-1 --chain genesis.json --grpc-address :10000 --libp2p :10001 --jsonrpc :10002 --seal
```

## How to migrate from an existing PoA chain

This section describes how to use BLS mode in existing PoA chain.
the following steps are required in order to enable BLS in PoA chain.

1. Generate BLS keys for validators
2. Add fork settings into genesis.json

### 1. Generate BLS key

As mentioned above, `secrets init` with `--bls` flag can generate BLS key. In order to keep existing ECDSA and Network key and add a new BLS key, `--ecdsa` and `--network` needs to be disabled.

```bash
polygon-edge secrets init --bls --ecdsa=false --network=false

[SECRETS INIT]
Public key (address) = 0x...
BLS Public key       = 0x...
Node ID              = 16...
```

### 2. Add fork settings

`ibft switch` command adds fork setting, which enables BLS in the middle, into `genesis.json`.

For PoA chain, validators need to be given in the command. As with the way of `genesis` command, `--ibft-validators-prefix-path` or `--ibft-validator` flags can be used to specify validator.

Specify the height from which the chain starts BLS mode for `--from`.

```bash
polygon-edge ibft switch --chain ./genesis.json --type PoA --ibft-validator-type bls --ibft-validators-prefix-path test-chain- --from 100
```

## How to migrate from an existing PoS chain

This section describes how to use BLS mode in existing PoS chain.
The following steps are required in order to enable BLS in PoS chain.

1. Generate BLS keys for validators
2. Add fork settings into genesis.json
3. Call the staking contract to register BLS Public Key

### 1. Generate BLS key

As mentioned above, `secrets init` with `--bls` flag can generate BLS key. In order to keep existing ECDSA and Network key and add a new BLS key, `--ecdsa` and `--network` needs to be disabled.

```bash
polygon-edge secrets init --bls --ecdsa=false --network=false

[SECRETS INIT]
Public key (address) = 0x...
BLS Public key       = 0x...
Node ID              = 16...
```

### 2. Add fork settings

`ibft switch` command adds fork setting, which enables BLS in the middle, into `genesis.json`.

In the PoS chain, `--ibft-validators-prefix-path` or `--ibft-validator` are not used because validator info (address and BLS Public Key) is stored in the staking contract.

Specify the height from which the chain starts BLS mode for `--from`, and the height at which the contract is updated for `--deployment`.

```bash
polygon-edge ibft switch --chain ./genesis.json --type PoS --ibft-validator-type bls --deployment 50 --from 200
```

### 3. Register BLS Public Key in staking contract

After fork is added and validators are restarted, each validator needs to call staking contract to register BLS Public Key. This must be done after the height specified in `--deployment` before the height specified in `--from`.

The script to register BLS Public Key is defined in [Staking Smart Contract repo](https://github.com/0xPolygon/staking-contracts). 

Set `BLS_PUBLIC_KEY` to be registered into `.env` file. Refer [pos-stake-unstake](/docs/edge/consensus/pos-stake-unstake#setting-up-the-provided-helper-scripts) for more details about other parameters.

```env
JSONRPC_URL=http://localhost:10002
STAKING_CONTRACT_ADDRESS=0x0000000000000000000000000000000000001001
PRIVATE_KEYS=0x...
BLS_PUBLIC_KEY=0x...
```

The following command register BLS Public Key given in `.env` to the contract.

```bash
npm run register-blskey
```

:::warning Validators need to register BLS Public Key manually
In BLS mode, validators must have their own address and BLS public key. The consensus layer ignores the validators that have not registered BLS public key in the contract when the consensus fetches validators info from the contract.
:::