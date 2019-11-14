# Dealer JSONRPC v1.0

Definitions and specification for the Dealer [JSONRPC (2.0)](https://www.jsonrpc.org/specification) API. This document specifies the first public Dealer JSONRPC version, `v1.0`.

Methods under the `dealer` namespace (with the `dealer_` prefix) comprise the public dealer API. Implementations MAY provide private administrative functionality through methods under a distinct namespace.

The Dealer JSONRPC can be served over WebSockets, HTTP POST, or HTTP GET, or other transports at the discretion of implementers (see notes).

### Conventions

- The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/rfc/rfc2119.txt).
- The word "implementation" is used to refer to any system that implements or provides the Dealer JSONRPC.

## Adherence requirements

In addition to notices in each section, each of the following must be true in order for an implementation to be considered in adherence with the specification.

These requirements are intended to motivate strong guarantees of compatibility between clients and servers and ensure maximum levels of safety for the operators of each: traders and dealers that implement this API.

- Implementations MUST implement all methods under the `dealer` namespace (see [Methods](#methods)).
- All supported assets MUST each have a unique string identifier called a "ticker" (e.g. DAI, ZRX, WETH).
- Implementations MUST use the canonical 0x v3 addresses for the active Ethereum network (not yet deployed).
- Implementations MUST only support ERC-20 assets (subject to change in future major API versions).
- Implementations MUST use arbitrary precision representations for integers.
- Implementations MAY support batch requests, in accordance with the JSONRPC 2.0 specification.
- Floating point numbers SHOULD be of fixed precision according to the precision of the parent market, if used at all.
- Pricing calculations SHOULD take place with the integer base unit representations of supported assets, and be represented as decimals only in the public API.
- Implementations SHOULD support Ether (ETH) trading, and if so, MUST do so via the canonical WETH contract for the active network.
- Implementations MAY require that quote requests include the potential taker's address.
	- The address provided by the taker MAY be used to restrict the `takerAddress` of the quotes underlying signed 0x order.
	- Implementations MAY record and use the address provided by the taker to alter prices or refuse to provide quotes.
- Implementations MAY use arrays or objects for return values and parameters (in accordance with the JSONRPC specification).
	- If arrays are used, the index specified in each method MUST match the implementation.
	- If objects are used, the keys MUST match the name specified for each parameter.
- Markets between assets with differing base unit representations MUST use a precision less than or equal to the number of decimals in the less-precise asset.
    - For example a DAI/USDC market MUST use 6 or less decimal places of precision.
- Implementations MAY choose to offer "arbitrary swap" functionality (e.g. Uniswap) or conventional bid/ask quotes only.
	- The specification permits markets to specify multiple quote assets, and a single base asset, which SHOULD be leveraged to implement swap functionality.



## Encoding notes
- Unless otherwise specified, values MUST use their equivalent JSON types (as shown in the examples).
- Binary data (EVM `bytes`, etc.) MUST be encoded as `0x`-prefixed all-lowercase hex-encoded strings in the JSONRPC. 
- Ethereum addresses MUST be encoded as all other binary data (`0x`-prefix, all lowercase, no EIP-55 checksum).

## Quotes

The [quote request interface](#method-dealer-fetchquote) specified provides flexibility for a wide array of market and pricing implementations.

For example, an implementation may choose to support `N` assets with `N` markets, where each has a stablecoin as the only quote asset. In this scenario, consider the following request bodies (according to the parameter ordering defined below).

```json
// [ BASE, QUOTE, BASE_SIZE, QUOTE_SIZE, SIDE ]

[
    ["ZRX", "DAI", 1000, null, "B"],
    ["ZRX", "DAI", null, 500, "B"],
    ["DAI", "ZRX", 500, null, "S"],
    ["DAI", "ZRX", null, 1000, "S"]
]
```
If an implementation chooses to only support conventional buys and sells on currency-pair markets (e.g. WETH/DAI), only the first request would be valid – where the quote is provided in the quote asset, requested by the trader in units of the base.

However, an implementation MAY choose to support all possible request formats. Such implementations are considered "arbitrary swap" marketplaces. The desire to offer flexibility for implementers and traders motivates the quote request format outlined in the [methods](#method-dealer-fetchquote) section, and the [market user representation](#schema-market) specified in the schemas section.


## Pagination

Paginated methods MUST implement pagination in accordance with this section.

Paginated methods MUST include two additional parameters in the `params` array or object (where `n` is the number of parameters the method accepts):

| Index | Name | JSON Type | Required | Default | Description |
| :---- | :--- | :-------- | :------- | :------ | :---------- |
| `n - 2` | `page` | `number` | `No` | `0` | The page number of the paginated results, where there are `perPage` items on each page. Must have a default value of `0` (the first page). |
| `n - 1` | `perPage` | `number` | `No` | Implementation specific | The number of items to include on each page (used by the server to calculate which results to include). |

## Errors

Implementations MUST support the reserved error codes specified in [the JSONRPC 2.0](https://www.jsonrpc.org/specification#error_object) specification, in addition to any method-specific errors. 

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

- **JSON Example:**

    ```json
    "ZRX"
    ```

### Schema: `Time`

All times are specified in seconds since the UNIX epoch.

Implementations MAY choose the level of time precision (decimals), which MUST be consistent across the public API.

If an implementation provides sub-second precision for timing, the decimal place and trailing zeros MAY be omitted if applicable.

- **JSON Example:**

    ```json
    1573774183.1353
    ```

### Schema: `Side`

Either the character `B` or `S` as an uppercase string.

Implementations MUST accept only the value `"B"` or `"S"` where the value applies ONLY to the corresponding description of each in the table below.

| Side | Value | JSON Type | Description |
| :--- | :---- | :-------- | :---------- |
| Buy | `B` | String | MUST indicate the client buying an amount of a market's base asset for an amount of a quote asset. |
| Sell | `S` | String | MUST indicate the client selling an amount of a market's base asset for an amount of a quote asset. |

- **JSON Example:**

    ```json
    "B"
    ```

### Schema: `UUID`

A universally unique identifier (UUID), according to UUID version 4.0.

- **JSON Example:**
    
    ```
    "bafa9565-598d-413a-80d3-7ec3b7e24a08"
    ```

### Schema: `TradeInfo`

Defines information about trades – settlement transactions sent to the 0x exchange contract.

- Fields:
  
  | Name | Schema | JSON Type | Description |
  | :--- | :----- | :-------- | :---------- |
  | `gasLimit` | - | `number` | The gas limit that will be used in `fillOrder` transactions submitted by the dealer. |
  | `gasPriceWei` | - | `number` | The gas price (in wei) that will be used in `fillOrder` transactions submitted by the dealer. |
  
- JSON Example:
  
    ```json
    {
        "gasLimit": 210000,
        "gasPriceWei": 12000000000
    }
    ```

### Schema: `QuoteInfo`

Defines information about quote parameters for a given market.

- Fields:
  
  | Name | Schema | JSON Type | Description |
  | :--- | :----- | :-------- | :---------- |
  | `minSize` | - | `number` | The minimum supported trade size, in units of a market's base asset. |
  | `maxSize` | - | `number` | The maximum supported trade size, in units of a market's base asset. |
  | `precision` | - | `number` | The "precision" of the market refers to the number of decimal places accounted for in prices and trade sizes. |
  | `durationSeconds` | - | `number` | The validity duration of qutoes for the market in seconds (`0` indicating no expiration). |
  
- JSON Example:
  
    ```json
    {
        "minSize": 0.001,
        "maxSize": 10,
        "precision": 6,
        "durationSeconds": 15
    }
    ```

### Schema: `Asset`

Defines information about an asset supported by a dealer implementation. 

- Fields:
  
  | Name | Schema | Required | JSON Type | Description |
  | :--- | :----- | :------- | :-------- | :---------- |
  | `ticker` | - | `Yes` |  `string` | Short-form name of the ERC-20 asset. SHOULD match the value provided by the contract. |
  | `name` | - | `Yes` | `string` | Long-form name of the ERC-20 asset. SHOULD match the value provided by the contract. |
  | `decimals` | - | `Yes` | `number` | The number of decimals used in the tokens user representation (see [EIP-20](https://eips.ethereum.org/EIPS/eip-20). |
  | `networkId` | - | `Yes` | `number` | The [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) network ID of the active Ethereum network (2).|
  | `assetData` | - | `Yes` | `string` | ABIv2 encoded asset data (including address) as used in the 0x system. |
  
- JSON Example:

    ```json
    {
        "ticker": "DAI",
        "name": "DAI Stablecoin (v1.0)",
        "decimals": 18,
        "networkId": 1,
        "assetData": "0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359",
    }
    ```

### Schema: `Market`

Defines a market: a trading venue that supports a base asset, and at least one quote asset (1).

- Fields:
  
  | Name | Schema | Required | JSON Type | Description |
  | :--- | :----- | :------- | :-------- | :---------- |
  | `baseAssetTicker` | - | `Yes` | `string` | The shorthand ticker of the markets base asset. |
  | `quoteAssetTickers` | - | `Yes` | `string[]` | An array of shorthand tickers for which quote assets are supported.
  | `tradeInfo` | `TradeInfo` | `Yes` | `object` | Information about trade settlement and execution for this market (gas price, etc.). |
  | `quoteInfo` | `QuoteInfo` | `Yes` | `object` | Information about quotes provided on this market (max/min size, precision, etc.). |
  | `metadata` | - | `No` | `object` | Optional and implementation-specific key-value pairs for additional market metadata. |
  
- JSON Example:
  
    ```json
    {
        "baseAssetTicker": "WETH",
        "quoteAssetTickers": [ "DAI", "USDC", "MKR", "ZRX" ],
        "tradeInfo": {
            "networkId": 1,
            "gasLimit": 210000,
            "gasPriceWei": 12000000000,
        },
        "quoteInfo": {
            "minSize": 0.001,
            "maxSize": 10,
            "precision": 6,
            "durationSeconds": 15
        },
        "metadata": {}
    }
    ```

### Schema: `Quote`

Defines a price quote from a dealer for a given base and quote asset, and other quote parameters.

- Fields:
  
  | Name | Schema | Required | JSON Type | Description |
  | :--- | :----- | :------- | :-------- | :---------- |
  | `baseAssetTicker` | - | `Yes` | `string` | The shorthand ticker of the markets base asset. |
  | `quoteAssetTickers` | - | `Yes` | `string[]` | An array of shorthand tickers for which quote assets are supported.
  | `tradeInfo` | `TradeInfo` | `Yes` | `object` | Information about trade settlement and execution for this market (gas price, etc.). |
  | `quoteInfo` | `QuoteInfo` | `Yes` | `object` | Information about quotes provided on this market (max/min size, precision, etc.). |
  | `metadata` | - | `No` | `object` | Optional and implementation-specific key-value pairs for additional market metadata. |
  
- JSON Example:
  
    ```json
    {
        "baseAssetTicker": "WETH",
        "quoteAssetTickers": [ "DAI", "USDC", "MKR", "ZRX" ],
        "tradeInfo": {
            "networkId": 1,
            "gasLimit": 210000,
            "gasPriceWei": 12000000000,
        },
        "quoteInfo": {
            "minSize": 0.001,
            "maxSize": 10,
            "precision": 6,
            "durationSeconds": 15
        },
        "metadata": {}
    }
    ```

## Methods

### Method: `dealer_authStatus`

Get dealer-specific authorization status for a given taker address. 

If the dealer is operating a blacklist or whitelist, this endpoint can inform the client why (or why not) they are able to trade.

Implementation MAY use arbitrary mechanisms to determine a taker's authorization, or place no restriction upon takers.

- **Request fields:**

    | Index | Name | JSON Type | Required | Default | Description |
    | :---- | :--- | :-------- | :------- | :------ | :---------- |
    | `0` | `takerAddress` | String | `Yes` | - | The 20-byte Ethereum address of the taker to check authorization for. |

- **Response fields:**

    | Index | Name | JSON Type | Schema | Description |
    | :---- | :--- | :-------- | :----- | :---------- |
    | `0` | `authorized` | Boolean | - | True or false, indicating the taker's authorization based on the implementation's configured access control mode (e.g. whitelist, blacklist, etc.). |
    | `1` | `reason` | String | - | A code or human-readable justification for the authorization status. |

- **Errors:**
   
   | Code | Description | Notes |
   | :--- | :---------- | :---- |
   | `-42001` | Invalid taker address. | Returned when an address is invalid or missing. |

- **Example request bodies:**

    ```json
    {
        "takerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459"
    }
    ```
    ```json
    [
        "0xcefc94f1c0a0be7ad47c7fd961197738fc233459"
    ]
    ```

- **Example response bodies:**

    ```json
    {
        "authorized": false,
        "reason": "BLACKLISTED"
    }
    ```
    ```json
    [
        false,
        "BLACKLISTED"
    ]
    ```

### Method: `dealer_getAssets`

Fetch information about currently supported ERC-20 assets.

This method, with no parameters, MUST return a paginated array of all supported assets. 

All parameters to this method (with the exception of `page` and `perPage`) act as filter parameters, returning only results that match all specified parameters.

This method MUST return an empty array if no results match the query. Implementations MAY return an error (`-32002`) if conflicting query parameters are provided. 

- **Request fields:**

    | Index | Name | JSON Type | Required | Default | Description |
    | :---- | :--- | :-------- | :------- | :------ | :---------- |
    | `0` | `address` | String | `No` | `null` | Match only assets with this address. MUST return only one result. |
    | `1` | `ticker` | String | `No` | `null` | Match only assets with this ticker. MUST return only one result. |
    | `2` | `assetData` | String | `No` | `null` | Match only assets with this asset data. MUST return only one result. |
    | `3` | `networkId` | Number | `No` | `1` | Only match assets with this network ID. |
    | `4` | `page` | Number | `No` | `0` | See [pagination.](#pagination) |
    | `5` | `perPage` | Number | `No` | Impl. specific | See [pagination.](#pagination) |

- **Response fields:**

    | Index | Name | JSON Type | Schema | Description |
    | :---- | :--- | :-------- | :----- | :---------- |
    | `0` | `assets` | Array | `[]Asset`| The array of asset results that match the request parameters. |
    | `1` | `items` | Number | - | The number of results that matched the request (MAY be 0). |
    | `2` | `page` | Number | - | The page index of the result (MUST match request). |
    | `3` | `perPage` | Number | - | The array of asset results that match the request parameters. |

- **Errors:**
   
   | Code | Description | Notes |
   | :--- | :---------- | :---- |
   | `-42002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested. |
   | `-42003` | Invalid asset address. | Returned when an invalid Ethereum address is provided. |
   | `-42004` | Invalid asset data. | Returned when malformed ABIv2 asset data is included in a request. |

- **Example request bodies:**

    ```json
    {
        "page": 0,
        "perPage": 2,
        "networkId": 1
    }
    ```
    ```json
    [
        null,
        null,
        1,
        0,
        2
    ]
    ```

- **Example response bodies:**

    ```json
    {
        "page": 0,
        "perPage": 2,
        "items": 2,
        "assets": [
            {
                "ticker": "DAI",
                "name": "DAI Stablecoin (v1.0)",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359",
            },
            {
                "ticker": "WETH",
                "name": "Wrapped Ether",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
            },
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
                "assetData": "0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359",
            },
            {
                "ticker": "WETH",
                "name": "Wrapped Ether",
                "decimals": 18,
                "networkId": 1,
                "assetData": "0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
            },
        ],
        2,
        0,
        2
    ]
    ```

### Method: `dealer_getMarkets`

Fetch information about currently supported [markets](#schema-market).

This method, with no parameters, MUST return a paginated array of all supported markets. 

All parameters to this method (with the exception of `page` and `perPage`) act as filter parameters, returning only results that match all specified parameters.

This method MUST return an empty array if no results match the query. Implementations MAY return an error (`-32002`) if conflicting query parameters are provided. 

- **Request fields:**

    | Index | Name | JSON Type | Required | Default | Description |
    | :---- | :--- | :-------- | :------- | :------ | :---------- |
    | `0` | `baseAssetTicker` | String | `No` | `null` | Match only markets with this base asset. |
    | `1` | `quoteAssetTicker` | String | `No` | `null` | Match only markets that support this quote asset ticker. |
    | `2` | `networkId` | Number | `No` | `1` | Match only markets supporting this network. |
    | `3` | `page` | Number | `No` | `0` | See [pagination.](#pagination) |
    | `4` | `perPage` | Number | `No` | Impl. specific | See [pagination.](#pagination) |

- **Response fields:**

    | Index | Name | JSON Type | Schema | Description |
    | :---- | :--- | :-------- | :----- | :---------- |
    | `0` | `markets` | Array | Market | The array of market results that match the request parameters. |
    | `1` | `items` | Number | - | The number of results that matched the request (MAY be 0). |
    | `2` | `page` | Number | - | The page index of the result (MUST match request). |
    | `3` | `perPage` | Number | - | The array of asset results that match the request parameters. |

- **Errors:**
   
   | Code | Description | Notes |
   | :--- | :---------- | :---- |
   | `-42002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested. |

- **Example request bodies:**

    ```json
    {
        "baseAssetTicker": "WETH",
        "networkId": 1,
        "page": 0,
        "perPage": 2
    }
    ```
    ```json
    [
        "WETH",
        null,
        1,
        0,
        2
    ]
    ```

- **Example response bodies:**

    ```json
    {
        "page": 0,
        "perPage": 2,
        "items": 2,
        "markets": [
            {
                "baseAssetTicker": "WETH",
                "quoteAssetTickers": [ "DAI", "MKR", "ZRX" ],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPriceWei": 12000000000,
                },
                "quoteInfo": {
                    "minSize": 0.001,
                    "maxSize": 10,
                    "precision": 18,
                    "durationSeconds": 15
                }
            },
            {
                "baseAssetTicker": "WETH",
                "quoteAssetTickers": [ "USDC" ],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPriceWei": 12000000000,
                },
                "quoteInfo": {
                    "minSize": 0.001,
                    "maxSize": 10,
                    "precision": 6,
                    "durationSeconds": 15
                }
            },
        ]
    }
    ```
    ```json
    [
        [
            {
                "baseAssetTicker": "WETH",
                "quoteAssetTickers": [ "DAI", "MKR", "ZRX" ],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPriceWei": 12000000000,
                },
                "quoteInfo": {
                    "minSize": 0.001,
                    "maxSize": 10,
                    "precision": 18,
                    "durationSeconds": 15
                }
            },
            {
                "baseAssetTicker": "WETH",
                "quoteAssetTickers": [ "USDC" ],
                "tradeInfo": {
                    "networkId": 1,
                    "gasLimit": 210000,
                    "gasPriceWei": 12000000000,
                },
                "quoteInfo": {
                    "minSize": 0.001,
                    "maxSize": 10,
                    "precision": 6,
                    "durationSeconds": 15
                }
            },
        ],
        2,
        0,
        2
    ]
    ```

### Method: `dealer_fetchQuote`

Primary method for requesting a quote from the dealer. Be sure to see the [quotes section.](#quotes)

To request a quote, a client MUST specify the base and quote asset (by ticker) and either a quote size or a base size.

Implementations MAY choose to support arbitrary swap quotes or simply return the corresponding error code (`-42008` or `-42009`) if the quote requested by the trader is unsupported.

Clients MUST leave at least one size field (either `baseAssetSize` or `quoteAssetSize`) as `null` or not specified that the dealer will fill-in to provide the price quote.

- **Request fields:**

    | Index | Name | JSON Type | Required | Default | Description |
    | :---- | :--- | :-------- | :------- | :------ | :---------- |
    | `0` | `baseAssetTicker` | String | `Yes` | - | Specify the base asset of the market and price calculation. |
    | `1` | `quoteAssetTicker` | String | `Yes` | - | Specify the quote asset of the market the asset used for the price. |
    | `2` | `baseAssetSize` | Number | `No` | `null` | Client MUST specify either this or `quoteAssetSize`. |
    | `3` | `quoteAssetSize` | Number | `No` | `null` | Client MUST specify either this or `baseAssetSize`. |
    | `4` | `side` | String | `Yes` | - | Specify if the client wants to buy (`"B"`) or sell (`"S"`) the base asset. |
    | `5` | `takerAddress` | String | `No` | (3) | The address of the taker that will fill the requested quote (see 3). |
    | `6` | `priceOnly` | Boolean | `No` | `false` | If `false`, the quote MUST include a signed 0x order for the offer (5). |

- **Response fields:**

    | Index | Name | JSON Type | Schema | Description |
    | :---- | :--- | :-------- | :----- | :---------- |
    | `0` | `quoteId`  | String | [UUID](#schema-uuid) | A UUID (v4) that MUST correspond to this offer only. |
    | `1` | `baseAssetTicker` | String | [Ticker](#schema-ticker) | Shorthand ticker of the quote's base asset (see [quotes](#quotes)). |
    | `2` | `quoteAssetTicker` | String | [Ticker](#schema-ticker) | Shorthand ticker of the quote's quote asset (see [quotes](#quotes)). |
    | `3` | `baseAssetSize` | Number | - | The quote's base asset size (see [quotes](#quotes)). |
    | `4` | `quoteAssetSize` | Number | - | The quote's quote asset size (see [quotes](#quotes)). |
    | `5` | `side` | String | [Side](#schema-side) | The quote "direction" from the client perspective (ONLY "B" or "S"). |
    | `6` | `price` | Number | - | The offer's price in units of the quote asset per base asset (4). |
    | `7` | `validUntil` | Number | [Time](#schema-time) | The UNIX timestamp after which the quote will be rejected for settlement. |
    | `8` | `orderHash` | String | - | The 0x-specific order hash, as defined in the [v3 specification](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#hashing-an-order).
    | `9` | `order` | Object | [Order](#schema-order) | The dealer-signed [0x order](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#orders) that corresponds to this offer.

- **Errors:**
   
   | Code | Description | Notes |
   | :--- | :---------- | :---- |
   | `-42005` | Two size requests. | Occurs when a client specifies `baseAssetSize` and `quoteAssetSize`. |
   | `-42006` | Size too precise. | Occurs when a client requests a quote size with too many decimals. |
   | `-42007` | Taker not authorized. | Occurs when the taker's address is not authorized for trading. |
   | `-42008` | Invalid side. | Available for implementations to indicate |
   | `-42008` | Temporary restriction. | Available for implementations to indicate taker-specific temporary restrictions. |
   | `-42008` | Unsupported market. | Occurs when the specified market (quote and base pair) is not supported. |
   | `-42009` | Unsupported quote asset for market. | Available for implementations to indicate lack of support for arbitrary swaps. |
   | `-42009` | Quote too large. | Occurs when a quote would exceed the market maximum or the dealer's balance. |
   | `-42010` | Quote too small. | Occurs when a quote would be smaller than the market's minimum size. |
   | `-42011` | Quote unavailable at this time. | Reserved for various states where dealers may not be serving quotes. |

- **Example request bodies:**

    ```json
    {
        "baseAssetTicker": "ZRX",
        "quoteAssetTicker": "DAI",
        "baseAssetSize": 1400,
        "side": "B",
        "takerAddress": "0xcefc94F1C0a0bE7aD47c7fD961197738fC233459"
    }
    ```
    ```json
    [
        "ZRX",
        "WETH",
        100,
        null,
        "B",
        "0xcefc94F1C0a0bE7aD47c7fD961197738fC233459"
    ]
    ```

- **Example response bodies:**

    ```json
    {
        "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "baseAssetTicker": "ZRX",
        "quoteAssetTicker": "WETH",
        "baseAssetSize": 100,
        "quoteAssetSize": 0.3,
        "side": "B",
        "price": 0.003,
        "validUntil": 1573775025,
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
        }
    }
    ```
    ```json
    [
        "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "ZRX",
        "WETH",
        100,
        0.3,
        "B",
        0.003,
        1573775025,
        "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
        {
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
        }
    ]
    ```

### Method: `dealer_submitFill`

Fetch an order-only object, provided a known (valid) quote UUID.

Can be used alongside the `fetchQuote` and `fetchSwapQuote` methods with a `priceOnly` filter for a 2-step quote process.


### Method: `dealer_getQuoteById`

Fetch a quote and signed order given a (valid) previously provided quote UUID.

Useful for operational flows where a client requests several price-only quotes, then chooses one to request a fill for.

## Appendix

1. This definition of a market makes an intentional departure from conventional currency-pair based markets in which their is a single quote asset and a single base asset.
1. If the dealer is operating on the main Ethereum network, they MUST treat the `networkID` of `1` as the Ethereum mainnet, as specified in EIP-155. Private and test networks may use any network ID, but SHOULD use conventions established by public test networks (e.g. Ropsten is 3).
1. The default value SHOULD be the null address (20 null bytes) represented as a hex string. Implementations MAY require takers to specify a `takerAddress`.
1. Quote sizes and prices within quotes when represented using an asset's user representation MUST use the level of precision specified in the corresponding market.
1. If a client requests a quote without an order, implementations MAY allow the client to get the order at a later time with a separate method. Quotes indicated as `priceOnly` can be seen as traders checking if a dealer's prices are favorable at a given time for a certain market and trade size.

