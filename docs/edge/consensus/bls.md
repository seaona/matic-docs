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

## How to create new chain using BLS

Setting up a network with the Polygon Edge is covered in
the [Local Setup](/docs/edge/get-started/set-up-ibft-locally)
/ [Cloud Setup](/docs/edge/get-started/set-up-ibft-on-the-cloud) sections.

The **only difference** between setting up ECDSA mode and BLS mode is to generate keys and `genesis.json`.

**When creating new keys for the chain of BLS mode, an additional flag is needed `--bls`**:

```bash
polygon-edge secrets init --bls ...
```

**Also, When creating genesis.json for the chain using BLS, an additional flag is needed `--ibft-validator-type`**

```bash
polygon-edge genesis --ibft-validator-type bls ...
```

## How to migrate from an existing chain to the chain using BLS

In order to enable BLS in an existing chain, the following steps are required.

1. Generate BLS keys for validators
2. Add fork settings into genesis.json
3. register BLS Public Key in staking contract (Only in PoS)

### Generate BLS key

As mentioned above, `secrets init` with `--bls` flag can generate BLS key. In order to keep existing ECDSA and Network key and add a new BLS key, `--ecdsa` and `--network` needs to be disabled.

```bash
polygon-edge secrets init --bls --ecdsa=false --network=false ...
```

### Add fork settings

`ibft switch` command adds fork setting, which enables BLS in the middle, into `genesis.json`.

For PoA chain, validators need to be given in the command. As with the way of `genesis` command, `--ibft-validators-prefix-path` or `--ibft-validator` flags can be used to specify validator. When using `--ibft-validator`, the value must follow the format, `[ADDRESS]:[BLS_PUBLIC_KEY]`. Address and BLS Public Keys are shown on executing `secrets init` command.

`--from` specifies the height from which the chain starts BLS mode.

```bash
polygon-edge ibft switch --chain ./genesis.json --type PoA --ibft-validator-type bls --ibft-validators-prefix-path test-chain- --from 100
```

In the PoS chain, `--ibft-validators-prefix-path` or `--ibft-validator` are not used because each validator must call staking contract function to register BLS Public Key in the contract after the contract is updated.

`--deployment` specifies the height from which the contract is updated at and `--from` specifies the height from which the chain starts BLS mode.

```bash
polygon-edge ibft switch --chain ./genesis.json --type PoS --ibft-validator-type bls --deployment 50 --from 200
```

### Register BLS Public Key in staking contract (Only in PoS)

After fork is added and validators are restarted, each validator needs to call staking contract to register BLS Public Key. This must be done after the height specified in `--deployment` before the height specified in `--from`.

The script to register BLS Public Key is defined in [Staking Smart Contract repo](https://github.com/0xPolygon/staking-contracts). 

Set `BLS_PUBLIC_KEY` to be registered into `.env` file. Refer [pos-stake-unstake](/docs/edge/get-started/pos-stake-unstake#setting-up-the-provided-helper-scripts) for more details about other parameters.

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