## Notice: in development

This section will be removed when the public interface has stabilized.

# Dealer JSONRPC v1.0

Definitions and specification for the Dealer [JSONRPC (2.0)](https://www.jsonrpc.org/specification) API. This document specifies the first public Dealer JSONRPC version, `v1.0`.

Methods under the `dealer` namespace (with the `dealer_` prefix) comprise the public dealer API. Implementations MAY provide private administrative functionality through methods under a distinct namespace.

The Dealer JSONRPC can be served over WebSockets, HTTP POST, or HTTP GET, or other transports at the discretion of implementers (see notes).

Be sure to see important notes and resources in [the appendix.](#appendix)

### Conventions

-   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/rfc/rfc2119.txt).
-   The word "implementation" is used to refer to any system that implements or provides the Dealer JSONRPC.
-   In accordance with the JSONRPC 2.0 spec, JSON types (String, Number, Object, and Array) are capitalized to differentiate between usage of the word and the JSON type.
-   The syntax "Array\<T>" is used to indicate an Array of type T (for JSON values) or an Array of schema T.

## Contents

-   [Requirements](#requirements)
-   [Encoding](#encoding)
-   [Quotes](#quotes)
-   [Pagination](#pagination)
-   [Errors](#errors)
-   [Schemas](#schemas)
    -   [Ticker](#schema-ticker)
    -   [Time](#schema-time)
    -   [UUID](#schema-uuid)
    -   [TradeInfo](#schema-tradeinfo)
    -   [QuoteInfo](#schema-quoteinfo)
    -   [Asset](#schema-asset)
    -   [Market](#schema-market)
    -   [Quote](#schema-quote)
    -   [Trade](#schema-trade)
-   [Methods](#methods)
    -   [AuthStatus](#method-dealer_authstatus)
    -   [GetAssets](#method-dealer_getassets)
    -   [GetMarkets](#method-dealer_getmarkets)
    -   [GetPastTrades](#method-dealer_getpasttrades)
    -   [GetQuote](#method-dealer_getquote)
    -   [SubmitFill](#method-dealer_submitfill)
    -   [Time](#method-dealer_time)
-   [Appendix](#appendix)
    -   [Notes](#notes)
    -   [Important resources](#important-resources)
    -   [Error codes](#error-codes)

## Requirements

In addition to notices in each section, each of the following must be true in order for an implementation to be considered in adherence with the specification.

These requirements are intended to motivate strong guarantees of compatibility between clients and servers and ensure maximum levels of safety for the operators of each: traders and dealers that implement this API.

-   Implementations MUST implement all methods under the `dealer` namespace (see [Methods](#methods)).
-   Implementations MUST implement all public object schematics (see [Schemas](#schemas)).
-   Implementations MUST use the canonical 0x v3 addresses for the active Ethereum network (not yet deployed).
-   Implementations MUST support asset settlement according to relevant sections in this document and [ZEIP-18](https://github.com/0xProject/ZEIPs/blob/master/ZEIPS/ZEIP-18.md).
-   Implementations MUST only support ERC-20 assets (subject to change in future major API versions).
-   All supported assets MUST each have a unique string identifier called a "ticker" (e.g. DAI, ZRX, WETH).
-   Implementations MUST use arbitrary precision (or sufficiently precise fixed-precision) representations for integers.
-   Implementations MUST NOT use floating points in the public API, except where denoting units of time.
-   Implementations MAY support batch requests, in accordance with the JSONRPC 2.0 specification.
-   Implementations SHOULD support Ether (ETH) trading, and if so, MUST do so via the canonical [WETH contract](https://etherscan.io/address/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2) for the active network.
-   Implementations MAY require that quote requests include the potential taker's address.
    -   The address provided by the taker MAY be used to restrict the `takerAddress` of the quotes underlying signed 0x order.
    -   Implementations MAY record and use the address provided by the taker to influence pricing or to restrict quote provision for blacklisted takers.
-   Implementations MAY use Arrays or Objects for return values and parameters (in accordance with the JSONRPC specification).
    -   If arrays are used, the index specified in each method MUST match the implementation.
    -   If objects are used, the keys MUST match the name specified for each parameter.

## Encoding

-   All asset amount values MUST be integer values (as Numbers) in their base units.
-   Binary data (EVM `bytes`, etc.) MUST be encoded as `0x`-prefixed all-lowercase hex-encoded strings in the JSONRPC.
-   Ethereum addresses MUST be encoded as all other binary data (`0x`-prefix, all lowercase, no EIP-55 checksum).
-   Unless otherwise specified, values MUST use their equivalent JSON types (as shown in the examples).

## Quotes

Dealer implementations use the [0x contract system](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md) for asset settlement, and as such, the trading interface provided by the Dealer API is designed to foster helpful abstractions over the underlying settlement system.

For this reason, instead of providing a currency pair (base/quote and price denominated markets) interface for quote requests,
traders are able to specify the `makerAsset`, `takerAsset`, and one of either `makerAssetSize` or `takerAssetSize` (the other provided by the dealer as the quote). Price can then be calculated in terms of either asset at higher levels depending on the use case. Similarly, all asset sizes included in requests and responses from the dealer MUST be in integer base units.

Because there is no concept of a base or quote asset, quotes include no notion of price. Instead allowing clients to calculate the price in terms of either asset.

Implementations MAY choose what types of markets to support, to replicate more conventional trading systems. Consider the following requests (syntax: `[ MAKER_ASSET, TAKER_ASSET, MAKER_SIZE, TAKER_SIZE ]`).

```json
[
    ["DAI", "ZRX", null, 100000000000000000000],
    ["ZRX", "DAI", 100000000000000000000, null],
    ["ZRX", "DAI", null, 10000000000000000000],
    ["DAI", "ZRX", 10000000000000000000, null]
]
```

1. Client requests to swap 100 ZRX for DAI (receiving the DAI, sending the ZRX)
    - If the market's quote asset is **DAI** this is a request to **sell ZRX** to the dealer.
    - If the market's quote asset is **ZRX** this is a request to **buy DAI** from the dealer.
1. Client requests to swap 100 ZRX for DAI (receiving the ZRX, sending the DAI)
    - If the market's quote asset is **DAI** this is a request to **buy ZRX** from the dealer.
    - If the market's quote asset is **ZRX** this is a request to **sell DAI** to the dealer.
1. Client requests to swap 10 DAI for ZRX (receiving the ZRX, sending the DAI)
    - If the market's quote asset is **DAI** this is a request to **buy ZRX** to the dealer.
    - If the market's quote asset is **ZRX** this is a request to **sell DAI** to the dealer.
1. Client requests to swap 10 DAI for ZRX (receiving the DAI, sending the ZRX)
    - If the market's quote asset is **DAI** this is a request to **sell ZRX** to the dealer.
    - If the market's quote asset is **ZRX** this is a request to **buy DAI** from the dealer.

## Pagination

Paginated methods MUST implement pagination in accordance with this section. If `page` is not specified, the default MUST be 0. The value for `perPage` MAY be implementation specific.

Paginated methods MUST include two additional parameters in the `params` array or object (where `n` is the number of request parameters). The response parameters MUST match the table below.

-   **Request parameters:**

    | Index   | Name      | JSON Type | Required | Default        | Description                                                                                             |
    | :------ | :-------- | :-------- | :------- | :------------- | :------------------------------------------------------------------------------------------------------ |
    | `n - 2` | `page`    | Number    | `No`     | `0`            | The page number of the paginated results, where there are `perPage` items on each page.                 |
    | `n - 1` | `perPage` | Number    | `No`     | Impl. specific | The number of items to include on each page (used by the server to calculate which results to include). |

-   **Response parameters:**

    | Index | Name      | JSON Type | Schema   | Description                                                     |
    | :---- | :-------- | :-------- | :------- | :-------------------------------------------------------------- |
    | `0`   | `records` | Array     | Array<T> | The array of matching record of schema T.                       |
    | `1`   | `total`   | Number    | -        | The total matching records for the query (length of `records`). |
    | `2`   | `page`    | Number    | -        | MUST match `page` request.                                      |
    | `3`   | `perPage` | Number    | -        | MUST match `perPage` request.                                   |

## Errors

Implementations MUST support the reserved error codes specified in [the JSONRPC 2.0](https://www.jsonrpc.org/specification#error_object) specification, in addition to any method-specific errors. Be sure to see the [error codes](#error-codes) appendix.

Implementations MAY omit specific error codes (in the `-42000` range) entirely, but MUST support the ones specified by JSONRPC if no specific codes are provided.

Implementations MAY add their own error codes and messages, and may use the `data` field provided by the JSONRPC specification.

Implementations MUST only adhere to the defined codes for the defined scenarios. The exact message text MAY not match the specification exactly, and implementations MAY omit certain specified codes.

Implementations MUST NOT used any defined error code to mean something contrary to what the specification defines.

## Schemas

Schematics and data structures used in the public API (JSON shown).

Fields indicated `Yes` in the `Required` column for each scheme MUST be implemented, while fields indicated `No` MAY be omitted.

All schemas in this section MUST be supported to the degree indicated in each section.

### Schema: `Ticker`

An asset's shorthand String representation.

No limitation is placed on tickers by the specification, but implementations SHOULD keep an asset's `Ticker` consistent with established conventions for that asset on it's corresponding network (e.g. DAI, WETH, etc.).

This value SHOULD come from the [ERC-20 `symbol`](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#symbol) method if implemented and logical for each supported asset.

-   **JSON Example:**

    ```json
    "ZRX"
    ```

### Schema: `Time`

All times are specified in seconds since the UNIX epoch as a Number.

Implementations MAY choose the level of time precision (decimals), which MUST be consistent across the public API.

If an implementation provides sub-second precision for timing, the decimal place and trailing zeros MAY be omitted if applicable.

-   **JSON Example:**

    ```json
    1573774183.1353
    ```

### Schema: `UUID`

A universally unique identifier (UUID), according to UUID version 4.0 as a String.

-   **JSON Example:**
    ```
    "bafa9565-598d-413a-80d3-7ec3b7e24a08"
    ```

### Schema: `TradeInfo`

Defines information about trades – settlement transactions sent to the 0x exchange contract.

Implementations MAY included implementation-specific fields in this section.

The value for `gasPrice` MUST match the value ultimately included in any 0x [fill transaction](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#transactions) by dealer implementations for a given market.

-   **Fields**:

    | Name       | Schema | JSON Type | Description                                                                                   |
    | :--------- | :----- | :-------- | :-------------------------------------------------------------------------------------------- |
    | `gasLimit` | -      | Number    | The gas limit that will be used in `fillOrder` transactions submitted by the dealer.          |
    | `gasPrice` | -      | Number    | The gas price (in wei) that will be used in `fillOrder` transactions submitted by the dealer. |

-   **JSON Example**:

    ```json
    {
        "gasLimit": 210000,
        "gasPrice": 12000000000
    }
    ```

### Schema: `QuoteInfo`

Defines information about quote parameters for a given market. Does NOT included specific validity parameters (`ValidityParameter`) for individual quotes.

-   **Fields**:

    | Name              | Schema | JSON Type | Description                                                                               |
    | :---------------- | :----- | :-------- | :---------------------------------------------------------------------------------------- |
    | `minSize`         | -      | Number    | The minimum supported trade size, in base units of a market's maker asset.                |
    | `maxSize`         | -      | Number    | The maximum supported trade size, in base units of a market's maker asset.                |
    | `durationSeconds` | -      | Number    | The validity duration of quotes for the market in seconds (`0` indicating no expiration). |

-   **JSON Example**:

    ```json
    {
        "minSize": 100000000000000,
        "maxSize": 10000000000000000000000000,
        "durationSeconds": 15
    }
    ```

### Schema: `ValidityParameter`

An implementation-specific quote validation parameter used to enforce "soft cancels" (rejection to fill quotes).

These parameters are NOT cryptographically enforced, and MAY be used by implementations to enforce custom "soft" cancellation parameters.

Implementations MAY provide additional fields in this schema (such as a link to a price feed, etc.).

-   Fields:

    | Name    | Schema | JSON Type | Description                                          |
    | :------ | :----- | :-------- | :--------------------------------------------------- |
    | `name`  | -      | String    | The name of the validation parameter.                |
    | `value` | -      | Any       | The enforced value (MAY be primitive or structured). |

-   **JSON Example**:

    This example may indicate to a trader that the dealer will only fill the quote if the ETH/USD price on Coinbase is above 176.25 USD.

    ```json
    {
        "name": "cb-eth-usd-price-above",
        "value": 176.25
    }
    ```

### Schema: `Asset`

Defines information about an asset supported by a dealer implementation.

-   **Fields**:

    | Name        | Schema                   | Required | JSON Type | Description                                                                                                                                                                   |
    | :---------- | :----------------------- | :------- | :-------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `ticker`    | [Ticker](#schema-ticker) | `Yes`    | String    | Short-form name of the ERC-20 asset. SHOULD match the value provided by the contract.                                                                                         |
    | `name`      | -                        | `Yes`    | String    | Long-form name of the ERC-20 asset. SHOULD match the value provided by the contract.                                                                                          |
    | `decimals`  | -                        | `Yes`    | Number    | The number of decimals used in the tokens user representation (see [EIP-20](https://eips.ethereum.org/EIPS/eip-20)).                                                          |
    | `networkId` | -                        | `Yes`    | Number    | The [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) network ID of the active Ethereum network (2).                                                    |
    | `assetData` | -                        | `Yes`    | String    | [ABIv2 encoded asset data](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#assetdata) (including address) as used in the 0x system. |

-   **JSON Example**:

    ```json
    {
        "ticker": "DAI",
        "name": "DAI Stablecoin (v1.0)",
        "decimals": 18,
        "networkId": 1,
        "assetData": "0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359"
    }
    ```

### Schema: `Market`

Defines a market: a trading venue that supports a maker asset (provided by the dealer) and at least one taker asset (provided by the trader).

The concept of a base and quote asset are intentionally omitted, and left for definition at higher levels based on the trading and market scenario.

Implementations MAY choose an arbitrary format for the `marketId` (UUIDs as shown in the example are OPTIONAL).

-   **Fields**:

    | Name                | Schema                           | Required | JSON Type | Description                                                                          |
    | :------------------ | :------------------------------- | :------- | :-------- | :----------------------------------------------------------------------------------- |
    | `marketId`          | -                                | Yes      | String    | An implementation-specific market ID string. MUST be unique for each market.         |
    | `makerAssetTicker`  | [Ticker](#schema-ticker)         | `Yes`    | String    | The shorthand ticker of the markets maker asset (provided by the dealer).            |
    | `takerAssetTickers` | Array\<[Ticker](#schema-ticker)> | `Yes`    | Array     | An array of shorthand tickers for which quotes are supported.                        |
    | `tradeInfo`         | [TradeInfo](#schema-tradeinfo)   | `Yes`    | Object    | Information about trade settlement and execution for this market (gas price, etc.).  |
    | `quoteInfo`         | [QuoteInfo](#schema-quoteinfo)   | `Yes`    | Object    | Information about quotes provided on this market (max/min size, etc.).               |
    | `metadata`          | -                                | `No`     | Object    | Optional and implementation-specific key-value pairs for additional market metadata. |

-   **JSON Example**:

    ```json
    {
        "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
        "makerAssetTicker": "WETH",
        "takerAssetTickers": ["DAI", "USDC", "MKR", "ZRX"],
        "tradeInfo": {
            "networkId": 1,
            "gasLimit": 210000,
            "gasPrice": 12000000000
        },
        "quoteInfo": {
            "minSize": 100000000000000,
            "maxSize": 10000000000000000000000000,
            "durationSeconds": 15
        },
        "metadata": {}
    }
    ```

### Schema: `Order`

A JSON representation of a dealer-signed [0x order.](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#orders)

MUST be implemented according to the 0x specification. Operations with 0x order messages SHOULD be carried out with [official 0x libraries](https://github.com/0xProject/0x-monorepo) (Go implementations [here](https://github.com/0xProject/0x-mesh)).

### Schema: `Quote`

Defines a price quote from a dealer for a given maker and taker asset, and other quote parameters.

Implementations MAY use the `validityParameters` field to specify custom "soft cancel" parameters for served quotes.

-   **Fields**:

    | Name                 | Schema                    | Required | JSON Type | Description                                                                                                                                                                                |
    | :------------------- | :------------------------ | :------- | :-------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `quoteId`            | [UUID](#schema-uuid)      | `Yes`    | String    | A UUID (v4) that MUST correspond to this offer only.                                                                                                                                       |
    | `makerAssetTicker`   | [Ticker](#schema-ticker)  | `Yes`    | String    | Shorthand ticker of the quote's maker asset (see [quotes](#quotes)).                                                                                                                       |
    | `takerAssetTicker`   | [Ticker](#schema-ticker)  | `Yes`    | String    | Shorthand ticker of the quote's taker asset (see [quotes](#quotes)).                                                                                                                       |
    | `makerAssetSize`     | -                         | `Yes`    | Number    | The quote's maker asset size provided by the dealer (see [quotes](#quotes)).                                                                                                               |
    | `quoteAssetSize`     | -                         | `Yes`    | Number    | The quote's taker asset size required by the client (see [quotes](#quotes)).                                                                                                               |
    | `expiration`         | [Time](#schema-time)      | `Yes`    | Number    | The UNIX timestamp after which the quote will be rejected for settlement.                                                                                                                  |
    | `orderHash`          | -                         | `No`     | String    | The 0x-specific order hash, as defined in the [v3 specification](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#hashing-an-order).              |
    | `order`              | [Order](#schema-order)    | `No`     | Object    | The dealer-signed [0x order](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#orders) that corresponds to this offer.                             |
    | `fillTx`             | -                         | `No`     | String    | The raw [0x fill transaction](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#transactions) data for this quote that the taker may sign (see 6). |
    | `validityParameters` | Array\<ValidityParameter> | `No`     | Array     | OPTIONAL implementation-specific "soft-cancel" parameters for this offer.                                                                                                                  |

-   **JSON Example**:

    ```json
    {
        "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "makerAssetTicker": "ZRX",
        "takerAssetTicker": "WETH",
        "makerAssetSize": 100000000000000000000,
        "takerAssetSize": 300000000000000000,
        "expiration": 1573775025,
        "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
        "order": {
            "makerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
            "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
            "feeRecipientAddress": "0x",
            "senderAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
            "makerAssetAmount": "100000000000000000000",
            "takerAssetAmount": "300000000000000000",
            "makerFee": "0",
            "takerFee": "0",
            "exchangeAddress": "0x080bf510fcbf18b91105470639e9561022937712",
            "expirationTimeSeconds": "1573790025",
            "signature": "0x1cc41fd3abd90ade56ae73626247516dfaa2ab8813a7938c20504376a3e52d2511438fcaac7f812eaa2138b67ef9b201c55d7f7eaa7301c0c8540ca3afbd0eea1202",
            "salt": "1572620203025",
            "makerAssetData": "0xf47261b0000000000000000000000000e41d2489571d322189246dafa5ebde1f4699f498",
            "takerAssetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
            "makerFeeAssetData": "0x",
            "takerFeeAssetData": "0x"
        },
        "validityParameters": [
            {
                "name": "coinbase-pro-eth-usd-above",
                "value": 178.88
            }
        ]
    }
    ```

### Schema: `Trade`

Defines a past (settled) trade from a dealer.

-   **Fields**:

    | Name               | Schema                   | Required | JSON Type | Description                                                           |
    | :----------------- | :----------------------- | :------- | :-------- | :-------------------------------------------------------------------- |
    | `quoteId`          | [UUID](#schema-uuid)     | `Yes`    | String    | The ID of the original quote that was filled in a trade.              |
    | `marketId`         | -                        | `Yes`    | String    | Implementation-specific ID corresponding to the correct market.       |
    | `orderHash`        | -                        | `Yes`    | String    | The 0x order hash of the order filled in a trade.                     |
    | `transactionHash`  | -                        | `Yes`    | String    | The Ethereum transaction hash (transaction ID) of fill.               |
    | `takerAddress`     | -                        | `Yes`    | String    | The Ethereum address of the taker who requested and filled the quote. |
    | `timestamp`        | [Time](#schema-time)     | `Yes`    | Number    | The UNIX timestamp the fill was submitted (or mined) at.              |
    | `makerAssetTicker` | [Ticker](#schema-ticker) | `Yes`    | String    | The ticker of the trade's maker asset.                                |
    | `takerAssetTicker` | [Ticker](#schema-ticker) | `Yes`    | String    | The ticker of the trade's taker asset.                                |
    | `makerAssetAmount` | -                        | `Yes`    | Number    | The amount of the maker asset transacted in the trade.                |
    | `takerAssetAmount` | -                        | `Yes`    | Number    | The amount of the taker asset transacted in the trade.                |

-   **JSON Example**:

    ```json
    {
        "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
        "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
        "transactionHash": "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
        "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
        "timestamp": 1574108114.3301,
        "makerAssetTicker": "WETH",
        "takerAssetTicker": "DAI",
        "makerAssetAmount": 883000000000000000,
        "takerAssetAmount": 143500000000000000000
    }
    ```

## Methods

### Method: `dealer_authStatus`

Get dealer-specific authorization status for a given taker address.

If the dealer is operating a blacklist or whitelist, this endpoint can inform the client why (or why not) they are able to trade.

Implementation MAY use arbitrary mechanisms to determine a taker's authorization, or place no restriction upon takers.

-   **Request fields:**

    | Index | Name           | JSON Type | Required | Default | Description                                                           |
    | :---- | :------------- | :-------- | :------- | :------ | :-------------------------------------------------------------------- |
    | `0`   | `takerAddress` | String    | `Yes`    | -       | The 20-byte Ethereum address of the taker to check authorization for. |

-   **Response fields:**

    | Index | Name         | JSON Type | Schema | Description                                                                                                                                         |
    | :---- | :----------- | :-------- | :----- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `0`   | `authorized` | Boolean   | -      | True or false, indicating the taker's authorization based on the implementation's configured access control mode (e.g. whitelist, blacklist, etc.). |
    | `1`   | `reason`     | String    | -      | A code or human-readable justification for the authorization status.                                                                                |

-   **Errors:**

    | Code     | Description            | Notes                                           |
    | :------- | :--------------------- | :---------------------------------------------- |
    | `-42001` | Invalid taker address. | Returned when an address is invalid or missing. |

-   **Example request bodies:**

    ```json
    {
        "takerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459"
    }
    ```

    ```json
    ["0xcefc94f1c0a0be7ad47c7fd961197738fc233459"]
    ```

-   **Example response bodies:**

    ```json
    {
        "authorized": false,
        "reason": "BLACKLISTED"
    }
    ```

    ```json
    [false, "BLACKLISTED"]
    ```

### Method: `dealer_getAssets`

Fetch information about currently supported ERC-20 assets, and optionally filter with query parameters. This method is [paginated.](#pagination)

With no request parameters, this method MUST return a paginated array of all supported assets.

All parameters to this method (with the exception of `page` and `perPage`) act as filter parameters, returning only results that match all specified parameters.

This method MUST return an empty array if no results match the query. Implementations MAY return an error (`-32002`) if conflicting query parameters are provided.

-   **Request fields:**

    | Index | Name        | JSON Type | Required | Default        | Description                                                          |
    | :---- | :---------- | :-------- | :------- | :------------- | :------------------------------------------------------------------- |
    | `0`   | `address`   | String    | `No`     | `null`         | Match only assets with this address. MUST return only one result.    |
    | `1`   | `ticker`    | String    | `No`     | `null`         | Match only assets with this ticker. MUST return only one result.     |
    | `2`   | `assetData` | String    | `No`     | `null`         | Match only assets with this asset data. MUST return only one result. |
    | `3`   | `networkId` | Number    | `No`     | `1`            | Only match assets with this network ID.                              |
    | `4`   | `page`      | Number    | `No`     | `0`            | See [pagination.](#pagination)                                       |
    | `5`   | `perPage`   | Number    | `No`     | Impl. specific | See [pagination.](#pagination)                                       |

-   **Response fields:**

    | Index | Name      | JSON Type | Schema        | Description                                                     |
    | :---- | :-------- | :-------- | :------------ | :-------------------------------------------------------------- |
    | `0`   | `records`  | Array     | Array\<Asset> | The array of asset results that match the request parameters.   |
    | `1`   | `total`   | Number    | -             | The number of results that matched the request (MAY be 0).      |
    | `2`   | `page`    | Number    | -             | The page index of the result (MUST match request).              |
    | `3`   | `perPage` | Number    | -             | The number of items included on each page (MUST match request). |

-   **Errors:**

    | Code     | Description               | Notes                                                              |
    | :------- | :------------------------ | :----------------------------------------------------------------- |
    | `-42002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested.   |
    | `-42003` | Invalid asset address.    | Returned when an invalid Ethereum address is provided.             |
    | `-42004` | Invalid asset data.       | Returned when malformed ABIv2 asset data is included in a request. |

-   **Example request bodies:**

    ```json
    {
        "page": 0,
        "perPage": 2,
        "networkId": 1
    }
    ```

    ```json
    [null, null, 1, 0, 2]
    ```

-   **Example response bodies:**

    ```json
    {
        "page": 0,
        "perPage": 2,
        "total": 2,
        "records": [
            {
                "ticker": "DAI",
                "name": "DAI Stablecoin (v1.0)",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359"
            },
            {
                "ticker": "WETH",
                "name": "Wrapped Ether",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"
            }
        ]
    }
    ```

    ```json
    [
        [
            {
                "ticker": "DAI",
                "name": "DAI Stablecoin (v1.0)",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359"
            },
            {
                "ticker": "WETH",
                "name": "Wrapped Ether",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"
            }
        ],
        2,
        0,
        2
    ]
    ```

### Method: `dealer_getMarkets`

Fetch information about currently supported [markets](#schema-market). This method is [paginated.](#pagination)

This method, with no parameters, MUST return a paginated array of all supported markets.

All parameters to this method (with the exception of `page` and `perPage`) act as filter parameters, returning only results that match all specified parameters.

This method MUST return an empty array if no results match the query. Implementations MAY return an error (e.g. `-42002`) if conflicting query parameters are provided.

-   **Request fields:**

    | Index | Name               | JSON Type | Required | Default        | Description                                                    |
    | :---- | :----------------- | :-------- | :------- | :------------- | :------------------------------------------------------------- |
    | `0`   | `makerAssetTicker` | String    | `No`     | `null`         | Match only markets with this maker asset.                      |
    | `1`   | `takerAssetTicker` | String    | `No`     | `null`         | Match only markets that support this taker asset ticker.       |
    | `2`   | `marketId`         | String    | `No`     | `null`         | Match only the market with this ID. MUST match 0 or 1 markets. |
    | `3`   | `networkId`        | Number    | `No`     | `1`            | Match only markets supporting this network.                    |
    | `4`   | `page`             | Number    | `No`     | `0`            | See [pagination.](#pagination)                                 |
    | `5`   | `perPage`          | Number    | `No`     | Impl. specific | See [pagination.](#pagination)                                 |

-   **Response fields:**

    | Index | Name      | JSON Type | Schema         | Description                                                    |
    | :---- | :-------- | :-------- | :------------- | :------------------------------------------------------------- |
    | `0`   | `records` | Array     | Array\<Market> | The array of market results that match the request parameters. |
    | `1`   | `total`   | Number    | -              | The number of results that matched the request (MAY be 0).     |
    | `2`   | `page`    | Number    | -              | The page index of the result (MUST match request).             |
    | `3`   | `perPage` | Number    | -              | The array of asset results that match the request parameters.  |

-   **Errors:**

    | Code     | Description               | Notes                                                            |
    | :------- | :------------------------ | :--------------------------------------------------------------- |
    | `-42002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested. |

-   **Example request bodies:**

    ```json
    {
        "makerAssetTicker": "WETH",
        "networkId": 1,
        "page": 0,
        "perPage": 2
    }
    ```

    ```json
    ["WETH", null, null, 1, 0, 2]
    ```

-   **Example response bodies:**

    ```json
    {
        "page": 0,
        "perPage": 2,
        "total": 2,
        "records": [
            {
                "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
                "makerAssetTicker": "WETH",
                "takerAssetTickers": ["DAI", "MKR", "ZRX"],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPrice": 12000000000
                },
                "quoteInfo": {
                    "minSize": 100000000000000,
                    "maxSize": 100000000000000000000,
                    "durationSeconds": 15
                }
            },
            {
                "marketId": "87c0ee47-44c0-4ff0-ba68-6638c79c11dd",
                "makerAssetTicker": "WETH",
                "takerAssetTickers": ["USDC"],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPrice": 12000000000
                },
                "quoteInfo": {
                    "minSize": 100000000000000,
                    "maxSize": 100000000000000000000,
                    "durationSeconds": 15
                }
            }
        ]
    }
    ```

    ```json
    [
        [
            {
                "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
                "makerAssetTicker": "WETH",
                "takerAssetTickers": ["DAI", "MKR", "ZRX"],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPrice": 12000000000
                },
                "quoteInfo": {
                    "minSize": 100000000000000,
                    "maxSize": 100000000000000000000,
                    "durationSeconds": 15
                }
            },
            {
                "marketId": "87c0ee47-44c0-4ff0-ba68-6638c79c11dd",
                "makerAssetTicker": "WETH",
                "takerAssetTickers": ["USDC"],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPrice": 12000000000
                },
                "quoteInfo": {
                    "minSize": 100000000000000,
                    "maxSize": 100000000000000000000,
                    "durationSeconds": 15
                }
            }
        ],
        2,
        0,
        2
    ]
    ```

### Method: `dealer_getPastTrades`

Get records of past trades. This method is [paginated.](#pagination)

If no filter parameters are provided, this method MUST return a paginated array of all past trades.

The specification for this method (and the `Trade` schema) has been designed such that, at minimum, implementations must only associate the quote ID and market ID with a transaction hash.

All other fields can be dynamically populated from 0x event logs based on a known transaction hash.

-   **Request fields:**

    | Index | Name               | JSON Type | Required | Default        | Description                                                      |
    | :---- | :----------------- | :-------- | :------- | :------------- | :--------------------------------------------------------------- |
    | `0`   | `quoteId`          | String    | `No`     | `null`         | Match only the trade with this quote ID. MUST match 0 or 1 item. |
    | `1`   | `marketId`         | String    | `No`     | `null`         | Match only trades for this market ID.                            |
    | `2`   | `takerAddress`     | String    | `No`     | `null`         | Match only trades filled by this taker address.                  |
    | `3`   | `transactionHash`  | String    | `No`     | `null`         | Match only the trade with this TX ID. MUST match 0 or 1 item.    |
    | `4`   | `orderHash`        | String    | `No`     | `null`         | Match only the trade with this order ID. MUST match 0 or 1 item. |
    | `5`   | `makerAssetTicker` | String    | `No`     | `null`         | Match only trades where this ticker was the maker asset.         |
    | `6`   | `takerAssetTicker` | String    | `No`     | `null`         | Match only trades where this ticker was the taker asset.         |
    | `7`   | `page`             | Number    | `No`     | 0              | See [pagination.](#pagination)                                   |
    | `7`   | `perPage`          | Number    | `No`     | Impl. specific | See [pagination.](#pagination)                                   |

-   **Response fields:**

    | Index | Name      | JSON Type | Schema        | Description                                                            |
    | :---- | :-------- | :-------- | :------------ | :--------------------------------------------------------------------- |
    | `0`   | `records`  | Array     | Array\<Trade> | The array of trade results that match the request (MAY be all trades). |
    | `1`   | `total`   | Number    | -             | The number of results that matched the request (MAY be 0).             |
    | `2`   | `page`    | Number    | -             | The page index of the result (MUST match request).                     |
    | `3`   | `perPage` | Number    | -             | The array of asset results that match the request parameters.          |

-   **Errors:**

    | Code     | Description               | Notes                                                                             |
    | :------- | :------------------------ | :-------------------------------------------------------------------------------- |
    | `-32603` | Internal error.           | Internal JSON-RPC error. MAY be used as generic internal error code.              |
    | `-42002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested.                  |
    | `-42003` | Invalid address.          | Returned when an invalid Ethereum address is provided.                            |
    | `-42021` | Invalid transaction ID.   | Available to indicate an invalid transaction hash in a request.                   |
    | `-42022` | Invalid order hash.       | Available to indicate an order transaction hash in a request.                     |
    | `-42023` | Invalid UUID.             | Available to indicate failure to validate a universally unique identifier (UUID). |

*   **Example request bodies:**

    ```json
    {
        "transactionHash": "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
        "page": 0,
        "perPage": 10
    }
    ```

    ```json
    [null, null, null, "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65", null, null, null, 0, 10]
    ```

*   **Example response bodies:**

    ```json
    {
        "page": 0,
        "perPage": 10,
        "items": 1,
        "records": [
            {
                "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
                "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
                "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
                "transactionHash": "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
                "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
                "timestamp": 1574108114.3301,
                "makerAssetTicker": "WETH",
                "takerAssetTicker": "DAI",
                "makerAssetAmount": 883000000000000000,
                "takerAssetAmount": 143500000000000000000
            }
        ]
    }
    ```

    ```json
    [
        [
            {
                "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
                "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
                "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
                "transactionHash": "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
                "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
                "timestamp": 1574108114.3301,
                "makerAssetTicker": "WETH",
                "takerAssetTicker": "DAI",
                "makerAssetAmount": 883000000000000000,
                "takerAssetAmount": 143500000000000000000
            }
        ],
        1,
        0,
        10
    ]
    ```

### Method: `dealer_getQuote`

Primary method for requesting a quote from the dealer. Be sure to see the [quotes section.](#quotes)

To request a quote, a client MUST specify the maker and taker asset (by ticker) and either a maker size or a taker size.

Implementations MAY allow traders to specify _both_ the maker and taker asset sizes, but this is highly illogical in most scenarios.

Implementations MAY choose to support arbitrary swap quotes or simply return the corresponding error code (`-42008` or `-42009`) if the quote requested by the trader is unsupported.

Clients SHOULD leave at least one size field (either `makerAssetSize` or `takerAssetSize`) as `null` or not specified that the dealer will fill-in to provide the price quote.

-   **Request fields:**

    | Index | Name               | JSON Type | Required | Default           | Description                                                                                                                                                                               |
    | :---- | :----------------- | :-------- | :------- | :---------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `0`   | `makerAssetTicker` | String    | `Yes`    | -                 | Specify the maker asset of the quote (sent by the dealer).                                                                                                                                |
    | `1`   | `takerAssetTicker` | String    | `Yes`    | -                 | Specify the taker asset of the quote (sent by the client).                                                                                                                                |
    | `2`   | `makerAssetSize`   | Number    | `No`     | `null`            | Client MUST specify either this or `takerAssetSize`.                                                                                                                                      |
    | `3`   | `takerAssetSize`   | Number    | `No`     | `null`            | Client MUST specify either this or `makerAssetSize`.                                                                                                                                      |
    | `4`   | `takerAddress`     | String    | `No`     | (See [3](#notes)) | The address of the taker that will fill the requested quote (see 4).                                                                                                                      |
    | `5`   | `includeOrder`     | Boolean   | `No`     | `true`            | If `true`, the quote MUST include a signed 0x order for the offer (5).                                                                                                                    |
    | `6`   | `includeTx`        | Boolean   | `No`     | `false`           | If `true`, the quote MUST included the [ABIv2 encoded fill transaction data](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#transactions) (6). |
    | `7`   | `extra`            | Object    | `No`     | `null`            | Optional extra structured data from the taker. MAY be omitted by implementations.                                                                                                         |

-   **Response fields:**

    | Index | Name        | JSON Type | Schema                         | Description                                               |
    | :---- | :---------- | :-------- | :----------------------------- | :-------------------------------------------------------- |
    | `0`   | `quote`     | Object    | [Quote](#schema-quote)         | A quote (offer) for the specified values from the client. |
    | `1`   | `tradeInfo` | Object    | [TradeInfo](#schema-tradeinfo) | Settlement information (e.g. gas price).                  |
    | `2`   | `extra`     | Object    | -                              | OPTIONAL extra structured data relevant to this offer.    |

-   **Errors:**

    | Code     | Description                         | Notes                                                                                      |
    | :------- | :---------------------------------- | :----------------------------------------------------------------------------------------- |
    | `-42005` | Two size requests.                  | Occurs when a client specifies both `baseAssetSize` and `quoteAssetSize`.                  |
    | `-42006` | Taker not authorized.               | Occurs when the taker's address is not authorized for trading.                             |
    | `-42007` | Invalid side.                       | Available for implementations to indicate lack of support for a currency-pair style quote. |
    | `-42008` | Temporary restriction.              | Available for implementations to indicate taker-specific temporary restrictions.           |
    | `-42009` | Unsupported market.                 | Occurs when the specified market (quote and base pair) is not supported.                   |
    | `-42010` | Unsupported taker asset for market. | Available for implementations to indicate lack of support for arbitrary swaps.             |
    | `-42011` | Quote too large.                    | Occurs when a quote would exceed the market maximum or the dealer's balance.               |
    | `-42012` | Quote too small.                    | Occurs when a quote would be smaller than the market's minimum size.                       |
    | `-42013` | Quote unavailable at this time.     | Reserved for various states where dealers may not be serving quotes.                       |

-   **Example request bodies:**

    ```json
    {
        "makerAssetTicker": "ZRX",
        "takerAssetTicker": "DAI",
        "makerAssetSize": 1435000000000000000,
        "takerAddress": "0xcefc94F1C0a0bE7aD47c7fD961197738fC233459",
        "includeOrder": true,
        "includeTx": false
    }
    ```

    ```json
    [
        "ZRX",
        "DAI",
        1435000000000000000,
        null,
        "0xcefc94F1C0a0bE7aD47c7fD961197738fC233459"
        true,
        false
    ]
    ```

-   **Example response bodies:**

    ```json
    {
        "tradeInfo": {
            "networkId": 1,
            "gasLimit": 210000,
            "gasPrice": 12000000000
        },
        "quote": {
            "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
            "makerAssetTicker": "ZRX",
            "takerAssetTicker": "DAI",
            "makerAssetSize": 1435000000000000000,
            "takerAssetSize": 300000000000000000,
            "expiration": 1573775025,
            "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
            "order": {
                "makerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
                "feeRecipientAddress": "0x",
                "senderAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "makerAssetAmount": "1435000000000000000",
                "takerAssetAmount": "300000000000000000",
                "makerFee": "0",
                "takerFee": "0",
                "exchangeAddress": "0x080bf510fcbf18b91105470639e9561022937712",
                "expirationTimeSeconds": "1573790025",
                "signature": "0x1cc41fd3abd90ade56ae73626247516dfaa2ab8813a7938c20504376a3e52d2511438fcaac7f812eaa2138b67ef9b201c55d7f7eaa7301c0c8540ca3afbd0eea1202",
                "salt": "1572620203025",
                "makerAssetData": "0xf47261b0000000000000000000000000e41d2489571d322189246dafa5ebde1f4699f498",
                "takerAssetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "makerFeeAssetData": "0x",
                "takerFeeAssetData": "0x"
            }
        }
    }
    ```

    ```json
    [
        {
            "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
            "makerAssetTicker": "ZRX",
            "takerAssetTicker": "DAI",
            "makerAssetSize": 1435000000000000000,
            "takerAssetSize": 300000000000000000,
            "expiration": 1573775025,
            "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
            "order": {
                "makerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
                "feeRecipientAddress": "0x",
                "senderAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "makerAssetAmount": "1435000000000000000",
                "takerAssetAmount": "300000000000000000",
                "makerFee": "0",
                "takerFee": "0",
                "exchangeAddress": "0x080bf510fcbf18b91105470639e9561022937712",
                "expirationTimeSeconds": "1573790025",
                "signature": "0x1cc41fd3abd90ade56ae73626247516dfaa2ab8813a7938c20504376a3e52d2511438fcaac7f812eaa2138b67ef9b201c55d7f7eaa7301c0c8540ca3afbd0eea1202",
                "salt": "1572620203025",
                "makerAssetData": "0xf47261b0000000000000000000000000e41d2489571d322189246dafa5ebde1f4699f498",
                "takerAssetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "makerFeeAssetData": "0x",
                "takerFeeAssetData": "0x"
            }
        },
        {
            "networkId": 1,
            "gasLimit": 210000,
            "gasPrice": 12000000000
        },
        null
    ]
    ```

### Method: `dealer_submitFill`

Submit a previously-fetched quote for settlement. Can be thought of as a "request-to-fill" by a trader.

This method MUST support executing the signed fill transaction from the client, according to [ZEIP-18.](https://github.com/0xProject/ZEIPs/issues/18)

Implementations MAY use the `validityParameters` from previously-submitted quotes to reject fills based on external parameters.

The order or fill transaction data SHOULD be stored by implementations and associated with the quote ID so clients need not submit the raw fill transaction or the transaction hash.

Dealer implementations MAY assume the signer address is equal to the originally-provided `takerAddress` if not provided in the request-to-fill.

Implementations SHOULD strive to ONLY require the first three parameters for fill requests (`quoteId`, `salt`, and `signature`).

-   **Request fields:**

    | Index | Name        | JSON Type | Required | Default | Description                                                                        |
    | :---- | :---------- | :-------- | :------- | :------ | :--------------------------------------------------------------------------------- |
    | `0`   | `quoteId`   | String    | `Yes`    | -       | The ID of the original quote that is being submitted for settlement.               |
    | `1`   | `salt`      | String    | `Yes`    | -       | The salt used to generate the fill transaction hash and signature.                 |
    | `2`   | `signature` | String    | `Yes`    | -       | The taker's signature of the fill transaction data.                                |
    | `3`   | `signer`    | String    | `No`     | -       | The address that signed the fill. SHOULD be omitted (SHOULD match original taker). |
    | `4`   | `data`      | String    | `No`     | -       | The full fill transaction call data. SHOULD be omitted.                            |
    | `5`   | `hash`      | String    | `No`     | -       | The salted hash of the fill transaction. SHOULD be omitted.                        |

-   **Response fields:**

    | Index | Name              | JSON Type | Schema               | Description                                                                                                                                                  |
    | :---- | :---------------- | :-------- | :------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `0`   | `quoteId`         | String    | [UUID](#schema-uuid) | The UUID of the original quote that has been submitted for settlement.                                                                                       |
    | `1`   | `orderHash`       | String    | -                    | The [hash of the 0x order that](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#hashing-an-order) is being filled. |
    | `2`   | `transactionHash` | String    | -                    | The hash of the submitted fill transaction (transaction ID).                                                                                                 |
    | `3`   | `submittedAt`     | Number    | [Time](#schema-time) | The UNIX timestamp the fill transaction was submitted at.                                                                                                    |
    | `4`   | `extra`           | Object    | -                    | OPTIONAL implementation-specific relevant structured data.                                                                                                   |

-   **Errors:**

    | Code     | Description                   | Notes                                                                                             |
    | :------- | :---------------------------- | :------------------------------------------------------------------------------------------------ |
    | `-42014` | Quote expired.                | MUST be implemented and used ONLY when a request-to-fill is received after the quotes expiration. |
    | `-42015` | Unknown quote.                | Available to allow implementations differentiate expired from never-quoted.                       |
    | `-42016` | Already filled.               | Available to allow implementations to indicate specific double-fill attempts.                     |
    | `-42017` | Fill validation failed.       | Available to indicate current chain state simulation validation failure.                          |
    | `-42018` | Insufficient taker balance.   | Available to indicate specific validation failure.                                                |
    | `-42019` | Insufficient taker allowance. | Available to indicate specific validation failure.                                                |
    | `-42020` | Quote validation failure.     | Available to indicate implementation-specific failures of extra quote data.                       |
    | `-42023` | Invalid UUID.                 | Available to indicate failure to validate a universally unique identifier (UUID).                 |

*   **Example request bodies:**

    ```json
    {
        "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "salt": "1572620203025",
        "signature": "0xd90ade56ae73626247516dfaa2ab8813a7938c20504376a3e52d25114367ef9b201c55d7f7eaa7301c0c8540ca3afbd02"
    }
    ```

    ```json
    [
        "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "1572620203025",
        "0xd90ade56ae73626247516dfaa2ab8813a7938c20504376a3e52d25114367ef9b201c55d7f7eaa7301c0c8540ca3afbd02"
    ]
    ```

*   **Example response bodies:**

    ```json
    {
        "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
        "transactionHash": "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
        "submittedAt": 1574108114.3301
    }
    ```

    ```json
    [
        "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
        "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
        1574108114.3301
    ]
    ```

### Method: `dealer_time`

Fetch the current time from the dealer server.

Optionally provide a time in the request (`clientTime`) to get the difference (useful for clock synchronization and estimating network latency).

-   **Request fields:**

    | Index | Name         | JSON Type | Required | Default | Description                                                 |
    | :---- | :----------- | :-------- | :------- | :------ | :---------------------------------------------------------- |
    | `0`   | `clientTime` | Number    | `No`     | `null`  | A timestamp from the client to get a difference in seconds. |

-   **Response fields:**

    | Index | Name   | JSON Type | Schema               | Description                                                                                  |
    | :---- | :----- | :-------- | :------------------- | :------------------------------------------------------------------------------------------- |
    | `0`   | `time` | Number    | [Time](#schema-time) | The UNIX timestamp of the dealer's clock at the time of request.                             |
    | `1`   | `diff` | Number    | -                    | The difference between the dealer time and the client time. ONLY if `clientTime` in request. |

-   **Errors:**

    | Code     | Description     | Notes                                                                |
    | :------- | :-------------- | :------------------------------------------------------------------- |
    | `-32603` | Internal error. | Internal JSON-RPC error. MAY be used as generic internal error code. |

*   **Example request bodies:**

    ```json
    {
        "clientTime": 1574108764.1019
    }
    ```

    ```json
    [1574108764.1019]
    ```

*   **Example response bodies:**

    ```json
    {
        "time": 1574108764.2118,
        "diff": 0.1099
    }
    ```

    ```json
    [1574108764.2118, 0.1099]
    ```

## Appendix

### Important resources

-   [JSONRPC 2.0 Specification](https://www.jsonrpc.org/specification)
-   [0x Protocol Specification (v3)](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md)
-   [0x Improvement Proposal (ZEIP) 18](https://github.com/0xProject/ZEIPs/issues/18)
-   [EIP-20 (ERC-20 specification)](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#symbol)

### Notes

1. This definition of a market makes an intentional departure from conventional currency-pair based markets in which their is a single quote asset and a single base asset. Defining only the maker and taker assets for a market allows greater flexibility for implementers, and allows pricing to be defined in terms of either asset at higher levels.
1. If the dealer is operating on the main Ethereum network, they MUST treat the `networkID` of `1` as the Ethereum mainnet, as specified in EIP-155. Private and test networks may use any network ID, but SHOULD use conventions established by public test networks (e.g. Ropsten is 3).
1. The default value SHOULD be the null address (20 null bytes) represented as a hex string. Implementations MAY require takers to specify a `takerAddress`.
1. If a client requests a quote without an order, implementations MAY allow the client to get the order at a later time with a separate method. Quotes indicated as `includeOrder` as `false can be seen as traders checking if a dealer's prices are favorable at a given time for a certain market and trade size.
    - Implementations MAY treat these types of quotes separately in internal tracking and/or pricing mechanisms.
1. This feature is desirable for some users as it opens the door for clients to sign fill transactions with lower-level cryptographic primitives, rather than require the generally larger libraries required to prepare the fill transaction data from the order itself.

### Error codes

A table of all specified error codes, which MAY be used in methods other than where they are specified, if applicable.

| Code     | Description                         | Notes                                                                                             |
| :------- | :---------------------------------- | :------------------------------------------------------------------------------------------------ |
| `-32700` | Parse error.                        | Invalid JSON was received by the server. MUST be implemented.                                     |
| `-32600` | Invalid request.                    | The JSON sent is not a valid request object (see JSONRPC spec). MUST be implemented.              |
| `-32601` | Method not found.                   | The method does not exist or is not available.                                                    |
| `-32602` | Invalid parameters.                 | Invalid method parameters. MAY be omitted in favor of more specific codes.                        |
| `-32603` | Internal error.                     | Internal JSON-RPC error. MAY be used as generic internal error code.                              |
| `-42002` | Invalid filter selection.           | Returned when conflicting or incompatible filters are requested.                                  |
| `-42003` | Invalid address.                    | Returned when an invalid Ethereum address is provided.                                            |
| `-42004` | Invalid asset data.                 | Returned when malformed ABIv2 asset data is included in a request.                                |
| `-42005` | Two size requests.                  | Occurs when a client specifies `baseAssetSize` and `quoteAssetSize`.                              |
| `-42006` | Taker not authorized.               | Occurs when the taker's address is not authorized for trading.                                    |
| `-42007` | Invalid side.                       | Available for implementations to indicate lack of support for arbitrary swaps.                    |
| `-42008` | Temporary restriction.              | Available for implementations to indicate taker-specific temporary restrictions.                  |
| `-42009` | Unsupported market.                 | Occurs when the specified market (quote and base pair) is not supported.                          |
| `-42010` | Unsupported taker asset for market. | Available for implementations to indicate lack of support for arbitrary swaps.                    |
| `-42011` | Quote too large.                    | Occurs when a quote would exceed the market maximum or the dealer's balance.                      |
| `-42012` | Quote too small.                    | Occurs when a quote would be smaller than the market's minimum size.                              |
| `-42013` | Quote unavailable at this time.     | Reserved for various states where dealers may not be serving quotes.                              |
| `-42014` | Quote expired.                      | MUST be implemented and used ONLY when a request-to-fill is received after the quotes expiration. |
| `-42015` | Unknown quote.                      | Available to allow implementations differentiate expired from never-quoted.                       |
| `-42016` | Order already filled.               | Available to allow implementations to indicate specific double-fill attempts.                     |
| `-42017` | Fill validation failed.             | Available to indicate current chain state simulation validation failure.                          |
| `-42019` | Insufficient taker balance.         | Available to indicate specific validation failure.                                                |
| `-42019` | Insufficient taker allowance.       | Available to indicate specific validation failure.                                                |
| `-42020` | Quote validation failure.           | Available to indicate implementation-specific failures of extra quote data.                       |
| `-42021` | Invalid transaction ID.             | Available to indicate an invalid transaction hash in a request.                                   |
| `-42022` | Invalid order hash.                 | Available to indicate an order transaction hash in a request.                                     |
| `-42023` | Invalid UUID.                       | Available to indicate failure to validate a universally unique identifier (UUID).                 |
