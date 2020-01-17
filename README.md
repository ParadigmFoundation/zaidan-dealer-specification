## Notice: in development

This section will be removed when the public interface has stabilized.

# Dealer JSONRPC v1.0

Definitions and specification for the Dealer [JSONRPC (2.0)](https://www.jsonrpc.org/specification) API. This document specifies the first public Dealer JSONRPC version, `v1.0`.

Methods under the `dealer` namespace (with the `dealer_` prefix) comprise the public dealer API. Implementations MAY provide private administrative functionality through methods under a distinct namespace.

Methods under the `feed` namespace (with the `feed_` prefix) specify an alternative/additional interaction model for trading with dealer implementations, and MAY be omitted by implementers.

The Dealer JSONRPC can be served over WebSockets, HTTP POST, or HTTP GET, or other transports at the discretion of implementers (see notes).

Be sure to see important notes and resources in [the appendix.](#appendix)

### Conventions

-   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/rfc/rfc2119.txt).
-   The word "implementation" is used to refer to any system that implements or provides the Dealer JSONRPC.
-   In accordance with the JSONRPC 2.0 spec, JSON types (String, Number, Object, and Array) are capitalized to differentiate between usage of the word and the JSON type.
-   The syntax "Array\<T>" is used to indicate an Array of type T (for JSON values) or an Array of schema T (for custom schemas defined in this document).
-   The term "dealer" refers to an entity operating a system that implements the Dealer JSONRPC specified in this document.

## Contents

-   [Requirements](#requirements)
-   [Encoding](#encoding)
-   [Quotes](#quotes)
-   [Quote feeds](#quote-feeds)
-   [Pagination](#pagination)
-   [Errors](#errors)
-   [Dealer methods](#dealer-methods)
    -   [AuthStatus](#method-dealer_authstatus)
    -   [GetMarkets](#method-dealer_getmarkets)
    -   [GetQuote](#method-dealer_getquote)
    -   [SubmitFill](#method-dealer_submitfill)
    -   [Time](#method-dealer_time)
-   [Feed methods](#feed-methods)
    -   [Subscribe](#method-feed_subscribe)
    -   [GetQuoteFromStub](#method-feed_getquotefromstub)
    -   [Unsubscribe](#method-feed_unsubscribe)
-   [Schemas](#schemas)
    -   [Ticker](#schema-ticker)
    -   [Time](#schema-time)
    -   [UUID](#schema-uuid)
    -   [TradeInfo](#schema-tradeinfo)
    -   [QuoteInfo](#schema-quoteinfo)
    -   [ValidityParameter](#schema-validityparameter)
    -   [Market](#schema-market)
    -   [Order](#schema-order)
    -   [Quote](#schema-quote)
    -   [QuoteStub](#schema-quotestub)
-   [Appendix](#appendix)
    -   [Notes](#notes)
    -   [Important resources](#important-resources)
    -   [Error codes](#error-codes)

## Requirements

In addition to notices in each section, each of the following must be true in order for an implementation to be considered in adherence with the specification.

These requirements are intended to motivate strong guarantees of compatibility between clients and servers and ensure maximum levels of safety for the operators of each: traders and dealers that implement this API.

-   Implementations MUST implement all methods under the `dealer` namespace (see [Methods](#methods)).
-   Implementations MUST implement all public object schematics necessary to complete the public `dealer` methods (see [Schemas](#schemas)).
-   Implementations MUST use the [canonical 0x v3 addresses](https://github.com/0xProject/0x-monorepo/blob/development/packages/contract-addresses/addresses.json) for the active Ethereum network.
-   Implementations MUST support asset settlement according to relevant sections in this document and [ZEIP-18](https://github.com/0xProject/ZEIPs/blob/master/ZEIPS/ZEIP-18.md).
-   Implementations MUST only support ERC-20 assets (subject to change in future major API versions).
-   Implementations MUST only reference assets by their ERC-20 contract's deployed address on the active Ethereum network.
-   Implementations MUST display asset amounts in base units of the corresponding assets; there MUST NOT be decimal asset amounts (see [encoding](#encoding)).
-   Implementations MUST use arbitrary precision (or sufficiently precise fixed-precision) representations for integers.
-   Implementations MUST encode all values denoting asset amounts as JSON Strings in the public API to preserve precision.
-   Implementations MUST NOT use floating points in the public API (including timestamps, which MUST be the integer UNIX time in milliseconds).
-   Implementations MUST use Arrays for return values and request parameters (in accordance with the JSONRPC specification).
-   Implementations MAY support batch requests, in accordance with the JSONRPC 2.0 specification.
-   Implementations MUST use Array for return values and parameters (in accordance with the JSONRPC specification).
-   Implementations SHOULD support Ether (ETH) trading, and if so, MUST do so via the canonical [WETH contract](https://etherscan.io/address/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2) for the active network.
-   Even though the specification allows markets to individual specify the active [chain ID](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md), implementations MUST NOT operate on more than one Ethereum network (identified by the chain ID) at a time (see [note 2](#notes)).
-   Implementations MAY require that quote requests include the potential taker's address.
    -   The address provided by the taker MAY be used to restrict the `takerAddress` of the quotes underlying signed 0x order.
    -   Implementations MAY record and use the address provided by the taker to influence pricing or to restrict quote provision for blacklisted takers.

## Encoding

-   All asset amount values, including gas prices and gas limits MUST be integer values (as Strings) in their base units.
-   Binary data (EVM `bytes` type, etc.) MUST be encoded as `0x`-prefixed all-lowercase hex-encoded strings in the JSONRPC.
-   Ethereum addresses MUST be encoded as all other binary data (`0x`-prefix, all lowercase, no EIP-55 checksum).
-   Unless otherwise specified, values MUST use their equivalent JSON types (as shown in the examples).

## Quotes

Dealer implementations use the [0x contract system](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md) for asset settlement, and as such, the trading interface provided by the Dealer API is designed to foster helpful abstractions over the underlying settlement system.

For this reason, instead of providing a currency pair (base/quote and price denominated markets) interface for quote requests,
traders are able to specify the `makerAsset`, `takerAsset`, and one of either `makerAssetSize` or `takerAssetSize` (the other is provided by the dealer as the quote). Price can then be calculated in terms of either asset at higher levels depending on the use case. Similarly, all asset sizes included in requests and responses from the dealer MUST be in integer base units.

Because there is no concept of a base or quote asset, quotes include no notion of price. Instead allowing clients to calculate the price in terms of either asset.

Implementations MAY choose what types of markets to support, to replicate more conventional trading systems.

In the example below, the mainnet (chain ID 1) address for the following assets are used:

-   DAI Stablecoin (DAI): `0x6b175474e89094c44da98b954eedeac495271d0f`
-   0x Protocol Token (ZRX): `0xe41d2489571d322189246dafa5ebde1f4699f498`

Consider the following requests (syntax: `[ MAKER_ASSET, TAKER_ASSET, MAKER_SIZE, TAKER_SIZE ]`).

```json
[
    [
        "0x6b175474e89094c44da98b954eedeac495271d0f",
        "0xe41d2489571d322189246dafa5ebde1f4699f498",
        null,
        "100000000000000000000"
    ],
    [
        "0xe41d2489571d322189246dafa5ebde1f4699f498",
        "0x6b175474e89094c44da98b954eedeac495271d0f",
        "100000000000000000000",
        null
    ],
    [
        "0xe41d2489571d322189246dafa5ebde1f4699f498",
        "0x6b175474e89094c44da98b954eedeac495271d0f",
        null,
        "10000000000000000000"
    ],
    [
        "0x6b175474e89094c44da98b954eedeac495271d0f",
        "0xe41d2489571d322189246dafa5ebde1f4699f498",
        "10000000000000000000",
        null
    ]
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

## Quote feeds

Supplemental to the standard quote request mechanism described in this spec (`dealer_getQuote`), implementations MAY also provide a public "quote feed" subscription mechanism.

A quote feed provides a subscription to "quote stubs" for a given set of markets, where a stub indicates an available quantity of the maker or taker asset, and provides a range of prices between which a quote solicited by the a trader will lie. In other words, each stub represents a soft price commitment by a dealer implementation to a specific quantity of an asset. See the [quote stub schema](#schema-quotestub), the [subscription method](#method-feed_subscribe), and [the method used to fetch a quote](#method-feed_getquotefromstub) for a given stub. Conceptually, the quote feed can be considered a limit order book with soft price bounds.

Quote feeds provide takers an additional means for takers to gather price information from a dealer. Notably, the quote feed allows a taker to remain anonymous during the dealer's initial price setting process. More formally, the quote feed provides quote stubs that represent upper and lower cost limits for a specific quantity of an asset. In the context of an order book, these cost limits could be considered bounded price levels.

When a potential taker fetches a quote corresponding to particular quote stub, they will receive an exact cost/price for the trade size specified by the taker (in units of either the maker asset or the taker asset). This value will fall somewhere in the range defined in the quote stub. The value's positioning in the range will depend on a taker's specific relationship to the dealer. Thus, this scheme also provides takers an additional metric on which to evaluate a dealer.

In addition to providing direct subscription to a quote feed via the [`feed_subscribe`](#method-feed_subscribe), dealers MAY choose to post quote stubs (either all, or a subset) to arbitrary venues. This includes [IPFS](https://ipfs.io/), a conventional DB/REST API, or perhaps eventually, a peer-to-peer liquidity sharing network such as [0x Mesh](https://github.com/0xProject/0x-mesh) (arbitrary messages are not current supported by mesh).

The number of quote stubs alive at a current time SHOULD be equal to:

```
(# of quote stubs) = 2 * (# of quantity levels per pair) * (# of pairs)
```

**Note:** `(# of pairs)` for dealers that support arbitrary swap functionality is equal to `n * (n - 1)` where `n` is the number of supported assets. For dealers that only support markets between unique base assets and a standard quote currency `(# of pairs)` is equal to `n`.

## Pagination

Paginated methods MUST implement pagination in accordance with this section. If `page` is not specified, the default MUST be 0. The value for `perPage` MAY be implementation specific.

Paginated methods MUST include two additional parameters in the `params` array (where `n` is the number of request parameters). The response parameters MUST match the table below.

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

## Dealer methods

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

    | Code     | Description                 | Notes                                                                                |
    | :------- | :-------------------------- | :----------------------------------------------------------------------------------- |
    | `-42001` | Invalid taker address.      | Returned when an address is invalid or missing.                                      |
    | `-42024` | Request rate limit reached. | Available to indicate a implementation-specific request rate limit has been reached. |

-   **Example request body:**

    ```json
    ["0xcefc94f1c0a0be7ad47c7fd961197738fc233459"]
    ```

-   **Example response body:**

    ```json
    [false, "BLACKLISTED"]
    ```

### Method: `dealer_getMarkets`

Fetch information about currently supported [markets](#schema-market). This method is [paginated.](#pagination)

This method, with no parameters, MUST return a paginated array of all supported markets.

All parameters to this method (with the exception of `page` and `perPage`) act as filter parameters, returning only results that match all specified parameters.

Implementations MUST return an empty array for `records` if no results match the query.

Implementations MAY return an error (e.g. `-42002`) if conflicting query parameters are provided.

-   **Request fields:**

    | Index | Name                | JSON Type | Required | Default        | Description                                                    |
    | :---- | :------------------ | :-------- | :------- | :------------- | :------------------------------------------------------------- |
    | `0`   | `makerAssetAddress` | String    | `No`     | `null`         | Match only markets with this maker asset.                      |
    | `1`   | `takerAssetAddress` | String    | `No`     | `null`         | Match only markets that support this taker asset ticker.       |
    | `2`   | `marketId`          | String    | `No`     | `null`         | Match only the market with this ID. MUST match 0 or 1 markets. |
    | `3`   | `page`              | Number    | `No`     | `0`            | See [pagination.](#pagination)                                 |
    | `4`   | `perPage`           | Number    | `No`     | Impl. specific | See [pagination.](#pagination)                                 |

-   **Response fields:**

    | Index | Name      | JSON Type | Schema         | Description                                                    |
    | :---- | :-------- | :-------- | :------------- | :------------------------------------------------------------- |
    | `0`   | `records` | Array     | Array\<Market> | The array of market results that match the request parameters. |
    | `1`   | `total`   | Number    | -              | The number of results that matched the request (MAY be 0).     |
    | `2`   | `page`    | Number    | -              | The page index of the result (MUST match request).             |
    | `3`   | `perPage` | Number    | -              | The array of asset results that match the request parameters.  |

-   **Errors:**

    | Code     | Description                 | Notes                                                                                |
    | :------- | :-------------------------- | :----------------------------------------------------------------------------------- |
    | `-42002` | Invalid filter selection.   | Returned when conflicting or incompatible filters are requested.                     |
    | `-42024` | Request rate limit reached. | Available to indicate a implementation-specific request rate limit has been reached. |

-   **Example request body:**

    ```json
    ["WETH", null, null, 0, 2]
    ```

-   **Example response body:**

    ```json
    [
        [
            {
                "marketId": "16b59ee0-7e01-4994-9abe-0561aac8ad7c",
                "makerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "takerAssetAddresses": [
                    "0x6b175474e89094c44da98b954eedeac495271d0f",
                    "0x9f8f72aa9304c8b593d555f12ef6589cc3a579a2",
                    "0xe41d2489571d322189246dafa5ebde1f4699f498"
                ],
                "tradeInfo": {
                    "chainId": 1,
                    "gasLimit": "210000",
                    "gasPrice": "12000000000"
                },
                "quoteInfo": {
                    "minSize": "100000000000000",
                    "maxSize": "100000000000000000000"
                }
            },
            {
                "marketId": "87c0ee47-44c0-4ff0-ba68-6638c79c11dd",
                "makerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "takerAssetAddresses": ["0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48"],
                "tradeInfo": {
                    "chainId": 1,
                    "gasLimit": "210000",
                    "gasPrice": "12000000000"
                },
                "quoteInfo": {
                    "minSize": "100000000000000",
                    "maxSize": "100000000000000000000"
                }
            }
        ],
        2,
        0,
        2
    ]
    ```

### Method: `dealer_getQuote`

Primary method for requesting a quote from the dealer. Be sure to see the [quotes section.](#quotes)

To request a quote, a client MUST specify the maker and taker asset (by address) and either a maker size or a taker size (but not both).

Implementations SHOULD NOT allow traders to specify _both_ the maker and taker asset sizes, as this is highly illogical in most scenarios, and would amount to the trader setting the price.

Implementations MAY choose to support arbitrary swap quotes or simply return the corresponding error code (`-42008` or `-42009`) if the quote requested by the trader is unsupported.

Clients SHOULD leave at least one size field (either `makerAssetSize` or `takerAssetSize`) as `null` or not specified that the dealer will fill-in to provide the price quote.

-   **Request fields:**

    | Index | Name                | JSON Type | Required | Default           | Description                                                                                              |
    | :---- | :------------------ | :-------- | :------- | :---------------- | :------------------------------------------------------------------------------------------------------- |
    | `0`   | `makerAssetAddress` | String    | `Yes`    | -                 | Specify the maker asset of the quote (sent by the dealer).                                               |
    | `1`   | `takerAssetAddress` | String    | `Yes`    | -                 | Specify the taker asset of the quote (sent by the client).                                               |
    | `2`   | `makerAssetSize`    | Number    | `No`     | `null`            | Client MUST specify either this or `takerAssetSize`.                                                     |
    | `3`   | `takerAssetSize`    | Number    | `No`     | `null`            | Client MUST specify either this or `makerAssetSize`.                                                     |
    | `4`   | `takerAddress`      | String    | `No`     | (See [3](#notes)) | The address of the taker that will fill the requested quote (see 3).                                     |
    | `5`   | `includeOrder`      | Boolean   | `No`     | `true`            | If `true`, the quote MUST include a signed 0x order and 0x transaction data for the offer [(4)](#notes). |
    | `6`   | `extra`             | Object    | `No`     | `null`            | Optional extra structured data from the taker. MAY be omitted by implementations.                        |

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
    | `-42024` | Request rate limit reached.         | Available to indicate a implementation-specific request rate limit has been reached.       |

-   **Example request body:**

    ```json
    [
        "0xe41d2489571d322189246dafa5ebde1f4699f498",
        "0x6b175474e89094c44da98b954eedeac495271d0f",
        "1435000000000000000",
        null,
        "0xcefc94f1c0a0be7ad47c7fd961197738fc233459"
    ]
    ```

-   **Example response body:**

    ```json
    [
        {
            "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
            "makerAssetAddress": "0xe41d2489571d322189246dafa5ebde1f4699f498",
            "takerAssetAddress": "0x6b175474e89094c44da98b954eedeac495271d0f",
            "makerAssetSize": "1435000000000000000",
            "takerAssetSize": "300000000000000000",
            "expiration": 1573775025312,
            "serverTime": 1573775014231,
            "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
            "order": {
                "chainId": 1,
                "makerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
                "feeRecipientAddress": "0x0000000000000000000000000000000000000000",
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
            "chainId": 1,
            "gasLimit": "210000",
            "gasPrice": "12000000000"
        },
        null
    ]
    ```

### Method: `dealer_submitFill`

Submit a previously-fetched quote for settlement. Can be thought of as a "request-to-fill" a quote by a trader.

This method MUST support executing the signed 0x fill transaction from the client, according to [ZEIP-18.](https://github.com/0xProject/ZEIPs/issues/18)

Implementations MAY use the `validityParameters` from previously-submitted quotes to reject fills based on external parameters.

The order or 0x fill transaction data, associated hash, and the original quote/order MAY be stored by implementations and associated with the quote ID so the data received by the client can be additionally verified prior to submitting for execution.

-   **Request fields:**

    | Index | Name        | JSON Type | Required | Default | Description                                                            |
    | :---- | :---------- | :-------- | :------- | :------ | :--------------------------------------------------------------------- |
    | `0`   | `quoteId`   | String    | `Yes`    | -       | The ID of the original quote that is being submitted for settlement.   |
    | `1`   | `salt`      | String    | `Yes`    | -       | The salt used to generate the 0x fill transaction hash and signature.  |
    | `2`   | `signature` | String    | `Yes`    | -       | The taker's signature of the 0x fill transaction data.                 |
    | `3`   | `signer`    | String    | `Yes`    | -       | The address that signed the fill. SHOULD match original taker.         |
    | `4`   | `data`      | String    | `Yes`    | -       | The full 0x fill transaction call data.                                |
    | `5`   | `hash`      | String    | `Yes`    | -       | The salted hash of the 0x fill transaction that was signed.            |
    | `6`   | `gasPrice`  | String    | `Yes`    | -       | The gas price originally specified in the original quote (MUST match). |

-   **Response fields:**

    | Index | Name              | JSON Type | Schema               | Description                                                                 |
    | :---- | :---------------- | :-------- | :------------------- | :-------------------------------------------------------------------------- |
    | `0`   | `quoteId`         | String    | [UUID](#schema-uuid) | The UUID of the original quote that has been submitted for settlement.      |
    | `1`   | `transactionHash` | String    | -                    | The hash of the submitted 0x fill (Ethereum transaction hash)               |
    | `2`   | `submittedAt`     | Number    | [Time](#schema-time) | The UNIX timestamp (in milliseconds) the fill transaction was submitted at. |
    | `3`   | `extra`           | Object    | -                    | OPTIONAL implementation-specific relevant structured data.                  |

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
    | `-42024` | Request rate limit reached.   | Available to indicate a implementation-specific request rate limit has been reached.              |

-   **Example request body:**

    ```json
    [
        "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "49698000413078424200806561747824722300644271588299439950633438451205967129974",
        "0xd90ade56ae73626247516dfaa2ab8813a7938c20504376a3e52d25114367ef9b201c55d7f7eaa7301c0c8540ca3afbd02",
        "0x7df1567399d981562a81596e221d220fefd1ff9b",
        "0x9b44d5560000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000002ee5547f0900000000000000000000000000000000000000000000000000000000000000000320000000000000000000000000533014661bfcaf6f0431b2e406dc6590f02ad61d0000000000000000000000008c9ce34862490e3553f7743f308c28b3714a9b0a0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000533014661bfcaf6f0431b2e406dc6590f02ad61d00000000000000000000000000000000000000000000000017e18dacf40de0e4000000000000000000000000000000000000000000000000002ee5547f09000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005e2210dd487e984026bcacc3f16dfe4006d0a6005e9c14f7a56a2db1ad4fff5c852e873000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000220000000000000000000000000000000000000000000000000000000000000028000000000000000000000000000000000000000000000000000000000000002800000000000000000000000000000000000000000000000000000000000000024f47261b00000000000000000000000006b175474e89094c44da98b954eedeac495271d0f000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024f47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000014000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000421b23fd730519e9fd76a16302e5b71f0183187917f1cea4418852e8647b52de2e1360a212883c978c62bae031f6209b4f059a6b4609b8e64aa5ed31a4110163345d03000000000000000000000000000000000000000000000000000000000000",
        "0x558e9ce660003681cd12e2c00ce1c92cbd8ac3d0f2db118fd1d0bf8278dc2949",
        "12000000000"
    ]
    ```

-   **Example response body:**

    ```json
    [
        "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "0x6100529dedbf80435ba0896f3b1d96c441690c7e3c7f7be255aa7f6ee8a07b65",
        1574108114301
    ]
    ```

### Method: `dealer_time`

Fetch the current time from the dealer server as a UNIX timestamp (in milliseconds).

Optionally provide a time in the request (`clientTime`) to get the difference (useful for clock synchronization and estimating network latency).

-   **Request fields:**

    | Index | Name         | JSON Type | Required | Default | Description                                                           |
    | :---- | :----------- | :-------- | :------- | :------ | :-------------------------------------------------------------------- |
    | `0`   | `clientTime` | Number    | `No`     | `null`  | A UNIX timestamp from the client to get a difference in milliseconds. |

-   **Response fields:**

    | Index | Name   | JSON Type | Schema               | Description                                                                                                  |
    | :---- | :----- | :-------- | :------------------- | :----------------------------------------------------------------------------------------------------------- |
    | `0`   | `time` | Number    | [Time](#schema-time) | The UNIX timestamp of the dealer's clock at the time of request.                                             |
    | `1`   | `diff` | Number    | -                    | The difference between the dealer time and the client time in milliseconds. ONLY if `clientTime` in request. |

-   **Errors:**

    | Code     | Description                 | Notes                                                                                |
    | :------- | :-------------------------- | :----------------------------------------------------------------------------------- |
    | `-32603` | Internal error.             | Internal JSON-RPC error. MAY be used as generic internal error code.                 |
    | `-42024` | Request rate limit reached. | Available to indicate a implementation-specific request rate limit has been reached. |

-   **Example request body:**

    ```json
    [1574108764019]
    ```

-   **Example response body:**

    ```json
    [1574108764218, 1099]
    ```

## Feed methods

### Method: `feed_subscribe`

Allows the caller to subscribe to a feed of quote stubs for a set of maker and taker asset denominated markets for live-updating offers in the form of [`QuoteStub`](#schema-quote-stub) and [`StubUpdate`](#schema-stub-update) messages.

If no parameters after are provided, the subscription MUST include quote stubs for all markets that support [quote feeds.](#quote-feeds) Upon creation of a new subscription, a snapshot is provided to the subscriber of all active stubs that match their query filter. Subsequent messages sent in the subscription MUST contain only new stubs, and updates to existing stubs (including cancellations).

To subscribe to multiple markets with different maker and taker assets, multiple subscriptions SHOULD be used.

-   **Request fields:**

    | Index | Name                  | JSON Type      | Required | Default | Description                                              |
    | :---- | :-------------------- | :------------- | :------- | :------ | :------------------------------------------------------- |
    | `0`   | `makerAssetAddresses` | Array\<String> | `No`     | `[]`    | Subscribe to stubs from markets with these maker assets. |
    | `1`   | `takerAssetAddresses` | Array\<String> | `No`     | `[]`    | Subscribe to stubs from markets with these taker assets. |

-   **Response fields:**

    | Index | Name             | JSON Type      | Schema                                  | Description                                                                                    |
    | :---- | :--------------- | :------------- | :-------------------------------------- | :--------------------------------------------------------------------------------------------- |
    | `0`   | `subscriptionId` | String         | [UUID](#schema-uuid)                    | A UUID used to identify this subscription (and used to [cancel](#method-feed_unsubscribe) it). |
    | `1`   | `snapshot`       | Array<\Object> | Array\<[QuoteStub](#schema-quote-stub)> | A snapshot of all currently active quote stubs that match the subscription filter.             |

-   **Event fields:**

    | Index | Name             | JSON Type      | Schema                                   | Description                                                          |
    | :---- | :--------------- | :------------- | :--------------------------------------- | :------------------------------------------------------------------- |
    | `0`   | `subscriptionId` | String         | [UUID](#schema-uuid)                     | The UUID indicating which subscription this event is from.           |
    | `1`   | `stubs`          | Array\<Object> | Array\<[QuoteStub](#schema-quotestub)>   | New quote stubs that match the subscription filters.                 |
    | `2`   | `updates`        | Array\<Object> | Array\<[StubUpdate](#schema-stubupdate)> | Updates to existing quote stubs that match the subscription filters. |

-   **Errors:**

    | Code     | Description                 | Notes                                                                                       |
    | :------- | :-------------------------- | :------------------------------------------------------------------------------------------ |
    | `-32603` | Internal error.             | Internal JSON-RPC error. MAY be used as generic internal error code.                        |
    | `-42002` | Invalid filter selection.   | Returned when conflicting or incompatible filters are requested (e.g. no matching markets). |
    | `-42024` | Request rate limit reached. | Available to indicate a implementation-specific request rate limit has been reached.        |

-   **Example request body:**

    ```json
    [
        ["0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"],
        [
            "0x6b175474e89094c44da98b954eedeac495271d0f",
            "0xe41d2489571d322189246dafa5ebde1f4699f498",
            "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48"
        ]
    ]
    ```

-   **Example response body:**

    ```json
    [
        "426dbcf8-1d84-405c-978f-454beb1566b8",
        [
            {
                "stubId": "d7ed72f4-d486-4b04-bed6-d8f78e1936a1",
                "makerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "takerAssetAddress": "0x6b175474e89094c44da98b954eedeac495271d0f",
                "makerSizeLimit": "14000000000000000000",
                "takerSizeLimit": null,
                "makerPriceBand": null,
                "takerPriceBand": [0.0078, 0.0082]
            }
        ]
    ]
    ```

-   **Example event:**

    ```json
    [
        "426dbcf8-1d84-405c-978f-454beb1566b8",
        [
            {
                "stubId": "1e342bd7-6dca-4cbe-9a91-7466e595206c",
                "makerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "takerAssetAddress": "0xe41d2489571d322189246dafa5ebde1f4699f498",
                "makerSizeLimit": "10000000000000000000",
                "takerSizeLimit": null,
                "makerPriceBand": null,
                "takerPriceBand": [5310, 5790]
            },
            {
                "stubId": "3efff541-135a-4be8-9da7-f310d5338b1c",
                "makerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "takerAssetAddress": "0x6b175474e89094c44da98b954eedeac495271d0f",
                "makerSizeLimit": "10000000000000000000",
                "takerSizeLimit": null,
                "makerPriceBand": null,
                "takerPriceBand": [120.4, 135]
            }
        ],
        [
            {
                "stubId": "d7ed72f4-d486-4b04-bed6-d8f78e1936a1",
                "makerSizeLimit": "14000000000000000000",
                "takerSizeLimit": null
            }
        ]
    ]
    ```

### Method: `feed_getQuoteFromStub`

Fetch a full quote and signed 0x order for a given quote stub (see [`feed_subscribe`](#method-feed_subscribe)).

To request a quote from a stub, either the `makerSize` or the `takerSize` MUST be included. Dealer implementations SHOULD allow traders to request a quote by specifying either the `makerSize` or the `takerSize`, filling in the value omitted in the request to create the price.

-   **Request fields:**

    | Index | Name           | JSON Type | Required | Default  | Description                                                                                   |
    | :---- | :------------- | :-------- | :------- | :------- | :-------------------------------------------------------------------------------------------- |
    | `0`   | `stubId`       | String    | `Yes`    | -        | The ID of the quote stub to get a full quote for.                                             |
    | `1`   | `makerSize`    | String    | `No`     | `null`   | If present, the dealer will use `makerSize` as the maker amount, filling in the taker amount. |
    | `2`   | `takerSize`    | String    | `No`     | `null`   | If present, the dealer will use `takerSize` as the taker amount, filling in the maker amount. |
    | `3`   | `takerAddress` | String    | `No`     | (note 3) | The address of the taker who will fill the quote. MAY be required by some implementations.    |
    | `4`   | `extra`        | Object    | `No`     | `null`   | OPTIONAL implementation-specific extra data.                                                  |

-   **Response fields:**

    | Index | Name        | JSON Type | Schema                         | Description                                                                                                    |
    | :---- | :---------- | :-------- | :----------------------------- | :------------------------------------------------------------------------------------------------------------- |
    | `0`   | `stubId`    | String    | [UUID](#schema-uuid)           | If valid, the UUID corresponding to the original quote stub. MUST be distinct from the `quoteId` in the quote. |
    | `1`   | `quote`     | Object    | [Quote](#schema-quote)         | A quote (offer) for the specified values from the client matching the original stub.                           |
    | `2`   | `tradeInfo` | Object    | [TradeInfo](#schema-tradeinfo) | Settlement information (e.g. gas price).                                                                       |
    | `3`   | `extra`     | Object    | -                              | OPTIONAL implementation-specific extra structured data relevant to this offer.                                 |

-   **Errors:**

    | Code     | Description                     | Notes                                                                                      |
    | :------- | :------------------------------ | :----------------------------------------------------------------------------------------- |
    | `-32603` | Internal error.                 | Internal JSON-RPC error. MAY be used as generic internal error code.                       |
    | `-42005` | Two size requests.              | Occurs when a client specifies both `baseAssetSize` and `quoteAssetSize`.                  |
    | `-42006` | Taker not authorized.           | Occurs when the taker's address is not authorized for trading.                             |
    | `-42007` | Invalid side.                   | Available for implementations to indicate lack of support for a currency-pair style quote. |
    | `-42008` | Temporary restriction.          | Available for implementations to indicate taker-specific temporary restrictions.           |
    | `-42011` | Quote too large.                | Occurs when a quote would exceed the market maximum or the dealer's balance.               |
    | `-42012` | Quote too small.                | Occurs when a quote would be smaller than the market's minimum size.                       |
    | `-42013` | Quote unavailable at this time. | Reserved for various states where dealers may not be serving quotes.                       |
    | `-42024` | Request rate limit reached.     | Available to indicate a implementation-specific request rate limit has been reached.       |
    | `-42025` | Stub no longer available.       | Can be used when a trader is trying to get a quote for a stub that has been updated.       |

-   **Example request body:**

    ```json
    [
        "3ff02eda-24e9-4e2c-9384-fcf08873dcc3",
        "135600000000000000000",
        null,
        "0x7df1567399d981562a81596e221d220fefd1ff9b"
    ]
    ```

-   **Example response body:**

    ```json
    [
        "3ff02eda-24e9-4e2c-9384-fcf08873dcc3",
        {
            "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
            "makerAssetAddress": "0xe41d2489571d322189246dafa5ebde1f4699f498",
            "takerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
            "makerAssetSize": "135600000000000000000",
            "takerAssetSize": "180000000000000000",
            "expiration": 1573775025132,
            "serverTime": 1573775014231,
            "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
            "order": {
                "makerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
                "feeRecipientAddress": "0x0000000000000000000000000000000000000000",
                "senderAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
                "makerAssetAmount": "135600000000000000000",
                "takerAssetAmount": "180000000000000000",
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
            "gasLimit": "210000",
            "gasPrice": "12000000000"
        },
        null
    ]
    ```

### Method: `feed_unsubscribe`

Terminate an open feed subscription, identified by the subscription UUID provided when the subscription was created.

-   **Request fields:**

    | Index | Name             | JSON Type | Required | Default | Description                             |
    | :---- | :--------------- | :-------- | :------- | :------ | :-------------------------------------- |
    | `0`   | `subscriptionId` | String    | `Yes`    | -       | The UUID of the subscription to cancel. |

-   **Response fields:**

    | Index | Name     | JSON Type | Schema | Description                                                              |
    | :---- | :------- | :-------- | :----- | :----------------------------------------------------------------------- |
    | `0`   | `status` | Boolean   | -      | MUST be `true` if the subscription was cancelled, and `false` otherwise. |

*   **Errors:**

    | Code     | Description                 | Notes                                                                                |
    | :------- | :-------------------------- | :----------------------------------------------------------------------------------- |
    | `-32603` | Internal error.             | Internal JSON-RPC error. MAY be used as generic internal error code.                 |
    | `-42024` | Request rate limit reached. | Available to indicate a implementation-specific request rate limit has been reached. |

*   **Example request body:**

    ```json
    ["3ff02eda-24e9-4e2c-9384-fcf08873dcc3"]
    ```

*   **Example response body:**

    ```json
    [true]
    ```

## Schemas

Schematics and data structures used in the public API (JSON shown).

Fields indicated `Yes` in the `Required` column for each scheme MUST be implemented, while fields indicated `No` MAY be omitted.

All schemas in this section MUST be supported to the degree indicated in each section if required to implement a required method.

### Schema: `Time`

All times are specified in milliseconds since the UNIX epoch as an integer Number.

Implementations MUST represent timestamps as whole integer numbers of milliseconds, without further decimal precision.

**Note:** the one exception is the `expirationTimeSeconds` field in [0x orders](#schema-order) which are UNIX seconds and represented as a String.

-   **JSON Example:**

    ```json
    1573774183353
    ```

### Schema: `UUID`

A universally unique identifier (UUID), according to UUID version 4.0 as a String.

-   **JSON Example:**
    ```
    "bafa9565-598d-413a-80d3-7ec3b7e24a08"
    ```

### Schema: `TradeInfo`

Defines information about trades – settlement transactions sent to the 0x exchange contract.

Implementations MAY include implementation-specific fields in this section.

The value for `gasPrice` MUST match the value ultimately included in any 0x [fill transaction](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#transactions) by dealer implementations for a given market.

-   **Fields**:

    | Name       | Schema | JSON Type | Description                                                                                                                               |
    | :--------- | :----- | :-------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
    | `chainId`  | -      | Number    | The [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) chain ID of the active Ethereum network (MUST match the EIP). |
    | `gasLimit` | -      | String    | The gas limit that will be used in `fillOrder` transactions submitted by the dealer.                                                      |
    | `gasPrice` | -      | String    | The gas price (in wei) that will be used in `fillOrder` transactions submitted by the dealer.                                             |

-   **JSON Example**:

    ```json
    {
        "chainId": 1,
        "gasLimit": "210000",
        "gasPrice": "12000000000"
    }
    ```

### Schema: `QuoteInfo`

Defines information about quote parameters for a given market. Does NOT included specific validity parameters (`ValidityParameter`) generated for individual quotes.

-   **Fields**:

    | Name      | Schema | JSON Type | Description                                                                |
    | :-------- | :----- | :-------- | :------------------------------------------------------------------------- |
    | `minSize` | -      | String    | The minimum supported trade size, in base units of a market's maker asset. |
    | `maxSize` | -      | String    | The maximum supported trade size, in base units of a market's maker asset. |

-   **JSON Example**:

    ```json
    {
        "minSize": "100000000000000",
        "maxSize": "10000000000000000000000000"
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
            "chainId": 1,
            "gasLimit": "210000",
            "gasPrice": "12000000000"
        },
        "quoteInfo": {
            "minSize": "100000000000000",
            "maxSize": "10000000000000000000000000"
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

    | Name                 | Schema                                                 | Required | JSON Type | Description                                                                                                                                                                                |
    | :------------------- | :----------------------------------------------------- | :------- | :-------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `quoteId`            | [UUID](#schema-uuid)                                   | `Yes`    | String    | A UUID (v4) that MUST correspond to this offer only.                                                                                                                                       |
    | `makerAssetAddress`   | -                              | `Yes`    | String    | Ethereum address of the quote's maker asset ERC-20 contract (see [quotes](#quotes)).                                                                                                                       |
    | `takerAssetAddress`   | -                              | `Yes`    | String    | Ethereum address of the quote's taker asset ERC-20 contract (see [quotes](#quotes)).                                                                                                                       |
    | `makerAssetSize`     | -                                                      | `Yes`    | String    | The quote's maker asset size provided by the dealer (see [quotes](#quotes)).                                                                                                               |
    | `takerAssetSize`     | -                                                      | `Yes`    | String    | The quote's taker asset size required by the client (see [quotes](#quotes)).                                                                                                               |
    | `expiration`         | [Time](#schema-time)                                   | `Yes`    | Number    | The UNIX timestamp after which requests to fill this quote will be rejected.                                                                                                               |
    | `serverTime`         | [Time](#schema-time)                                   | `Yes`    | Number    | The UNIX timestamp at which the server generated the quote. Helpful for clock synchronization.                                                                                             |
    | `orderHash`          | -                                                      | `No`     | String    | The 0x-specific order hash, as defined in the [v3 specification](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#hashing-an-order).              |
    | `order`              | [Order](#schema-order)                                 | `No`     | Object    | The dealer-signed [0x order](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#orders) that corresponds to this offer.                             |
    | `fillTx`             | -                                                      | `No`     | String    | The raw [0x fill transaction](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#transactions) data for this quote that the taker may sign (see 6). |
    | `validityParameters` | Array\<[ValidityParameter](#schema-validityparameter)> | `No`     | Array     | OPTIONAL implementation-specific "soft-cancel" parameters for this offer.                                                                                                                  |

-   **JSON Example**:

    ```json
    {
        "quoteId": "bafa9565-598d-413a-80d3-7ec3b7e24a08",
        "makerAssetAddress": "0xe41d2489571d322189246dafa5ebde1f4699f498",
        "takerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
        "makerAssetSize": "100000000000000000000",
        "takerAssetSize": "300000000000000000",
        "expiration": 1573775025112,
        "serverTime": 1573775014231,
        "orderHash": "0x0aeea0263e2c41f1c525210673f30768a4f8f280b2d35ffe776d548ea5004375",
        "order": {
            "chainId": 1,
            "makerAddress": "0xcefc94f1c0a0be7ad47c7fd961197738fc233459",
            "takerAddress": "0x7df1567399d981562a81596e221d220fefd1ff9b",
            "feeRecipientAddress": "0x0000000000000000000000000000000000000000",
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

### Schema: `QuoteStub`

Defines a public "quote stub," indicating a price bound and quantity limit for a given maker/taker asset pair that a trader may request a full quote for at a later time (see `dealer_getQuoteFromStub`).

When requesting a quote based on a price stub the trader can denominate the size in either the asset they are sending (the taker asset) or the asset they are receiving (the maker asset). The dealer implementation MUST fill in the other value, similar to how regular [quotes](#quotes) work. Thus, dealer's MUST service quote requests denominated in a quantity of either the maker or taker asset.

By using either the `makerSizeLimit` or the `takerSizeLimit`, dealers are able to choose whether to restrict price levels based on a quantity of either the maker or taker asset, which is generally useful to simplify internal accounting. For example, consider the following two quote stub:

```json
[
    {
        "stubId": "2b769dc1-87f3-4814-a2df-252d514188e8",
        "makerAssetTicker": "DAI",
        "takerAssetTicker": "WETH",
        "makerSizeLimit": null,
        "takerSizeLimit": "10000000000000000000",
        "makerPriceBand": [122.5, 124.1],
        "takerPriceBand": null
    },
    {
        "stubId": "2e58f94f-3e9f-4830-b775-1fd1483a199a",
        "makerAssetTicker": "WETH",
        "takerAssetTicker": "DAI",
        "makerSizeLimit": "10000000000000000000",
        "takerSizeLimit": null,
        "makerPriceBand": null,
        "takerPriceBand": [123.8, 125.2]
    }
]
```

Either `makerSizeLimit` and `takerPriceBand` or `takerSizeLimit` and `makerPriceBand` MUST be included in each stub — whichever fields are not present MUST be `null` and dealers MUST NOT include both `makerSizeLimit` and `takerSizeLimit` or both `makerPriceBand` and `takerPriceBand`.

More formally, the following MUST be true for each stub:

-   If `makerSizeLimit` is defined, `takerPriceBand` MUST be defined and `makerPriceBand` MUST be `null`.
-   If `takerSizeLimit` is defined, `makerPriceBand` MUST be defined and `takerPriceBand` MUST be `null`.
-   If `makerSizeLimit` is defined, `takerSizeLimit` MUST be `null`.
-   If `takerSizeLimit` is defined, `makerSizeLimit` MUST be `null`.

The dealer providing the stubs above has chosen to always represent prices on WETH/DAI markets in terms of the amount of DAI received or provided per unit of WETH. Dealers MAY choose to always represent priced in terms of the maker asset, taker asset, or a common "quote" asset (DAI in the example above).

-   **Fields**:

    | Name               | Schema                   | Required | JSON Type      | Description                                                                                            |
    | :----------------- | :----------------------- | :------- | :------------- | :----------------------------------------------------------------------------------------------------- |
    | `stubId`           | [UUID](#schema-uuid)     | `Yes`    | String         | A unique ID for this stub, needed to fetch a corresponding quote.                                      |
    | `makerAssetAddress` | - | `Yes`    | String         | The Ethereum address of the asset being offered by the dealer (maker) in this stub.                              |
    | `takerAssetAddress` | - | `Yes`    | String         | The Ethereum address of the asset being offered by the trader (taker) in this stub.                              |
    | `makerSizeLimit`   | -                        | `No`     | String         | The maximum available quantity of maker asset at the corresponding price level.                        |
    | `takerSizeLimit`   | -                        | `No`     | String         | The maximum available quantity of taker asset at the corresponding price level.                        |
    | `makerPriceBand`   | -                        | `No`     | Array\<Number> | The lower and upper bounds for the amount of the maker asset offered for each unit of the taker asset. |
    | `takerPriceBand`   | -                        | `No`     | Array\<Number> | The lower and upper bounds for the amount of the taker asset offered for each unit of the maker asset. |

*   **JSON Example**:

    ```json
    {
        "stubId": "3efff541-135a-4be8-9da7-f310d5338b1c",
        "makerAssetAddress": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
        "takerAssetAddress": "0x6b175474e89094c44da98b954eedeac495271d0f",
        "makerSizeLimit": "10000000000000000000",
        "takerSizeLimit": null,
        "makerPriceBand": null,
        "takerPriceBand": [120.9, 135.1]
    }
    ```

### Schema: `StubUpdate`

Dealers can indicate a change to an existing quote stub’s `makerSizeLimit` or `takerSizeLimit` with `StubUpdate` messages (sent to subscribing traders, or posted to a public venue), which reference the stub’s original ID and a new `makerSizeLimit` or `takerSizeLimit`.

If the respective value is set to 0, this indicates the stub has been canceled. Dealer’s MUST only provide updates that modify the max size for the maker or taker asset; to change price bounds for a stub, it MUST be cancelled and a new one SHOULD be issued.

Individual stub update messages MUST ONLY specify the `makerSizeLimit` or `takerSizeLimit` (not both), which MUST match the limit used in the original stub. For example, if a quote stub with ID `X` initially specifies a `makerSizeLimit`, updates to that stub MUST ONLY specify updated `makerSizeLimit` values. A stub update with ID `X` that specified a `takerSizeLimit` would be in violation of this specification and MUST NOT occur.

-   **Fields**:

    | Name             | Schema               | Required | JSON Type | Description                                                  |
    | :--------------- | :------------------- | :------- | :-------- | :----------------------------------------------------------- |
    | `stubId`         | [UUID](#schema-uuid) | `Yes`    | String    | The UUID of the stub that is being updated.                  |
    | `makerSizeLimit` | -                    | `No`     | String    | The new size limit for the stub in terms of the maker asset. |
    | `takerSizeLimit` | -                    | `No`     | String    | The new size limit for the stub in terms of the taker asset. |

*   **JSON Example**:

    ```json
    {
        "stubId": "3efff541-135a-4be8-9da7-f310d5338b1c",
        "makerSizeLimit": "8000000000000000000",
        "takerSizeLimit": null
    }
    ```

## Appendix

### Important resources

-   [JSONRPC 2.0 Specification](https://www.jsonrpc.org/specification)
-   [0x Protocol Specification (v3)](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md)
-   [0x Improvement Proposal (ZEIP) 18](https://github.com/0xProject/ZEIPs/issues/18)
-   [EIP-20 (ERC-20 specification)](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#symbol)

### Notes

1. This definition of a market makes an intentional departure from conventional currency-pair based markets in which their is a single quote asset and a single base asset. Defining only the maker and taker assets for a market allows greater flexibility for implementers, and allows pricing to be defined in terms of either asset at higher levels.
1. If the dealer is operating on the main Ethereum network, they MUST treat the `chainId` of `1` as the Ethereum mainnet, as [specified in EIP-155.](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) Private and test networks may use any chain ID, but SHOULD use conventions established by public test networks (e.g. Ropsten is 3).
1. The default value SHOULD be the null address (20 null bytes) represented as a hex string. Implementations MAY require takers to specify a `takerAddress`.
1. Quotes indicated as `includeOrder` as `false` can be seen as traders checking if a dealer's prices are favorable at a given time for a certain market and trade size.
    - Implementations MAY treat these types of quotes separately in internal tracking and/or pricing mechanisms.
1. Due to confusion between the word "transaction" referring to either an Ethereum transaction or a 0x ZEIP-18 transaction (`ZeroExTransaction`), wherever the word refers to an Ethereum transaction, it is explicitly stated. Similarly, wherever "transaction" refers to a ZEIP-18 message, it is explicitly described as a "0x transaction" or a "0x fill transaction" (the same goes for transaction hashes).

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
| `-42005` | Two size requests.                  | Occurs when a client specifies `makerAssetSize` and `takerAssetSize`.                             |
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
| `-42018` | Insufficient taker balance.         | Available to indicate specific validation failure.                                                |
| `-42019` | Insufficient taker allowance.       | Available to indicate specific validation failure.                                                |
| `-42020` | Quote validation failure.           | Available to indicate implementation-specific failures of extra quote data.                       |
| `-42021` | Invalid transaction ID.             | Available to indicate an invalid Ethereum transaction hash in a request.                          |
| `-42022` | Invalid order hash.                 | Available to indicate an order transaction hash in a request.                                     |
| `-42023` | Invalid UUID.                       | Available to indicate failure to validate a universally unique identifier (UUID).                 |
| `-42024` | Request rate limit reached.         | Available to indicate a implementation-specific request rate limit has been reached.              |
| `-42025` | Stub no longer available.           | Available to indicate that a quote stub has expired or is no longer supported for another reason. |
