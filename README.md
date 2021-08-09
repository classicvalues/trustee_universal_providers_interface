# Trustee Universal Interface

## Introduction

Each method must be implemented as a separate API endpoint.

![General](https://github.com/trustee-wallet/trustee_universal_providers_interface/blob/master/img/General.png)

## GET: Exchange ways list

You must implement an endpoint that will return a list of exchange ways.

Requirements for the exchange ways list are the same as for exchanger monitors.

Here is an [example of documentation](https://www.bestchange.com/wiki/rates.html) on how to properly create a exchange ways list.

```xml
<rates>
    <item>
        <from>XMR</from>
        <to>QWRUB</to>
        <in>1.00000000</in>
        <out>21284.32320000</out>
        <amount>851556.18</amount>
        <fromfee>0.000000 XMR</fromfee>
        <tofee>0.00 RUB</tofee>
        <minamount>0.039796 XMR</minamount>
        <maxamount>3.757901 XMR</maxamount>
    </item>
    <item>
        <from>LTC</from>
        <to>CARDRUB</to>
        <in>1.00000000</in>
        <out>13596.07014551</out>
        <amount>851556.18</amount>
        <fromfee>0.000000 LTC</fromfee>
        <tofee>0.00 RUB</tofee>
        <minamount>0.552109 LTC</minamount>
        <maxamount>3.330972 LTC</maxamount>
    </item>
</rates>
```

If you need to specify a fixed and percentage fee together, then you can do this as follows:

```xml
<fromfee>5 UAH</fromfee>
<fromfee>1.5 %</fromfee>
```

## Authentication

Methods **estimate amount**, **create order**, **check order** and **cancel order** must be authenticated.

Authentication parameters are transmitted in **Headers**.

The **request based auth headers** are transmitted by Trustee and should be checked on the exchanger side. The **response based auth headers** are transmitted by exchanger and should be checked on the Trustee side.

The signature for the request and response should be generated by the same logic.

![Auth](https://github.com/trustee-wallet/trustee_universal_providers_interface/blob/master/img/Auth.png)

An [example](https://github.com/trustee-wallet/trustee_universal_providers_interface/blob/master/signature.js) of generating a signature.

API keys are generated once by any side (Trustee or exchanger) and then share them to the other side.

| Header | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **trustee-public-key** | String  | optional | Partner's public key. |
| **trustee-timestamp**  | String | required | Timestamp that was used to generate the signature. |
| **trustee-signature** | String | required | Signature. |
| **trustee-env** | String | optional | The environment where a request is sent *(LOCAL, DEV or PROD)*. |

You can check the signature using this endpoint:

> first you need to share a API keys with the Trustee

`
https://testapiv3.trustee.deals/trustee-universal/check-signature
`

### Request body:

body that will be used to generate signature.

### Request headers:

| Header | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **trustee-public-key** | String  | required | Partner's public key. |
| **trustee-timestamp**  | String | required | Timestamp which will be used to generate signature. |

### Response body:

| Parameter | Type |  Description |
| ------ | ------ | ------ |
| **parametersSequence** | String  | Sequence of parameters for **initString**. |
| **initString**  | String | A string that will be used to generate a signature. |
| **signature**  | String | Signature for request body. |

### Response headers:

| Header | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **trustee-timestamp** | String  | required | Timestamp which was used to generate a response signature. |
| **trustee-signature**  | String | required | Signature for response body. |

With the help of **response based auth headers**, you can check the generation of response signature. It is identical to the request signature process.

### Example:

#### Request:

```curl
curl --location --request POST 'https://testapiv3.trustee.deals/trustee-universal/check-signature' \
--header 'trustee-public-key: <PUBLIC_KEY>' \
--header 'trustee-timestamp: 1623850218201' \
--header 'Content-Type: application/json' \
--data-raw '{
    "from": "ETH",
    "to": "CARDUAH",
    "fromAmount": 0.1,
    "extraToFee": 0.005
}'
```

#### Response body:

```JSON
{
    "initString": "0.005eth0.1carduah1623850218201",
    "parametersSequence": "extraToFee | from | fromAmount | to | timestamp",
    "signature": "af8aed2cb60d963e6d7c4bbac1cfd4dd4f06bad9ad9e24b10b7560e3b48bdf6ff69e62828fe0d7ecda7d8d4447c89f88d4f4981845528f86353ef0062245d551"
}
```

#### Response headers:

```JSON
{
    "trustee-timestamp": "1623850218201",
    "trustee-signature": "a5888ca92bd8aed805854474842dcda463acfd33bb359cb4aa93dd0bf36206a0d20d883b08f831359ee7dccbaeba92ed0b1212c3516ee74b577be446495dadae"
}
```

## POST: Estimate amount

> The endpoint must have request and response authentication.

### Request body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **from** | String  | required | Code for **from** currency. Same as in the exchange ways list. |
| **to**  | String | required | Code for **to** currency. Same as in the exchange ways list. |
| **fromAmount** or **toAmount** | Number  | required | The amount for which you need to calculate. Transmitted in the **from** currency or **to** currency. |
| **extraFromFee**\*  | Number | required if the exchanger supports\*\* | Trustee fee which will be taken from the **from** currency. |
| **extraToFee**\*  | Number | required if the exchanger supports\*\* | Trustee fee which will be taken from the **to** currency. |

\* – If Trustee fee is 0.5% then 0.005 must be transmitted to **extraFromFee** and/or **extraToFee**.

\*\* – If the exchanger does not support the dynamic setting of fees (**extraFromFee** and **extraToFee** parameters), then it can set it statically on its side. In this case, different Trustee fees will be set for different pairs of API keys.

### Response body:

| Parameter | Type | Required |  Description | Example
| ------ | ------ | ------ | ------ | ------ |
| **from** | String  | required | Code for **from** currency. Same as in the exchange ways list. | CARDRUB |
| **to**  | String | required | Code for **to** currency. Same as in the exchange ways list. | BTC |
| **fromAmount** | Number  | required | The amount that the client must pay. | 6500 |
| **toAmount** | Number  | required | The amount that the client will receive. | 0.0016 |
| **fromRate**\* | Number  | required | Rate that is represented in the **from** currency. | 3079761.9 |
| **toRate**\* | Number  | required | Rate that is represented in the **to** currency. | 1 |
| **fromFee**  | Number | required | Exchanger fee which will be taken from the **from** currency. | 0 |
| **toFee**  | Number | required | Exchanger fee which will be taken from the **to** currency. | 0.0005 |
| **extraFromFee**  | Number | required | Trustee fee which will be taken from the **from** currency. | 32.5 |
| **extraToFee**  | Number | required | Trustee fee which will be taken from the **to** currency. | 0 |

\* – One of the parameters (**fromRate** or **toRate**) must be "1", and the other show the rate.

#### Calculation example:

Trustee fee is 0.5%.

0.5% from 6500 = 32.5 RUB (**extraFromFee**)

6500 RUB – 32.5 RUB = 6467.5 RUB * 3079761.9 (**fromRate**) = 0.0021 BTC

0.0021 BTC – 0.0005 BTC (**toFee**) = 0.0016 BTC (**toAmount**).

#### In case of error:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **errorCode** | String  | required | Code for error. |
| **message**  | String | required | Error description. |
| **minAmount** or **maxAmount**\* | Number  | required\* | The limit on which the user has not passed. You need to transmit only one parameter. |

\* – **minAmount** or **maxAmount** must be transmitted only when the **errorCode** is equal to "EXCEEDING_LIMITS".
If the **fromAmount** was transmitted to the request, then the **minAmount** or **maxAmount** must be specified in the **from** currency.
If the **toAmount** was transmitted to the request, then the **minAmount** or **maxAmount** must be specified in the **to** currency.

#### Error codes list:

| Parameter |  Description |
| ------ | ------ |
| **EXCEEDING_LIMITS** | The user has not passed on acceptable limits. |
| **PROVIDER_ERROR**  | Any other error. |

### Example:

#### Request:

```curl
curl --location --request POST <EXCHANGER_ENDPOINT> \
--header 'trustee-public-key: <PUBLIC_KEY>' \
--header 'trustee-timestamp: 1624449249310' \
--header 'trustee-signature: 91a648f5b40092a4a3600e6cbe39f12a06c78ac81044a6028a28d9648f2c924488498263df318a73593f6f97f2395a0575055f2ff5096b85f348b0a2f15b14b0' \
--header 'Content-Type: application/json' \
--data-raw '{
    "from": "BTC",
    "to": "ETH",
    "fromAmount": 0.001,
    "extraFromFee": 0.002
}'
```

#### Response body:

```JSON
{
    "from": "BTC",
    "to": "ETH",
    "fromAmount": 0.001,
    "toAmount": 0.013062,
    "fromRate": 1,
    "toRate": 16.696118100522533,
    "fromFee": 0,
    "toFee": 0.0036,
    "extraFromFee": 0.000002,
    "extraToFee": 0
}
```

#### Response headers:

```JSON
{
    "trustee-timestamp": "1624450908003",
    "trustee-signature": "74a3f34bbe12838daa60737ca43a5b9e9b61f265d9ba5911211e0ea68a17650880a26a368698327c05638bb06c55bbc640a5f23d4054e54a73a2e7d08ec9d89c"
}
```

## POST: Create order

> The endpoint must have request and response authentication.

### Request body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **from** | String  | required | Code for **from** currency. Same as in the exchange ways list. |
| **to**  | String | required | Code for **to** currency. Same as in the exchange ways list. |
| **fromAmount** | Number  | required | The amount that the client must pay. |
| **toAmount** | Number  | required | The amount that the client will receive. |
| **userId** | String  | optional | Anonymous user ID. |
| **redirectUrl** | String  | required | Url where you need to redirect the user after payment via **payUrl** (only for fiat deposit). |
| **toPaymentDetails** | String  | required | Payment details of where the user will receive funds. |
| **fromPaymentDetails** | String  | optional | Payment details from which will take funds. |
| **toMemo** | String  | optional | If additional data must be attached to the **toPaymentDetails**, for example for XRP currency. |
| **extraFromFee**\*  | Number | required if the exchanger supports\*\* | Trustee fee which will be taken from the **from** currency. |
| **extraToFee**\*  | Number | required if the exchanger supports\*\* | Trustee fee which will be taken from the **to** currency. |

\* – If Trustee fee is 0.5% then 0.005 must be transmitted to **extraFromFee** and/or **extraToFee**.

\*\* – If the exchanger does not support the dynamic setting of fees (**extraFromFee** and **extraToFee** parameters), then it can set it statically on its side. In this case, different Trustee fees will be set for different pairs of API keys.

### Response body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **id** | String | required | Order ID. |
| **payUrl**\* | String | required\* | Link to pay fiat. It needs to be opened to the client. |
| **payCryptoAddress**\* | String | required\* | Cryptocurrency address where the client needs to send money. |
| **payCryptoMemo** | String | optional | Additional information to the **payCryptoAddress** (need for example for XRP currency). |
| **from** | String  | required | Code for **from** currency. Same as in the exchange ways list. |
| **to**  | String | required | Code for **to** currency. Same as in the exchange ways list. |
| **fromAmount** | Number  | required | The amount that the client must pay. |
| **toAmount** | Number  | required | The amount that the client will receive. |
| **fromAmountReceived**\*\* | Number  | optional | The amount without a bank fee if it is (*only for Fiat -> Crypto exchange ways*). |
| **userId** | String  | optional | Anonymous user ID. |
| **redirectUrl** | String  | required | The amount that the client will receive. |
| **toPaymentDetails** | String  | required | Payment details of where the user will receive funds. |
| **fromPaymentDetails** | String  | optional | Payment details from which will take funds. |
| **toMemo** | String  | optional | If additional data must be attached to the **toPaymentDetails**, for example for XRP currency. |
| **extraFromFee**  | Number | required | Trustee fee which will be taken from the **from** currency. |
| **extraToFee**  | Number | required | Trustee fee which will be taken from the **to** currency. |

\* – Only one of the parameter must be returned in the response (**payUrl** or **payCryptoAddress**). It depends on whether the client needs to make a fiat deposit (**payUrl** must be returned) or crypto deposit (**payCryptoAddress** must be returned).

\*\* If **fromAmount** = 500 UAH and bank fee = 3%, then **fromAmountReceived** = 485 UAH.

All parameters that were used when creating should return to the response.

#### In case of error:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **message**  | String | required | Error description. |

### Example for crypto deposit:

#### Request:

```curl
curl --location --request POST <EXCHANGER_ENDPOINT> \
--header 'trustee-public-key: <PUBLIC_KEY>' \
--header 'trustee-timestamp: 1624453098700' \
--header 'trustee-signature: 582b04b693ae737c43a13821273c5ded8892444d542376157f87a734653cc2272e84a7fbed1e4bbc482f852885a15e806a149257d128fddb6db782c5751bec44' \
--header 'Content-Type: application/json' \
--data-raw '{
    "from": "BTC",
    "to": "ETH",
    "fromAmount": 0.001,
    "toAmount": 0.013018,
    "toPaymentDetails": "0xc24D2f7E2d6355dCe30C45CcDdA56b5C5fecC254",
    "userId": "4MkQ4RTk",
    "extraFromFee": 0.002
}'
```

#### Response body:

```JSON
{
    "id": "QCqQF7U2BTebKL3Z",
    "status": "WAITING",
    "payCryptoAddress": "1BLvYTj7oDgYVA21sYNdxj96dKoUHvdazc",
    "toPaymentDetails": "0xc24D2f7E2d6355dCe30C45CcDdA56b5C5fecC254",
    "from": "BTC",
    "to": "ETH",
    "fromAmount": "0.001",
    "toAmount":"0.013048",
    "fromRate": 1,
    "toRate": 16.682383815934614,
    "fromFee": 0,
    "toFee": 0.0036,
    "extraFromFee": 0.000002,
    "extraToFee": 0
}
```

#### Response headers:

```JSON
{
    "trustee-timestamp": "1624453099750",
    "trustee-signature": "8a7e8c23fc3da94263fbe4e2ed4162fa9a1f05e027b8dc47862449879acca0c80e75797c66e5e70ba1816e2451412d0de53975d9e47473ccda8c2f5ebd109ff4"
}
```

## GET: Check order

> The endpoint must have request and response authentication.

### Request body (Query string):

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **id** | String  | required | Order ID. |

### Response body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **id** | String | required | Order ID. |
| **status** | String | required | Order status |
| **payUrl**\* | String | required\* | Link to pay fiat. It needs to be opened to the client. |
| **payCryptoAddress**\* | String | required\* | Cryptocurrency address where the client needs to send money. |
| **payCryptoMemo** | String | optional | Additional information to the **payCryptoAddress** (need for example for XRP currency). |
| **from** | String  | required | Code for **from** currency. Same as in the exchange ways list. |
| **to**  | String | required | Code for **to** currency. Same as in the exchange ways list. |
| **fromAmount** | Number  | required | The amount that the client must pay. |
| **toAmount** | Number  | required | The amount that the client will receive. |
| **fromAmountReceived**\*\* | Number  | optional | The amount without a bank fee if it is (*only for Fiat -> Crypto exchange ways*). |
| **userId** | String  | optional | Anonymous user ID. |
| **redirectUrl** | String  | required | The amount that the client will receive. |
| **toPaymentDetails** | String  | required | Payment details of where the user will receive funds. |
| **fromPaymentDetails** | String  | optional | Payment details from which will take funds. |
| **toMemo** | String  | optional | If additional data must be attached to the **toPaymentDetails**, for example for XRP currency. |
| **fromTxHash** | String  | required | Hash transaction of client deposit. |
| **toTxHash** | String  | required | Hash transaction of payment to the client. |
| **extraFromFee**  | Number | required | Trustee fee which will be taken from the **from** currency. |
| **extraToFee**  | Number | required | Trustee fee which will be taken from the **to** currency. |

\* – Only one of the parameter must be returned in the response (**payUrl** or **payCryptoAddress**). It depends on whether the client needs to make a fiat deposit (**payUrl** must be returned) or crypto deposit (**payCryptoAddress** must be returned).

\*\* If **fromAmount** = 500 UAH and bank fee = 3%, then **fromAmountReceived** = 485 UAH.

### Order statuses

| Status |  Description |
| ------ | ------ |
| **WAITING** | Waiting for client deposit. |
| **RECEIVED**  | Received client deposit. |
| **EXCHANGING** | Exchange is carried out. |
| **SENDING**  | In the process of payment to the client. |
| **COMPLETED** | Order successfully completed. |
| **NOT_ENTIRE_WITHDRAW**  | Not the whole amount is paid (occurs during fiat conclusions). In this case, the amount that the client has already received should be specified in **toAmount**. |
| **REFUNDED** | The money was returned to the client. |
| **EXPIRED**  | The order was not paid for the time allocated (some exchangers know how to automatically restore such orders by recalculating the course through another endpoint). |
| **CANCELED** | The order was canceled. |
| **FAILED**  | In the process of execution of the order, an error occurred. |
| **HOLDED** | The warrant is suspended to check KYC. |

#### In case of error:
| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **message**  | String | required | Error description. |

### Example:

#### Request:

```curl
curl --location --request GET '<EXCHANGER_ENDPOINT>?id=QCqQF7U2BTebKL3Z' \
--header 'trustee-public-key: <PUBLIC_KEY>' \
--header 'trustee-timestamp: 1624454742270' \
--header 'trustee-signature: 0d7efa790650a4ea719d998eb494ac11d2969da9ec1873e7477d02d51dc4ba5c2b1d3b4290b333ec854bd88aaadcd26ccac9ff08bb4f9fa60572cb69040fc7a0'
```

#### Response body:

```JSON
{
    "id": "QCqQF7U2BTebKL3Z",
    "status": "WAITING",
    "payCryptoAddress": "1BLvYTj7oDgYVA21sYNdxj96dKoUHvdazc",
    "toPaymentDetails": "0xc24D2f7E2d6355dCe30C45CcDdA56b5C5fecC254",
    "from": "BTC",
    "to": "ETH",
    "fromAmount": 0.001,
    "toAmount": 0.013048,
    "fromAmountReceived": 0,
    "fromRate": 1,
    "toRate": 16.682383815934614,
    "fromFee": 0,
    "toFee": 0.0036,
    "extraFromFee": 0.000002,
    "extraToFee": 0
}
```

#### Response headers:

```JSON
{
    "trustee-timestamp": "1624454742612",
    "trustee-signature": "8016bad0fc5adb2ba7553c46e9c29e4236f608816bab16dd043a32797b8c3163de15bcacf0f8d18b1aa8e6b9507082e2a7ba9e577c60ede418099c46c366c80f"
}
```

## POST: Cancel order

> The endpoint must have request and response authentication.

### Request body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **id** | String  | required | Order ID. |

### Response body:
| Parameter | Type | Required |  Description | Example
| ------ | ------ | ------ | ------ | ------ |
| **status** | String  | required | Status of the canceling. | SUCCESS |

#### In case of error:
| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **message**  | String | required | Error description. |

### Example:

#### Request:

```curl
curl --location --request POST <EXCHANGER_ENDPOINT> \
--header 'trustee-public-key: <PUBLIC_KEY>' \
--header 'trustee-timestamp: 1624455217149' \
--header 'trustee-signature: 3ef5bd5345c1227d1e84517e2cb5d25a13a93335c7bb4e2f0327a9911d98067e6dfcef7944857d96b9b26084ac8789c1dbef4f263f8ecf74ab0865ee373676b5' \
--header 'Content-Type: application/json' \
--data-raw '{
    "id": "QCqQF7U2BTebKL3Z"
}'
```

#### Response body:

```JSON
{
    "status": "SUCCESS"
}
```

#### Response headers:

```JSON
{
    "trustee-timestamp": "1624455218707",
    "trustee-signature": "ab3dafb47d7079735bbc7e0b0e67707abce2123d6e6d6bb8890a209fee58931d284bca36f4cc1ecf63419ad95184750e70a0550bd0ded9c60ab810b9bda46cbd"
}
```

## POST: Restore EXPIRED order (optional)

> The endpoint must have request and response authentication.

### Request body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **id** | String  | required | Order ID. |

### Response body:

| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **id** | String | required | Order ID. |
| **status** | String | required | Order status |
| **payUrl**\* | String | required\* | Link to pay fiat. It needs to be opened to the client. |
| **payCryptoAddress**\* | String | required\* | Cryptocurrency address where the client needs to send money. |
| **payCryptoMemo** | String | optional | Additional information to the **payCryptoAddress** (need for example for XRP currency). |
| **from** | String  | required | Code for **from** currency. Same as in the exchange ways list. |
| **to**  | String | required | Code for **to** currency. Same as in the exchange ways list. |
| **fromAmount** | Number  | required | The amount that the client must pay. |
| **toAmount** | Number  | required | **Actual** amount that the client will receive. |
| **fromAmountReceived**\*\* | Number  | optional | The amount without a bank fee if it is (*only for Fiat -> Crypto exchange ways*). |
| **userId** | String  | optional | Anonymous user ID. |
| **redirectUrl** | String  | required | The amount that the client will receive. |
| **toPaymentDetails** | String  | required | Payment details of where the user will receive funds. |
| **fromPaymentDetails** | String  | optional | Payment details from which will take funds. |
| **toMemo** | String  | optional | If additional data must be attached to the **toPaymentDetails**, for example for XRP currency. |
| **fromTxHash** | String  | required | Hash transaction of client deposit. |
| **toTxHash** | String  | required | Hash transaction of payment to the client. |
| **extraFromFee**  | Number | required | Trustee fee which will be taken from the **from** currency. |
| **extraToFee**  | Number | required | Trustee fee which will be taken from the **to** currency. |

\* – Only one of the parameter must be returned in the response (**payUrl** or **payCryptoAddress**). It depends on whether the client needs to make a fiat deposit (**payUrl** must be returned) or crypto deposit (**payCryptoAddress** must be returned).

\*\* If **fromAmount** = 500 UAH and bank fee = 3%, then **fromAmountReceived** = 485 UAH.

#### In case of error:
| Parameter | Type | Required |  Description |
| ------ | ------ | ------ | ------ |
| **message**  | String | required | Error description. |

### Example:

#### Request:

```curl
curl --location --request POST <EXCHANGER_ENDPOINT> \
--header 'trustee-public-key: <PUBLIC_KEY>' \
--header 'trustee-timestamp: 1624455217149' \
--header 'trustee-signature: 3ef5bd5345c1227d1e84517e2cb5d25a13a93335c7bb4e2f0327a9911d98067e6dfcef7944857d96b9b26084ac8789c1dbef4f263f8ecf74ab0865ee373676b5' \
--header 'Content-Type: application/json' \
--data-raw '{
    "id": "QCqQF7U2BTebKL3Z"
}'
```

#### Response body:

```JSON
{
    "id": "QCqQF7U2BTebKL3Z",
    "status": "WAITING",
    "payCryptoAddress": "1BLvYTj7oDgYVA21sYNdxj96dKoUHvdazc",
    "toPaymentDetails": "0xc24D2f7E2d6355dCe30C45CcDdA56b5C5fecC254",
    "from": "BTC",
    "to": "ETH",
    "fromAmount": 0.001,
    "toAmount": 0.013048,
    "fromAmountReceived": 0,
    "fromRate": 1,
    "toRate": 16.682383815934614,
    "fromFee": 0,
    "toFee": 0.0036,
    "extraFromFee": 0.000002,
    "extraToFee": 0
}
```

#### Response headers:

```JSON
{
    "trustee-timestamp": "1624455218707",
    "trustee-signature": "ab3dafb47d7079735bbc7e0b0e67707abce2123d6e6d6bb8890a209fee58931d284bca36f4cc1ecf63419ad95184750e70a0550bd0ded9c60ab810b9bda46cbd"
}
```
