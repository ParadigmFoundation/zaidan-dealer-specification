# Dealer JSONRPC v1.0

Definitions and specification for the Dealer [JSONRPC (2.0)](https://www.jsonrpc.org/specification) API. This document specifies the first public Dealer JSONRPC version, `v1.0`.

Methods under the `dealer` namespace (with the `dealer_` prefix) comprise the public dealer API. Implementations MAY provide private administrative functionality through methods under a distinct namespace.

The Dealer JSONRPC can be served over WebSockets, HTTP POST, or HTTP GET, or other transports at the discretion of implementers (see notes).

### Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/rfc/rfc2119.txt).

## Basic requirements
- Implementations MUST implement all methods under the `dealer` namespace (see [Methods](#methods)).
- All supported assets MUST each have a unique string identifier called a "ticker" (e.g. DAI, ZRX, WETH).
- Implementations MUST use the canonical 0x v3 addresses for the active Ethereum network (not yet deployed).
- Implementations MUST only support ERC-20 assets (subject to change in future major API versions).
- Implementations MUST use arbitrary precision representations for integers.
- Implementations MAY support batch requests, in accordance with the JSONRPC 2.0 specification.
- Floating point numbers SHOULD be of fixed precision according to the precision of the parent market, if used at all.
- Pricing calculations SHOULD take place with the integer base unit representations of supported assets, and be represented as decimals only in the public API.
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
- Implementations SHOULD support Ether (ETH) trading.
	- Implementations that support ETH trading MUST do so via the canonical wrapped Ether (WETH) contract for the active network. 


## Encoding notes
- Unless otherwise specified, values MUST use their equivalent JSON types (as shown in the examples).
- Binary data (EVM `bytes`, etc.) MUST be encoded as `0x`-prefixed all-lowercase hex-encoded strings in the JSONRPC. 
- Ethereum addresses MUST be encoded as all other binary data (`0x`-prefix, all lowercase, no EIP-55 checksum).

## Pagination

Paginated methods MUST implement pagination in accordance with this section.

Paginated methods MUST include two additional parameters in the `params` array or object (where `n` is the number of parameters the method accepts):

| Index | Name | JSON Type | Required | Default | Description |
| :---- | :--- | :-------- | :------- | :------ | :---------- |
| `n - 2` | `page` | `number` | `No` | `0` | The page number of the paginated results, where there are `perPage` items on each page. Must have a default value of `0` (the first page). |
| `n - 1` | `perPage` | `number` | `No` | Implementation specific | The number of items to include on each page (used by the server to calculate which results to include). |

## Errors

Implementations MUST support the reserved error codes specified in [the JSONRPC 2.0](https://www.jsonrpc.org/specification#error_object) specification, in addition to any method-specific errors.

Implementations MAY add their own error codes and messages, and may use the `data` field provided by the JSONRPC specification.

Implementations MUST only adhere to the defined codes for the defined scenarios. The exact message text MAY not match the specification exactly.

## Schemas

Schematics and data structures used in the public API (JSON shown). 

Fields indicated `Yes` in the `Required` column for each scheme MUST be implemented, while fields indicated `No` MAY be omitted.

All schemas in this section with at least one required field MUST be implemented.

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

- **Request fields:**

    | Index | Name | JSON Type | Required | Default | Description |
    | :---- | :--- | :-------- | :------- | :------ | :---------- |
    | `0` | `takerAddress` | `string` | `Yes` | - | The 20-byte Ethereum address of the taker to check authorization for. |

- **Response fields:**

    | Index | Name | JSON Type | Description |
    | :---- | :--- | :-------- | :---------- |
    | `0` | `authorized` | `boolean` | True or false, indicating the taker's authorization based on the implementation's configured access control mode (e.g. whitelist, blacklist, etc.). |
    | `1` | `reason` | `string` | A code or human-readable justification for the authorization status. |

- **Errors:**
   
   | Code | Description | Notes |
   | :--- | :---------- | :---- |
   | `-32001` | Invalid taker address. | Returned when an address is invalid or missing. |

- **Example requests:**

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

- **Example responses:**

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
   | `-32002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested. |
   | `-32003` | Invalid asset address. | Returned when an invalid Ethereum address is provided. |
   | `-32004` | Invalid asset data. | Returned when malformed ABIv2 asset data is included in a request. |

- **Example request:**

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

- **Example response:**

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
    | `0` | `markets` | Array | `[]Market`| The array of market results that match the request parameters. |
    | `1` | `items` | Number | - | The number of results that matched the request (MAY be 0). |
    | `2` | `page` | Number | - | The page index of the result (MUST match request). |
    | `3` | `perPage` | Number | - | The array of asset results that match the request parameters. |

- **Errors:**
   
   | Code | Description | Notes |
   | :--- | :---------- | :---- |
   | `-32002` | Invalid filter selection. | Returned when conflicting or incompatible filters are requested. |

- **Example request:**

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

- **Example response:**

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

Primary method for requesting a quote from the dealer.


### Method: `dealer_submitFill`

Fetch an order-only object, provided a known (valid) quote UUID.

Can be used alongside the `fetchQuote` and `fetchSwapQuote` methods with a `priceOnly` filter for a 2-step quote process.


### Method: `dealer_getQuoteById`

Fetch a quote and signed order given a (valid) previously provided quote UUID.

Useful for operational flows where a client requests several price-only quotes, then chooses one to request a fill for.

## Appendix

1. This definition of a market makes an intentional departure from conventional currency-pair based markets in which their is a single quote asset and a single base asset.
1. If the dealer is operating on the main Ethereum network, they MUST treat the `networkID` of `1` as the Ethereum mainnet, as specified in EIP-155. Private and test networks may use any network ID, but SHOULD use conventions established by public test networks (e.g. Ropsten, 3


