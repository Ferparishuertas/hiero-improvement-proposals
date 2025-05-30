---
hip: 18
title: Custom Hedera Token Service Fees
author: Cooper Kunz <cooper@calaxy.com>, Rahul Kothari <rahul.kothari.201@gmail.com>
type: Standards Track
category: Service
needs-council-approval: Yes
status: Final
release: v0.16.0
superseded-by: 573
created: 2021-04-30
discussions-to: https://github.com/hiero-ledger/hiero-improvement-proposals/discussions/92
updated: 2021-10-27
---
## Abstract

We propose adding a limited set of functionality to the Hedera Token Service (HTS) that allows developers to define
custom fees between different tokens at time of creation, or at a later modification date with Hedera’s controlled
mutability ([1](https://hedera.com/blog/code-is-law-but-what-if-the-law-needs-to-change)). These custom token fees will
enable dynamic ecosystem fees as we see with projects such as
Uniswap ([2](https://uniswap.org/docs/v2/protocol-overview/how-uniswap-works/)), or even Hedera
itself ([3](https://hedera.com/hh_whitepaper_v2.1-20200815.pdf)). Additional implementations could even allow
implementation of custom, and if desired, perpetual
royalties ([4](https://messari.io/article/explain-it-like-i-am-5-nfts)), for any token issued on HTS, whether fungible
or non-fungible.

## Motivation

In order to build a robust and sufficiently decentralized ecosystem, where no single entity can control community fees,
or otherwise, developers need to be able to programmatically define and control fee relationships between various tokens
issued on Hedera. Currently, the Hedera Token Service allows developers to easily configure, issue, and administer
various types of cryptocurrencies, or tokens ([5](https://hedera.com/hh_tokenization-whitepaper_v2_20210101.pdf)).
However, there is currently very limited programmability, and HTS is not supported within Hedera Smart
Contracts ([6](https://docs.hedera.com/guides/docs/sdks/tokens/define-a-token)), nor implemented within a layer 2
application/business network running on the Hedera Consensus
Service ([7](https://github.com/hiero-ledger/hiero-improvement-proposals/issues/33)).

In practice, this may look something like creating a “protocol” or “ecosystem” token for a given service. The ecosystem
may have dozens, or potentially hundreds of “related” or “sub” tokens, generated by different entities. By defining a
custom token relationship (i.e. a token fee schedule) for each of the related tokens at the time of its creation, we can
guarantee that the protocol or ecosystem token is able to capture value and distribute it trustlessly to various
ecosystem participants. Without this functionality there is no true “on-chain”, or in Hedera’s case, “on ledger”, way to
ensure that all future transactions of a given token are associated with a protocol or native token, or similarly
royalties for an NFT are going to the appropriate party.

To provide a concrete example, there may be a protocol token called $PROTOCOL_GAS, which is required to be held by
everyone participating in an ecosystem. Any tokens created within this ecosystem can have various amounts of
$PROTOCOL_GAS, attached to them. So in order to send {protocol-token-1} to a user, they could, for example, attach 1
$PROTOCOL_GAS token to send to {entity-1}, 2 $PROTOCOL_GAS tokens to {entity-2}, and 3 $PROTOCOL_GAS tokens to
{entity-3}, then each entity, e.g. a DAO, can do whatever they wish with the fees accumulated.

## Rationale

We believe and emphasize that this needs to be done natively on Hedera’s ledger via the Token Service, or some type of
scripting language, and not at a layer 2 or via an application network to ensure that it has the same immutability
characteristics and deterministic functionality required for things like 3rd party cryptocurrency exchange listings and
3rd party wallet support. While there may be some coordinated cryptographic workarounds to achieve this with
multisignature and/or multiparty transfers and integration with Hedera’s newly released scheduled
transactions ([8](https://docs.hedera.com/guides/core-concepts/scheduled-transaction)), this approach is a notably
better user experience for both developers and end-users.

Currently each transaction on Hedera requires a transaction
fee ([9](https://docs.hedera.com/guides/docs/hedera-api/basic-types/feedata)). This includes a list of accounts that are
part of each transaction. By enabling the ability for new HTS based tokens to optionally add a few custom entities and 
associated payments to this list dramatically expand the functionality and programmability available natively on Hedera.

## Specification

At the conceptual level, we propose adding the following:
```
CustomFee Class:
    - one of
        - percentageOfFee percent out of the total amount every time a transfer of units of this token is executed
        - fixedFee fixed amount of some token
    - AccountID feeCollector

HederaToken:
    - CustomFee[] customFees
    - getCustomFees()

HederaTokenTransfer(tokenId, amount, toAddress):
    customFees = tokenId.getCustomFees()
    remainingPercent = 1

    for customFee in customFees:
        if customFee is percentageOfFee
            tokenTransfer(tokenId, amount*percentageOfFee.percent, customFee.feeCollector)
            remainingPercent -= percentageOfFee.percent
        else customFee is fixedFee
            tokenTransfer(fixedFee.token, fixedFee.amount, customFee.feeCollector)

	tokenTransfer(tokenId, amount*remainingPercent, toAddress)

TokenUpdate transactions:
    - updateCustomFees()
    - updateFeeScheduleKey()

TokenInfo:
    - call getCustomFees()
```

Note: see the reference implementation section for more details.

## HAPI Changes

At the gRPC Protocol level we propose adding 6 new messages and modifies 6 existing messages in a forward compatible way:

### Fraction
Adds message `Fraction` representing a fraction of the amount of a transfer to collect as a custom fee.
```protobuf
message Fraction {
  // The fraction's numerator
  int64 numerator = 1;   
  // The fractions's denominator
  int64 denominator = 2; 
}
```

### FractionalFee
Adds message `FractionalFee` representing a fee type that is a fraction of the transferred units of a token to assess 
as a fee in the denomination of units of the token to which this fractional fee is attached.
```protobuf
message FractionalFee {
  // The fraction of the transferred units to assess as a fee
  Fraction fractional_amount = 1; 
  // The minimum amount to assess
  int64 minimum_amount = 2;
  // The maximum amount to assess (zero implies no maximum)
  int64 maximum_amount = 3;
}
```

### FixedFee
Adds message `FixedFee` representing a fee type having a fixed number of units (hBar or token) charged when the 
transaction transferring the token is executed.
```protobuf
message FixedFee {
  // The number of units to assess as a fee
  int64 amount = 1;
  // The denomination of the fee; taken as hbar if left unset
  TokenID denominating_token_id = 2;
}
```

### RoyaltyFee
A fee to assess during a CryptoTransfer that changes ownership of an NFT. Defines
the fraction of the fungible value exchanged for an NFT that the ledger should collect
as a royalty. ("Fungible value" includes both ℏ and units of fungible HTS tokens.) When
the NFT sender does not receive any fungible value, the ledger will assess the fallback
fee, if present, to the new NFT owner. Royalty fees can only be added to tokens of type
type NON_FUNGIBLE_UNIQUE.

```protobuf
message RoyaltyFee {
  // The fraction of fungible value exchanged for an NFT to collect as royalty
  Fraction exchange_value_fraction = 1;
  // If present, the fixed fee to assess to the NFT receiver when no fungible value is exchanged with the sender
  FixedFee fallback_fee = 2; 
}
```

### CustomFee
Adds message `CustomFee` defining type of fee and account receiving the fee assessed during a CryptoTransfer of the 
associated token. A custom fee may be either fixed or fractional and must specify a fee collector account to receive 
the assessed fees.
```protobuf
message CustomFee {
  oneof fee {
    // Fixed fee to be charged
    FixedFee fixed_fee = 1;
    // Fractional fee to be charged
    FractionalFee fractional_fee = 2;
    // Royalty fee to be charged
    RoyaltyFee royalty_fee = 4; 
  }
  // The account to receive the custom fee
  AccountID fee_collector_account_id = 3; 
}
```

### AssessedCustomFee
Adds message `AssessedCustomFee` representing a fee that was assessed during processing of a CryptoTransfer.
```protobuf
message AssessedCustomFee {
  // The number of units assessed for the fee
  int64 amount = 1; 
  // The denomination of the fee; taken as hbar if left unset
  TokenID token_id = 2;
  // The account to receive the assessed fee
  AccountID fee_collector_account_id = 3;
  // The sender or receiver account(s) that were charged the custom fees
  repeated AccountID effective_payer_account_id = 4; 
}
```

### TokenFeeScheduleUpdateTransactionBody
Adds message `TokenFeeScheduleUpdateTransactionBody` representing a request made to the network to update custom fees for a token.
```protobuf
message TokenFeeScheduleUpdateTransactionBody {
  // The token whose fee schedule is to be updated
  TokenID token_id = 1;
  // The new custom fees to be assessed during a 
  // CryptoTransfer that transfers units of this token
  repeated CustomFee custom_fees = 2;
}
```

### TransactionBody
Updates message `TransactionBody` adding the transaction body for updating custom fees for a given token.
```protobuf
message TransactionBody {
  ...
  oneof data {
    ...
    // Updates a token's custom fee schedule
    TokenFeeScheduleUpdateTransactionBody token_fee_schedule_update = 45;
    ...
  }
}
```

### TokenService
Updates message `TokenService` to include the method updating a custom fee schedule for a token.
```protobuf
service TokenService {
    ...
    // Updates the custom fee schedule on a token
    rpc updateTokenFeeSchedule (Transaction) returns (TransactionResponse);
    ...
}
```

### TokenCreateTransactionBody
Updates message `TokenCreateTransactionBody` by adding a `fee_schudle_key` property identifying the key that must sign 
transactions updating the list of custom fees and `custom_fees` for the list of fixed and fractional custom fees
associated with the created token.
```protobuf
message TokenCreateTransactionBody {
    ...
    // The key which can change the token's custom fee schedule; 
    // must sign a TokenFeeScheduleUpdate transaction. If not specified
    // the custom fee schedule cannot be changed after creation.
    Key fee_schedule_key = 20;
    // The custom fees to be assessed during a
    // CryptoTransfer that transfers units of this token
    repeated CustomFee custom_fees = 21; 
}
```

### TokenInfo
Updates message `TokenInfo` returned from token information queries to include the administrative key for updating
custom fee schedules and the current list of custom fixed and fractional fees associated with the token.
```protobuf
message TokenInfo {
    ...
    // The key which can change the custom fee 
    // schedule of the token; if not set, the fee
    // schedule is immutable.
    Key fee_schedule_key = 22;
    // The custom fees to be assessed during a 
    // CryptoTransfer that transfers units of this token
    repeated CustomFee custom_fees = 23;
}
```

### TokenUpdateTransactionBody
Updates message `TokenUpdateTransactionBody` to include the administrative key for updating custom fees as an optional entry.
```protobuf
message TokenUpdateTransactionBody {
    ...
    // If set, the new key to use to update the token's 
    // custom fee schedule; if the token does not 
    // currently have this key, transaction will resolve 
    // to TOKEN_HAS_NO_FEE_SCHEDULE_KEY
    Key fee_schedule_key = 14; 
}
```

### TransactionRecord
Updates message `TransactionRecord` to include the list of custom fees assessed as a part of executing the transaction
represented by this record.
```protobuf
message TransactionRecord {
    ...
    // All custom fees that were assessed during 
    // a CryptoTransfer, and must be paid if the 
    // transaction status resolved to SUCCESS
    repeated AssessedCustomFee assessed_custom_fees = 13;
}
```

## Backwards Compatibility

There are no known backwards compatibility issues. All tokens defined on HTS at the time of this implementation could
retain their current transaction record structure and be processed by the network, mirror nodes, etc. as if they were
tokens created without custom token fee schedules.

For future compatibility with this issue there are changes that will be required to the Hedera Mirror Nodes, on and off
ramps, SDKs, wallets, and applications that are looking to support HTS assets with custom token fees.

## Security Implications

This shouldn’t necessarily change security implications not already known by Hedera’s current fee model and transaction
record structure/implementation.

However, there is a potential additional DDoS vector which can and should be addressed with limitations to the number of
custom fees per token issued on HTS, as well as additional HBAR related fees for token transactions requiring a custom
defined token fee schedule, and associated more computationally expensive operations.

## How to teach this

The most easily accessible way to explain this to users and developers in the ecosystem is that “just like HBAR is a
required transaction fee to use the Hedera network, other tokens can be required to be used for additional custom fees
in relation to other tokens.”

So similarly to how users might currently experience an insufficient transaction fee, like “insufficient transaction
fee: 0.05” which we can deduce is related to HBAR, after the implementation of this HIP they may need to note
“insufficient transaction fee: 0.05 {HBAR, or token name}” depending on the configuration of the token being
transferred.

## Reference implementation

Current transaction record:
```json
{
  "signatures": [],
  "transactionID": {
    "accountID": {
      "num": 40938,
      "shardNum": 0,
      "realmNum": 0
    },
    "validStartDate": "2021-05-04T14:08:51.927014828Z"
  },
  "nodeAccountID": {
    "num": 11,
    "shardNum": 0,
    "realmNum": 0
  },
  "transactionFee": 1000000000,
  "transactionValidDurationInSec": 120,
  "generateRecord": false,
  "memo": "",
  "fileName": "recordstreams/record0.0.6/2021-05-04T14_09_00.011224000Z.rcd",
  "index": 388,
  "record": {
    "transactionHash": "acacc3ceb6f71171f80cc5fefbdb2b6355a1fe90e06194091851523f996e8b0ae1b35ad586440105e0aecaeb8bcab681",
    "consensusTimeStamp": "2021-05-04T14:09:01.674103Z",
    "transactionFee": 310019,
    "transactionID": {
      "accountID": {
        "num": 40938,
        "shardNum": 0,
        "realmNum": 0
      },
      "validStartDate": "2021-05-04T14:08:51.927014828Z"
    },
    "memo": "",
    "transfers": [
      {
        "accountID": {
          "num": 11,
          "shardNum": 0,
          "realmNum": 0
        },
        "amount": 21801
      },
      {
        "accountID": {
          "num": 98,
          "shardNum": 0,
          "realmNum": 0
        },
        "amount": 288218
      },
      {
        "accountID": {
          "num": 40938,
          "shardNum": 0,
          "realmNum": 0
        },
        "amount": -310019
      }
    ],
    "tokenTransfers": [
      {
        "tokenID": {
          "num": 127877,
          "shardNum": 0,
          "realmNum": 0
        },
        "accountAmount": [
          {
            "accountID": {
              "num": 40938,
              "shardNum": 0,
              "realmNum": 0
            },
            "amount": -10000000000
          },
          {
            "accountID": {
              "num": 54429,
              "shardNum": 0,
              "realmNum": 0
            },
            "amount": 10000000000
          }
        ]
      }
    ],
    "receipt": {
      "responseCode": "SUCCESS",
      "currentExchangeRate": {
        "hBarEquiv": 30000,
        "centEquiv": 923617,
        "expirationTime": "2021-05-04T15:00:00Z"
      },
      "nextExchangeRate": {
        "hBarEquiv": 30000,
        "centEquiv": 920642,
        "expirationTime": "2021-05-04T16:00:00Z"
      }
    }
  },
  "transfers": [],
  "transactionType": "CRYPTO_TRANSFER"
}
```

Example of a new transaction record:
```json
{
  "signatures": [],
  "transactionID": {
    "accountID": {
      "num": 40938,
      "shardNum": 0,
      "realmNum": 0
    },
    "validStartDate": "2021-05-04T14:08:51.927014828Z"
  },
  "nodeAccountID": {
    "num": 11,
    "shardNum": 0,
    "realmNum": 0
  },
  "transactionFee": 1000000000,
  "assessedCustomFees": [
    {
      "amount": 10000000000,
      "tokenId": {
        "num": 127877,
        "shardNum": 0,
        "realmNum": 0
      },
      "feeCollectorAccountId": {
        "num": 54429,
        "shardNum": 0,
        "realmNum": 0
      }
    }
  ],
  "transactionValidDurationInSec": 120,
  "generateRecord": false,
  "memo": "",
  "fileName": "recordstreams/record0.0.6/2021-05-04T14_09_00.011224000Z.rcd",
  "index": 388,
  "record": {
    "transactionHash": "acacc3ceb6f71171f80cc5fefbdb2b6355a1fe90e06194091851523f996e8b0ae1b35ad586440105e0aecaeb8bcab681",
    "consensusTimeStamp": "2021-05-04T14:09:01.674103Z",
    "transactionFee": 310019,
    "transactionID": {
      "accountID": {
        "num": 40938,
        "shardNum": 0,
        "realmNum": 0
      },
      "validStartDate": "2021-05-04T14:08:51.927014828Z"
    },
    "memo": "",
    "transfers": [
      {
        "accountID": {
          "num": 11,
          "shardNum": 0,
          "realmNum": 0
        },
        "amount": 21801
      },
      {
        "accountID": {
          "num": 98,
          "shardNum": 0,
          "realmNum": 0
        },
        "amount": 288218
      },
      {
        "accountID": {
          "num": 40938,
          "shardNum": 0,
          "realmNum": 0
        },
        "amount": -310019
      }
    ],
    "tokenTransfers": [
      {
        "tokenID": {
          "num": 127877,
          "shardNum": 0,
          "realmNum": 0
        },
        "accountAmount": [
          {
            "accountID": {
              "num": 40938,
              "shardNum": 0,
              "realmNum": 0
            },
            "amount": -10000000000
          },
          {
            "accountID": {
              "num": 54429,
              "shardNum": 0,
              "realmNum": 0
            },
            "amount": 10000000000
          }
        ]
      }
    ],
    "receipt": {
      "responseCode": "SUCCESS",
      "currentExchangeRate": {
        "hBarEquiv": 30000,
        "centEquiv": 923617,
        "expirationTime": "2021-05-04T15:00:00Z"
      },
      "nextExchangeRate": {
        "hBarEquiv": 30000,
        "centEquiv": 920642,
        "expirationTime": "2021-05-04T16:00:00Z"
      }
    }
  },
  "transfers": [],
  "transactionType": "CRYPTO_TRANSFER"
}
```

## Additional examples
```
new TokenCreateTransaction()
    .setName(..)
    .addCustomFee({token-id}, {value}, {account-id})
    .addCustomFee(0.0.987, 100, 0.0.123)
    ...
    .execute(client)
```

In this example, the token created would require 100 tokens with entity ID 0.0.987 to be transferred into the account
with entity ID 0.0.123. Account 0.0.123 could obviously be a single individual’s account, or it could be a
multisignature account managed by an HCS based consensus node network, as we’ve recently seen in Greg Scullard’s NFT auction
demo ([10](https://www.youtube.com/watch?v=hCPXKR1e7Ro)).

This could be expanded to include up to 10 custom fees per token.
```
new TokenCreateTransaction()
    .setName(..)
    .addCustomFee(0.0.987, 100, 0.0.123)
    .addCustomFee(0.0.654, 200, 0.0.456)
    .addCustomFee(HBAR, 10, 0.0.753)
    ...
    .execute(client)
```

An obvious example for this is a multiparty ecosystem, with a community managed treasury, a foundation treasury, and the
genesis application. Or in the meta-example, Hedera having distinct buckets for each the node, service, and network fees
respectively which could trustlessly be managed by different entities, like a DAO.

We also propose adding the following percentage based variation of the base implementation. Note that the token in the
percentage will be the same as the token to be created or updated, hence omitted in the transaction.
```
new TokenCreateTransaction()
    .setName(..)
    .addCustomFeePercentage({percent}, {account-id})
    .addCustomFeePercentage(10%, 0.0.123)
    ...
    .execute(client)
```

Typically, percentage royalties will be deducted from the tokens being transferred. If there are no tokens being
transferred, then when tokens are created there can be a “minimum” or fallback price denominated in any token. In the
example below 10 tokens would be required to be sent to account 0.0.123 in the case there was a 0 value transaction, or
anything that would result in a royalty percentage fee less than the minimum.
```
new TokenCreateTransaction()
    .setName(..)
    .addCustomFeePercentage(10%, 0.0.123)
        .setMinimumAmount(10)
    ...
    .execute(client)
```

If the token relationship or fee ID is defined as empty, or null, then the custom fee is applied to the token that is
being created (because there isn’t yet an entity ID by which we can refer to it). If there are not enough tokens, or
those tokens are not sufficiently divisible, then an error for “insufficient custom token transaction fee” (or something
similar) would be thrown. It would be on the user transacting to go out and accumulate enough tokens in their wallet to
successfully execute the transaction with the defined custom token fee schedules.

```
new TokenCreateTransaction()
    .setName(..)
    .addCustomFee(null, 100, 0.0.123)
    ...
    .execute(client)
```

## Use cases 
Currently on Hedera, there is no decentralized and trustless mechanism of distributing the value that is captured when
a token transfer occurs. At least, not natively at Layer 1. If issuing an NFT, for a single example, a user could simply
transact that token peer-to-peer via a wallet that supports HTS and circumvent any potential fees that are implemented
at the application layer, removing the trustless value proposition that new asset classes like NFTs enable for their
creators. This prposal allows this use case to be easily addressed, among a large variety of others.

Other general/categorical examples include:

- Multiparty / multi stakeholder ecosystem treasuries
- Liquidity provisioning 
- Automated community incentives 
- Native protocol tokens being built upon Hedera, e.g. those used in many 3rd party apps
- Trustless perpetual royalties, e.g. NFTs

## References
- 1 - https://hedera.com/blog/code-is-law-but-what-if-the-law-needs-to-change
- 2 - https://uniswap.org/docs/v2/protocol-overview/how-uniswap-works/
- 3 - https://hedera.com/hh_whitepaper_v2.1-20200815.pdf
- 4 - https://messari.io/article/explain-it-like-i-am-5-nfts
- 5 - https://hedera.com/hh_tokenization-whitepaper_v2_20210101.pdf
- 6 - https://docs.hedera.com/guides/docs/sdks/tokens/define-a-token
- 7 - https://github.com/hiero-ledger/hiero-improvement-proposals/issues/33
- 8 - https://docs.hedera.com/guides/core-concepts/scheduled-transaction
- 9 - https://docs.hedera.com/guides/docs/hedera-api/basic-types/feedata
- 10 - https://www.youtube.com/watch?v=hCPXKR1e7Ro

## Copyright
This document is licensed under the Apache License, Version 2.0 -- see [LICENSE](../LICENSE)
or (https://www.apache.org/licenses/LICENSE-2.0)
