# Tresso Open API Specification

> ```
> Copyright Tresso © 2019   v1.4.3
> ```

[TOC]


# Note

* The base URL for following API is always ： **https://api.tresso.com/broker/spot/openApi**
* ALL message in the API  are in JSON format.
* All time and timestamp are UNIX epoch time, in milliseconds.



## Quick Start

1. Apply institution account and get `accountID` and `secretKey`
2. Use `JWT` to sign every request.
3. Send new order through `order.insertOrder`
4. Use `Websocket` client to get order status.



## Make Request
Use HTTP POST to make request, for example:
```assembly
URL：
https://api.tresso.com/broker/spot/openApi?signature={signature}

HTTP：
POST /broker/spot/openApi HTTP/1.1
Host: api.tresso.com
Content-Type: application/json
cache-control: no-cache
{
    "accountId": "ST-00000001"
}
```

The request end point is imbedded in the `signature`.



## Signature

Please use JWT to generate Signature for each request , please refer to [jwt.io](https://jwt.io/) on how to use JWT.

For every request, please use JWT and secretKey to generate Signature and use Signature in HTTP post parameter：

```js
signature = JWT(payload={
    "accountId": "accountId",
    "secretKeyId": "secretKeyId",
    "digest": hash256(RequestBody),
    "method": "module.methodName",
    "exp": 1535526941
}, key=secretKey, algorithm='HS256')
```

Payload in Signature includes following content:

- digest：hash value of the request body using sha256 hash.
- method：api method
- exp：expiration time of signature.



## Standard response

After making API request, the server will send back a response in JSON format, the structure of response is

```json
{
  "result": null,
  "error": {
    "code": 131114,
    "message": "account not found"
  }
}
```
The response include

* result：when request is successful, this key will include result of request. When request if not successful, its value is null.
* error：when request is not successful, the key will give the error message. Its value is null when request is successful.
* code：Error code. Please refer to error code explanation section.
* message：Error message.



# API

**Note：**the response in following API is the `result` part of standard response.



## Order API

Order ID needs to be following format: account id plus 13 digit of timestamp + 3 digit of random number.  


### Create New Order

Make a new order a request：

```assembly
method:	order.insertOrder
```
Parameter:

```json
{
  //Account ID
  "accountId": "ST-0000001",
  //Order id
  "orderId": "000000121574697605570321",
  //Order Info
  "orderInfo": {
    //symbol
    "symbol": "BTCUSD",
    //order type
    "orderType": "LIMIT",
    //order side
    "orderSide": "BUY",
    //limit price
   	"limitPrice": "100",
    //quantity
    "quantity": "1.12"
  }
}
```

**Response:**

```json
{
  //Account Id
  "accountId": "ST-0000001",
  //Order Id
  "orderId": "000000121574697605570321",
  // Exchange Order Id -- reserved for future enhancement
  "exchOrdId" : "",
  //symbol
  "symbol": "BTCUSD",
  //order type
  "orderType": "LIMIT",
  //order side
  "orderSide": "BUY",
  //limit price
  "limitPrice": 100,
  //quantity
  "quantity": 1.12,
  //average fill price
  "filledAveragePrice": 0,
  //accumulated fill quanity
  "filledCumulativeQuantity": 0,
  //Open quantity to be filled
  "openQuantity": 1.12,
  //Order Status
  "orderStatus": "PENDING_SUBMIT",
  //Order creation timestamp
  "createdAt": 1537857271784,
  //Order update timestamp
  "updatedAt":  1537857271784,
  //if it is cancelled, cancellation timestamp    
  "cancelledUpdatedAt": null,
  //Last filled timestamp
  "filledUpdatedAt": null

}
```

**Order Type:**

- LIMIT: Limit Order

**Order Side:**

- BUY
- SELL

**Order Status:**

| Order Status   | Explanation                                                 |
| -------------- | ----------------------------------------------------------- |
| PENDING_SUBMIT | Order has been send but not confirmed                       |
| SUBMITTED      | and has been accepted and acknowledged or Partially filled. |
| FILLED         | Completely filled                                           |
| CANCELLED      | Order is cancelled                                          |
| REJECTED       | Order is rejected.                                          |


### Cancel Order

Request Cancel Order，cancelledUpdatedAt in Order will be updated.

**Note：Order can only be cancelled when order status is not `PENDING_SUBMIT`，and `cancelledUpdatedAt` is null.**

```assembly
method:		order.cancelOrder
```
**Parameters:**

```json
{
  "accountId": "ST-0000001",
  "orderId": "000000121574697605570321"
}
```

**Response:**

```json
{
  //Same as new order response
}
```



### Query Single Order Info

```assembly
method:	order.queryOrderInfo
```

**Parameters:**

```json
{
  "accountId": "STA-01102061",
  "orderId": "000000121574697605570321"
}
```

**Response:**

```json
{
  //same as new order response
}
```



### Query Multiple Order Info

```assembly
method:	order.listOrderInfo
```

**Parameter:**

```json
{
  "accountId": "STA-01102061",
	//the total number of order id cannot exceed 500
	"orderIdList": ["000000121574697605570321","000000121574697754664817"]
}
```

**Response:**

```json
[
  {
     //Same as new order response
  }，
  {
     //Same as new order response
  }
]
```



### Query All Open Order

```assembly
method:	order.listOpenOrder
```
**Parameter:**

```json
{
  "accountId": "ST-0000001"
}
```

**Response:**

```json
[
  {
     //Same as new order response
  }，
  {
     //Same as new order response
  }
]
```



### Query All Completed Order

Query all filled orders, cancelled orders, and rejected orders

```assembly
method:	order.listCompletedOrder
```

**Parameters:**

```json
{
  //Account ID
	"accountId": "ST-0000001",
  //start time
  "startTime": 1537857271784,
  //end time
  "endTime": 1537857271784,
  //total number of orders, max is 500
  "limit": 100,
  //true for forward, false for backward. (optional)
  "forward": null,
  //base asset. for example, for BTCUSD，it is BTC. (optional)
  "baseAsset": null,
  //quote asset，For BTCUSD，it is USD. (optional)
  "quoteAsset": null,
  //Order Side: BUY, SELL. (optional)
  "orderSide": null,
  //Order Status: FILLED, CANCELLED. (optional)
  "orderStatus": null
}
```

**Response:**

```json
[
  {
     //Same as new order response
  }，
  {
     //Same as new order response
  }
]
```



## Order Status Push Notice

###Order Status change notice API

Order status change notice can be obtained through HTTP and Websocket.

```assembly
method:	order.streamDetail
```
**Parameter:**

```json
{
    "accountId": "ST-00000001"
}
```

**Response:**

```json
{
 "logTimestamp" : 1552116459512,
 "payload" : {
   "accountId" : "ST-00000001",
   "orderId" : "000000121574697605570321",
   "symbol" : "BTCUSD",
   "orderType" : "LIMIT",
   "orderSide" : "BUY",
   "limitPrice" : 100.0,
   "quantity" : 1.12,
   "filledAveragePrice" : 0,
   "filledCumulativeQuantity" : 0,
   "openQuantity" : 0.001,
   "orderStatus" : "PENDING_SUBMIT",
   "createdAt" : 1537857271784,
   "updatedAt" : 1537857271784,
   "cancelledUpdatedAt" : null,
   "filledUpdatedAt" : null,
   "lastFilledQuantity" : null,
   "lastFilledPrice" : null,
   "lastFilledCreatedAt" : null
 }
}
```
```json
{
 "logTimestamp" : 1552116459536,
 "payload" : {
   "accountId" : "ST-00000001",
   "orderId" : "000000121574697605570321",
   "symbol" : "BTCUSD",
   "orderType" : "LIMIT",
   "orderSide" : "BUY",
   "limitPrice" : 100.0,
   "quantity" : 1.12,
   "filledAveragePrice" : 100.0,
   "filledCumulativeQuantity" : 1.12,
   "openQuantity" : 0,
   "orderStatus" : "FILLED",
   "createdAt" : 1537857271784,
   "updatedAt" : 1537857271784,
   "cancelledUpdatedAt" : null,
   "filledUpdatedAt" : 1552116459467,
   //last filled quantity
   "lastFilledQuantity" : 1.12,
   //last filled price
   "lastFilledPrice" : 100.0,
   //last filled time
   "lastFilledCreatedAt" : 1537857271784
 }
}
```



### WebSocket API

Order update notice can be obtained through WebSocket ：

```assembly
# Need to add one parameter in url：&p=webSocket
ws[s]://api.tresso.com/broker/spot/openApi?signature={signature}&body={body}&p=webSocket
method:	order.streamDetail

RESULT:
{
 "logTimestamp" : 1552116459536,
 "payload" : {
   "accountId" : "ST-00000001",
   "orderId" : "000000121574697605570321",
   "symbol" : "BTCUSD",
   "orderType" : "LIMIT",
   "orderSide" : "BUY",
   "limitPrice" : 100.0,
   "quantity" : 1.12,
   "filledAveragePrice" : 100.0,
   "filledCumulativeQuantity" : 1.12,
   "openQuantity" : 0,
   "orderStatus" : "FILLED",
   "createdAt" : 1552116459467,
   "updatedAt" : 1552116459467,
   "cancelledUpdatedAt" : null,
   "filledUpdatedAt" : 1552116459467,
   //last filled quantity
   "lastFilledQuantity" : 1.12,
   //last filled price
   "lastFilledPrice" : 100.0,
   //last filled time
   "lastFilledCreatedAt" : 1552116459467
 }
}
```



## Account Information

### Query Account Info		

```assembly
method:	account.queryAccountInfo
```

**Parameters:**

```json
{
  //Account ID
  "accountId": "ST-0000001",
}
```

**Response:**

```json
{
  //Account ID
  "accountId": "STA-0000001",
  //Account Status
  "accountStatus": "OPENED",
  //Commission Rate
  "orderCommissionRate": "1",
  //Commission adjust is allowed
  "orderCommissionAdjust": "false",
  //creation time
  "createdAt": 1537857271784,
  //update time
  "updatedAt": 1551956476878,
}
```

**Account Status:**

| Account Status | Explanation                                                  |
| -------------- | ------------------------------------------------------------ |
| OPENED         | normal                                                       |
| RESTRICTED     | restricted. Cannot submit order, cancel order, but can query order. |
| SUSPENDED      | Suspended, all operation are not allowed.                    |



## Asset

### Query Account Asset

```assembly
method:	asset.listBalance
```

**Parameters:**

```json
{
  //Account ID
  "accountId": "ST-0000001",
}
```

**Response:**

```json
[
  {
    //Account ID
    "accountId": "ST-0000001",
    //Symbol of asset
    "currency": "BTC",
    //Available quantity
    "available": 51.95,
    //frozen value
    "frozen": 0,
    //amount
    "amount": 51.95,
    //update time
    "updatedAt": 1547208404061
  }
]
```



## Tools API

### Query List Symbol

```assembly
method:	utils.listSymbol
```

**Parameters:**
None

**Response:**

```json
[
    {
      //symbol
      "symbol": "BTCUSD",
      //symbol status
      "status": "TRADING",
      //next status
      "nextStatus": null,
      //next status time
      "nextStatusAt": null,
      //base asset
      "baseAsset": "BTC",
      //base asset precision
      "baseAssetPrecision": 8,
      //quote asset，the right side of symbol
      "quoteAsset": "USD",
      //quote asset precision
      "quotePrecision": 8,
      //min price
      "minPrice": 0.00000001,
      //max price
      "maxPrice": 99999,
      //min price increament size
      "priceTickSize": 0.000001,
      //min quantity
      "minQuantity": 0.01,
      //max quantity
      "maxQuantity": 99999,
      //quantity step size
      "quantityStepSize": 0.01,
      //commission rate
      "commissionRate": 0.01
    }
  ]
```

**symbol status:**

| symbol status | explanation       |
| ------------- | ----------------- |
| TRADING       | normal trading    |
| HALT          | trading is halted |



### Query asset currency

```assembly
method:	utils.listCurrency
```

**Parameters:**
None

**Response:**

```json
[
  {
    //currency
    "currency": "BTC",
    //currency precision
    "currencyPrecision": 8,
    //max withdraw amount
    "withdrawMaxAmount": 1000,
    //min withdraw ammount
    "withdrawMinAmount": 0.01,
    //min withdraw fee
    "withdrawMinFee": 0.0001,
    //current status
    "status": "DEPOSIT_WITHDRAW",
    //next status
    "nextStatus": null,
    //next status time
    "nextStatusAt": null
  }
]
```

**currency status:**

|status|explanation|
| -- | -- |
| DEPOSIT_WITHDRAW| deposit and withdraw are both allowed |
| NOT_DEPOSIT_WITHDRAW| deposit is not allowed, withdraw is allowed |
| DEPOSIT_NOT_WITHDRAW| deposit is allowed, but withdraw is not allowed |
| NOT_DEPOSIT_NOT_WITHDRAW| deposit and withdraw are both not allowed |



### Query current timestamp of server

```assembly
method:	utils.currentTimeMillis
```

**Parameter:**
None

**Response:**

```json
//the server timestamp
1552112542263
```



# Market Data Specification

Market data is obtained through `Websocket` or `RESTful` interface. 



## Websocket Inteface

The websocket end point is

```assembly
ws://api.tresson.com/v1 
```



The request for subscription must be in JSON, is as follows:

```json
{
  "channel":"orderbook",
  "symbol":"ETHBTC",
  "action":"sub"
} 
```

Request Message for unsubscribing (Must be in JSON) is 

```json
{
  "channel":"orderbook",
  "symbol":"ETHBTC",
  "action":"unsub"
} 
```


The response of websocket request is also in json format, 

```json
{
  "channel" : "orderbook",
  "symbol" : "ETHBTC",
  "orderbook" : {
    "symbol" : "ETHBTC",
    "updatedAt" : 1574713605000,
    "asks": [
        [10000.0, 0.004]
      	// [price, size]
      	// ……
    ],
    "bids": [
        [7000.0, 0.04],
        [0.031046, 0.0048]
    ]
 } 
```



## RESTful Interface 

RESTFul interface endpoint is: 
```assembly
https://ap.tressio.com/v1/marketdata/orderbook/#SYMBOL#
```

The `#SYMBOL#` is the pair to be subscribed. For example, ETHBTC.



The response is in following json format:
```json
{
    "symbol": "ETHBTC",
    "updatedAt": 1574713711000,
    "asks": [
        [10000.0, 0.004]
      	// [price, size]
      	// ……
    ],
    "bids": [
        [7000.0, 0.04],
        [0.031046, 0.0048]
    ]
} 
```



# FIX Protocol Specification

FIX Protocol can be implemented using `QuickFIX` open source engine，please refer to ：[QuickFIX](http://www.quickfixengine.org)

The following language is supported ：

- [C++, Python, Ruby](http://www.quickfixengine.org/)
- [Java ](https://sourceforge.net/projects/quickfixj/files/)
- [Go](https://www.quickfixgo.org/)
- [.NET](http://quickfixn.org/)



## Configuration for quickfix

an optional sample configuration for login using quickfix：

```properties
[default]
ConnectionType=initiator
SenderCompID=ST-00000015
TargetCompID=broker
SocketConnectHost=localhost
StartTime=00:00:00
EndTime=00:00:00
HeartBtInt=30
ReconnectInterval=5
UseDataDictionary=Y
ResetSeqNumFlag=Y
ResetOnLogon=Y
ResetOnLogout=Y
ResetOnDisconnect=Y

[session]
BeginString=FIX.4.2
SocketConnectPort=9878
```

**Below is an example using `Python`：**



## Logon

Add `JWT` encrypted token in `toAdmin` ：
```python
 def toAdmin(self, message, sessionID):
        if message.getHeader().getField(35) is "A":
            key = 'XXXX'
            signature = get_signature('new_gbbo_order', 'ST-00000015', 'STA-00000015','STA-00000015', key)
            message.setField(fix.RawDataLength(len(signature)))
            message.setField(fix.RawData(signature))
        ...
```

 `get_signature` is as following：
```python
def get_signature(method, account_id, secret_key_id, params, secret_key):
    now = datetime.datetime.now()
    expect_time = now + datetime.timedelta(seconds=30)
    params_json_string = json.dumps(params) if isinstance(params, dict) else params
    digest = hashlib.sha256(params_json_string.encode()).hexdigest()
    signature = jwt.encode({
        'accountId': account_id,
        'secretKeyId': secret_key_id,
        'digest': digest,
        'method': method,
        'exp': int(expect_time.timestamp()),
    }
        , secret_key)
    # print(str(signature, encoding='utf-8'))
    return str(signature, encoding='utf-8')
```



## New Order

New Order message is following in ：

```python
		...
    msg = quickfix42.NewOrderSingle()
    msg.setField(fix.ClOrdID(order_id))	# generated order id：00000015+13digt time stamp + 3 digit random number
    msg.setField(fix.Symbol(symbol)) # symbol:BTCUSD
    msg.setField(fix.HandlInst('1'))
    msg.setField(fix.OrdType(fix.OrdType_LIMIT)) # OrdType：limit
    msg.setField(fix.TransactTime())
    msg.setField(fix.Side(side))
    msg.setField(fix.Price(price))
    msg.setField(fix.Account(account_id))
    msg.setField(fix.OrderQty(qty))
    msg.setField(fix.Text("new Order"))
    quickfix.Session.sendToTarget(msg, application.sessionID)
```



## Cancel Order

Cancel order message：

```python
		...
		msg = quickfix42.OrderCancelRequest()
    msg.setField(fix.OrigClOrdID(order_id))
    msg.setField(fix.ClOrdID(order_id))
    msg.setField(fix.Symbol(symbol))
    msg.setField(fix.TransactTime())
    msg.setField(fix.Side(side))
    msg.setField(fix.Account(account_id))
    msg.setField(fix.Text("Cancel Order"))
```



## Query order status

query order message is as follwoing：
```python
		...
		msg = quickfix42.OrderStatusRequest()
    msg.setField(fix.ClOrdID(order_id))
    msg.setField(fix.Symbol(symbol))
    msg.setField(fix.Side(side))
    msg.setField(fix.Account(account_id))
```



# Others

##HTTP Error code

- HTTP *200* Normal response, if there is error it will be in the response body
- HTTP *4XX* Error code for bad request
- HTTP *403* Error code is for too many errors and ip is blocked
- HTTP *5XX*  Error code for server side errors
- HTTP *502* Error code means service is not available



## Response Error code

The error code in the response is hex integer number。

Last number is A means client side error, B means server side error.

Please don't do frequent request when error happens. If it is client side error, please correct error and resubmit  request. If it is server side error, it can retried but in low frequency, if it still has error, please contact customer support.

If too many error happens or frequent request, the ip will be banned.

| Error Code | Explanation |
| ------ | ------------------- |
| 0x01001A | general request error, please check if the parameters in request are correct. |
| 0x01002B | general server side error, retry or contact customer service |
| 0x02002A | Account doesn't exist, please verify account ID. |
| 0x02005A | Account status doesn't satisfy request, please verify account status. |
| 0x05001A | Order ID doesn't exist, please check order id. |
| 0x05002B | New order failure，retry or contact customer service |
| 0x05003A | Order doesn't exist, please change order ID |
| 0x05004B | cancel order error, need to contact customer service |
| 0x05005A | timeout for new order or other request, please wait and retry. |
| 0x05006A | too many open orders, cannot create new order, need to delete some orders and retry. |
| 0x05007A | cannot cancel order under current order status, need confirm order status and retry or abandon request |
| 0x05008A | timeout for cancel order operation, need wait and retry. |
| 0x05011A | the number of new order within 24 hours exceed limit, please contact customer service. |
| 0x05014A | the order is being cancelled, please verify order status |
| 0x05015A | order cannot be cancelled when in Pending_Submit status，please wait for status changed to submitted to cancel order.( if  order status is still pending submit after 10 minutes, please contact customer service. ） |
| 0x06002A | Not enough asset available. Please verify if there is enough asset in account. |
