# Kard Postman Collection

Welcome to the Kard Postman Collection! Use this collection for a quick and easy way to get started working in the Kard sandbox environment.

## Table of Contents
- [How it Works](https://github.com/kard-financial/kard-postman/edit/main/README.md#how-it-works)
- [Recommended Integration Patterns](https://github.com/kard-financial/kard-postman/edit/main/README.md#recommended-integration-patterns)
- [Recommended User Experiences (Coming Soon!)](https://github.com/kard-financial/kard-postman/edit/main/README.md#recommended-user-experiences)

# How it Works

### I. Set up the Collection
- [![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/21266479-95275e3b-b758-4ac9-9224-2a71f06eee87?action=collection%2Ffork&collection-url=entityId%3D21266479-95275e3b-b758-4ac9-9224-2a71f06eee87%26entityType%3Dcollection%26workspaceId%3Ded9de5ad-ef96-4c9d-b038-4ff97148ed91): click button to fork the collection
- [Import postman_environment.json](https://learning.postman.com/docs/getting-started/importing-and-exporting-data/#importing-postman-data): provided by the Sales team, contains sensitive `clientHash` and `issuer_webhook_key` variables.

### [II. GET a Session Token](https://developer.getkard.com/#section/Authentication)

Code Recipe:
- `GET Session Token` request in collection root directory
- baseURL: `https://test-rewards-api.auth.us-east-1.amazoncognito.com`
- `{clientHash}`: base64 encoded copy of `client_id:client_secret`, provided in the postman_environment.json.
```
const axios = require('axios');

const config = {
  method: 'post',
  url: 'https://test-rewards-api.auth.us-east-1.amazoncognito.com/oauth2/token?grant_type=client_credentials',
  headers: { 
    'Content-Type': 'application/x-www-form-urlencoded', 
    'Authorization': 'Basic {clientHash}'
  }
};

axios(config)
.then(function (response) {
  console.log(JSON.stringify(response.data));
})
.catch(function (error) {
  console.log(error);
});
```
example response:
```
{
    "access_token": "jwt-info",
    "expires_in": 3600,
    "token_type": "Bearer"
}
```

### III. Call an EndPoint
Code Recipe:
- `B. Merchant Offers` folder in collection
- baseURL: `https://test-rewards-api.getkard.com`
- `{access_token}`: provided in response body of authentication call
```
const axios = require('axios');

const config = {
  method: 'get',
  url: 'https://test-rewards-api.getkard.com/rewards/merchant/offers',
  headers: { 
    'Content-Type': 'application/json', 
    'Authorization': '{access_token}'
  }
};

axios(config)
.then(function (response) {
  console.log(JSON.stringify(response.data));
})
.catch(function (error) {
  console.log(error);
});

```

# Recommended Integration Patterns

## [A. Cardholders](https://developer.getkard.com/#tag/Users)

### I. Aggregators
Your application allows cardholders to register cards from other programs. In general, the cardholder cardBIN is not known at rewards program program inception, so the UI must capture cardBIN and cardLastFour when the user is being created.

Code Recipe:

Creating a User with cardInfo object:
- *required:* userName OR email, referringPartnerUserId, cardInfo
- cardInfo["issuer"]: Program Name as known to Kard, rather than the underlying Card Issuer
```
{
  "email": "testuser@test-TEST.com",
  "userName": "testUser",
  "referringPartnerUserId": "438103",
  "cardInfo": {
    "last4": "4321",
    "bin": "123456",
    "issuer": "TEST",
    "network": "VISA"
  }
}
```

### II. Issuers
Your application supports only cards issued by your program manager. In general, the cardBIN is known at the rewards program inception so a user can be created with or without card-level information. If creating without the cardInfo object, the user can later be updated.

Code Recipe:

Creating a User:
- *required:* userName OR email, referringPartnerUserId
```
{
  "email": "testuser@test-TEST.com",
  "userName": "testUser",
  "referringPartnerUserId": "438103"
}
```

Add cardInfo to User:
- cardInfo["issuer"]: Program Name as known to Kard, rather than the underlying Card Issuer
```
{
  "referringPartnerUserId": "438103",
  "cardInfo": {
    "last4": "4321",
    "bin": "123456",
    "issuer": "TEST",
    "network": "VISA"
  }
}
```

## [B. Merchant Offers (Coming Soon!)](https://developer.getkard.com/#tag/Merchant)



## [C. Transaction CLO Matching](https://developer.getkard.com/#operation/incomingTransactionEndpoint)

The three primary patterns used to send transactions to Kard for matching processing are:
1. Dual Message
2. Single Message
3. One Authorization to Many Settlements
 
The following are descriptions and code recipes for each pattern.

### I. Dual Message

The most common pattern used to transmit transactions is the Dual Message system, also known as a Signature Debit transaction. Using this system, a transaction is submitted in 2 events. The first, originating event is a temporary transaction state followed by a second event that is a final, clearing transaction state.

1. Temporary Transaction Event: APPROVED
2. Final Transaction Event: SETTLED, REVERSED, DECLINED, RETURNED*  
*special case where the originating, temporary transaction ID is not readily identifiable

Code Recipe:

Temporary transaction event:
- status: APPROVED
- authorizationDate timestamp
```
{
   "transactionId": "sandbox-web-303",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "status": "APPROVED",
   "authorizationDate": "2022-10-29T17:48:06.135Z"
}
```

Final transaction event:
- status: SETTLED
- settledDate timestamp
- identical transactionId as the originating APPROVED event
```
{
   "transactionId": "sandbox-web-303",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "status": "SETTLED",
   "settledDate": "2022-10-30T17:48:06.135Z"
}
```

### II. Single Message

Another common pattern used to transmit transactions is the Single Message system, also known as a PIN debit transaction. In these transactions, the cardholder is required to enter a PIN. The PIN is validated in real-time by the bank, so a transaction submitted as a single message will be a final transaction event and the authorization and settlement dates are effectively the same.

Code Recipe: 
- status: SETTLED
- authorizationDate timestamp
- settledDate timestamp
```
{
   "transactionId": "sandbox-web-313",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "status": "SETTLED",
   "authorizationDate": "2022-10-30T17:48:06.135Z",   
   "settledDate": "2022-10-30T17:48:06.135Z"
}
```

### III. One Authorization to Many Settlements

The last pattern is one where a single authorization is followed by multiple settlement events. This pattern is generally seen when a single transaction represents multiple objects. 

For example, imagine using an e-commerce site and checking out a cart with multiple items. If these items are shipped individually, there may be multiple, subsequent settlement events.

Code Recipe: 
- identical referringPartnerUserId for all events
- identical transactionId for all events
- different settledDate timestamps for each SETTLED event

Temporary transaction event: (1 of 1)
- status: APPROVED
- authorizationDate timestamp
- $100 transaction amount
```
{
   "transactionId": "sandbox-web-323",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "status": "APPROVED",
   "authorizationDate": "2022-10-30T17:48:06.135Z"
}
```
Final transaction event: (1 of 2)
- status: SETTLED
- settledDate timestamp ("2022-10-3***0***T18:48:06.135Z")
- $75 transaction amount
```
{
   "transactionId": "sandbox-web-323",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 7500,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "status": "SETTLED",
   "settledDate": "2022-10-30T18:48:06.135Z"
}
```
Final transaction event: (2 of 2)
- status: SETTLED
- settledDate timestamp ("2022-10-3***1***T18:48:06.135Z")
- $25 transaction amount
```
{
   "transactionId": "sandbox-web-323",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 2500,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "status": "SETTLED",
   "settledDate": "2022-10-31T18:48:06.135Z"
}
```
### [IV. HMAC Signature Verification](https://developer.getkard.com/#operation/issuerEarnedRewardWebhook)
An issuer will be provided a webhook key for both the sandbox and production environments.

The webhook key is used to generate an HMAC of the webhook body. We calculate the HMAC and send it in the notify-signature header.

To validate the message, you should generate the HMAC with the body, key, and SHA-256 hashing algorithm, then compare it to the HMAC in the header.
 
The following is a Node.js code recipe that shows one approach to:
1. Stand up a service to ingest an earned reward webhook
2. Implement HMAC signature verification

Code Recipe: 
- `auth.js`: signature verification middleware 
- `index.js`: POST endpoint

```
# auth.js

const { createHmac } = require("crypto");
const secretKey = issuer_webhook_key; //provided in postman_environment.json
 
const verifyToken = (req, res, next) => {
   // grab HMAC signature from Notify-signature header of request
   const token = req.headers["notify-signature"];
 
   if (!token) {
       return res.status(403).send("A token is required for authentication");
   }
 
   try {
       // cast webhook as string
       const stringRequest = JSON.stringify(req.body);
      
       // hash using sha256, webhook key, and webhook body as string
       const hash = createHmac('sha256', secretKey)
       .update(stringRequest)
       .digest('base64')
 
       // verified request
       if (token === hash){
           return next();           
       }
      
       // unverified request
       return res.status(401).send("Invalid token");
   } catch (err) {
       return res.status(400).send("Bad Request");
   }
};
  
module.exports = verifyToken;
```

```
# index.js

const express = require('express');
const app = express();
const port = 3000;
const auth = require('./auth');
app.use(express.json());

app.post('/earned-rewards-webhook', auth, (req, res) => {
   try {
   // insert code that processes the webhook
   console.log('earned reward webhook: ', req.body);
   }
   catch (err) {
       res.send(err);
   }
   res.status(200).send('thanks Kard!');
});

app.listen(port, () => {
 console.log(`Example app listening on port ${port}`);
});
```

# Recommended User Experiences
Coming soon!
