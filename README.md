# Kard Postman Collection

Welcome to the Kard Postman Collection! Use this collection for a quick and easy way to get started working in the Kard sandbox environment.

## Table of Contents
- [How it Works](#how-it-works)
- [Recommended Integration Patterns](#recommended-integration-patterns)
   - [Cardholders](https://github.com/kard-financial/kard-postman#a-cardholders)
   - [Targeted Offers](https://github.com/kard-financial/kard-postman#b-targeted-offers)
   - [Transaction CLO Matching](https://github.com/kard-financial/kard-postman#c-transaction-clo-matching)
- [Recommended User Experiences](#recommended-user-experiences)
   - [Discover a New Customer CLO](https://github.com/kard-financial/kard-postman#a-discover-a-new-customer-clo)
   - [Discover a Lapsed Customer CLO](https://github.com/kard-financial/kard-postman#b-discover-a-lapsed-customer-clo)
   - [Discover CLOs Near You (Map View)](https://github.com/kard-financial/kard-postman#c-discover-clos-near-you-map-view)   
   - [Trigger an Earned Reward Webhook](https://github.com/kard-financial/kard-postman#d-trigger-an-earned-reward-webhook)
- [User Acceptance Test Cases](https://github.com/kard-financial/kard-postman#user-acceptance-test-cases)
# How it Works

### I. Set up the Collection
- [![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/21266479-fa039121-1fff-49f1-b689-708247f1543d?action=collection%2Ffork&collection-url=entityId%3D21266479-fa039121-1fff-49f1-b689-708247f1543d%26entityType%3Dcollection%26workspaceId%3Ded9de5ad-ef96-4c9d-b038-4ff97148ed91): click button to fork the collection
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
Your application allows cardholders to link cards from other programs. As a result, transactions may originate from a variety of different card networks, so the UI must capture cardBIN and cardLastFour when the user is being created.

Code Recipe:

Creating a User with cardInfo object:
- *required:* userName OR email, referringPartnerUserId, zipCode, cardInfo
- cardInfo["issuer"]: Program Name as known to Kard, rather than the underlying Card Issuer
```
{
  "email": "testuser@test-TEST.com",
  "userName": "testUser",
  "referringPartnerUserId": "438103",
  "zipCode": "30047",
  "cardInfo": {
    "last4": "4321",
    "bin": "123456",
    "issuer": "TEST",
    "network": "VISA"
  }
}
```
*Note*: If a cardholder tries to enroll a card of a direct partner the following error message will be returned from the API:
```
"Invalid request: This card is already registered. Provide another card for user enrollment."
```

### II. Issuers
Your application supports only cards issued by your program manager. As a result, transactions will originate from known cardBINs and networks so a user can be created with or without card-level information. If creating without the cardInfo object, the user can later be updated.

Code Recipe:

Creating a User:
- *required:* userName OR email, referringPartnerUserId, zipCode
```
{
  "email": "testuser@test-TEST.com",
  "userName": "testUser",
  "referringPartnerUserId": "438103",
  "zipCode": "30047",
}
```

Add cardInfo to User:
- cardInfo["issuer"]: Program Name as known to Kard, rather than the underlying Card Issuer
- *required:* referringPartnerUserId, cardInfo
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

### III. Issuers + Aggregators
Your application supports cards issued by your program manager as well as  cards your members want to link to your program. You will be provided 1 Issuer environment (for issued cards) and 1 Aggregator environment (for linked cards), and manage the integration of these environments to your application environment.

## [B. Targeted Offers](https://docs.google.com/document/d/12LYpEv3xf6hmiKM7RW2Qbz8ZrMbeHGRIPJMTe606d_M/edit?usp=sharing)

Personalized National CLOs, based on an individual cardholder's transaction history.
1. [GET Eligible Merchants](https://developer.getkard.com/#operation/getEligibleRewardsMerchants)
2. [GET Eligible Offers](https://developer.getkard.com/#operation/getEligibleRewardsOffers)
3. [GET Eligible Locations](https://developer.getkard.com/#operation/getEligibleLocations)

Each sandbox environment is configured with the following cardholder personas:
- `sandbox-{issuerName}-new-customer`: this cardholder has no record of prior transactions at the merchant.
- `sandbox-{issuerName}-lapsed-customer`: this cardholder has prior transaction history at the merchant, but none within the last 6 months.  

These personas demonstrate targeting functionality in terms of offer discovery (the cardholder is viewing a personalized offer) and transaction matching (the cardholder transaction matches to the personalized offer). The `{issuerName}` variable is provided in the sandbox environment.json. 

## [C. Transaction CLO Matching](https://developer.getkard.com/#operation/incomingTransactionEndpoint)

The three primary patterns used to send transactions to Kard for matching processing are:
1. [Dual Message](#i-dual-message)
2. [Single Message](#ii-single-message)
3. [One Authorization to Many Settlements](#iii-one-authorization-to-many-settlements)

In order to accurately match incoming transactions, specific fields must be provided which can be found here:https://developer.getkard.com/#operation/incomingTransactionEndpoint. 

To properly ingest a matched transaction earned reward webhook, check out the section on [HMAC Signature Verification](#iv-hmac-signature-verification).
 
The following are descriptions and code recipes for each pattern.

### I. Dual Message

The most common pattern used to transmit transactions is the Dual Message system, also known as a Signature transactions. Using this system, a transaction is submitted in 2 events. The first, originating event is a temporary transaction state followed by a second event that is a final, clearing transaction state.

1. Temporary Transaction Event: APPROVED
2. Final Transaction Event: SETTLED, REVERSED, DECLINED, RETURNED*  
*special case where the originating, temporary transaction ID is not readily identifiable

#### Code Recipe: Cleared, Signature Transaction

Temporary transaction event:
- status: APPROVED
- authorizationDate timestamp
```
{
   "transactionId": "sandbox-web-313",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "mcc": "1234",
   "merchantId": "123456789101213",
   "amount": 10000,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "APPROVED",
   "authorizationDate": "2022-10-29T17:48:06.135Z"
}
```

Final transaction event:
- status: SETTLED
- authorizationDate timestamp
- settledDate timestamp
- identical transactionId as the originating APPROVED event
```
{
   "transactionId": "sandbox-web-313",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "mcc": "1234",
   "merchantId": "123456789101213",
   "amount": 10000,
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "SETTLED",
   "authorizationDate": "2022-10-29T17:48:06.135Z"
   "settledDate": "2022-10-30T17:48:06.135Z"
}
```


Code Recipe: Reversed, Signature Transaction    
*Sending Reversal infomration enables the platform's Transaction Monitoring, where Kard conducts internal fraud detection to identify suspicious behavior through abnormal transaction amounts and high volume transactions or returns per cardholder on a daily basis. We then notify Issuers of any potential fraud to be investigated if it is found.*

Temporary transaction event:
- status: APPROVED
- authorizationDate timestamp
```
{
   "transactionId": "sandbox-web-313",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "APPROVED",
   "authorizationDate": "2022-10-29T17:48:06.135Z"
}
```

Final transaction event:
- status: REVERSED
- transactionDate timestamp
- identical transactionId as the originating APPROVED event
```
{
   "transactionId": "sandbox-web-313",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "REVERSED",
   "transactionDate": "2022-10-30T17:48:06.135Z"
}
```

### II. Single Message

Another common pattern used to transmit transactions is the Single Message system, also known as a PIN debit transaction. In these transactions, the cardholder is required to enter a PIN. The PIN is validated in real-time by the bank, so a transaction submitted as a single message will be a final transaction event and the authorization and settlement dates are effectively the same.

### Code Recipe: Clearing, PIN debit Transaction
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
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "SETTLED",
   "authorizationDate": "2022-10-29T17:48:06.135Z"
   "settledDate": "2022-10-30T17:48:06.135Z"
}
```

### III. One Authorization to Many Settlements

The last pattern is one where a single authorization is followed by multiple settlement events. This pattern is generally seen when a single transaction represents multiple objects. 

For example, imagine using an e-commerce site and checking out a cart with multiple items. If these items are shipped individually, there may be multiple, subsequent settlement events.

Code Recipe: Single Auth, Multiple Settlements Transaction
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
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "APPROVED",
   "authorizationDate": "2022-10-29T17:48:06.135Z"
}
```
Final transaction event: (1 of 2)
- status: SETTLED
- authorization timestamp ("2022-10-3***0***T17:48:06.135Z")
- settledDate timestamp ("2022-10-3***0***T18:48:06.135Z")
- $75 transaction amount
```
{
   "transactionId": "sandbox-web-323",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 7500,
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "SETTLED",
   "authorizationDate": "2022-10-30T17:48:06.135Z"
   "settledDate": "2022-10-30T18:48:06.135Z"
}
```
Final transaction event: (2 of 2)
- status: SETTLED
- authorizationDate timestamp ("2022-10-3***0***T17:48:06.135Z")
- settledDate timestamp ("2022-10-3***1***T18:48:06.135Z")
- $25 transaction amount
```
{
   "transactionId": "sandbox-web-323",
   "referringPartnerUserId": "438103",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 2500,
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "Hilltop BBQ",
   "merchantId": "542814140150267",
   "merchantName": "Hilltop BBQ",
   "merchantAddrCity": "Atlanta",
   "merchantAddrState": "GA",
   "merchantAddrStreet": "123 Peachtree St",
   "status": "SETTLED",
   "authorizationDate": "2022-10-30T17:48:06.135Z"
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

Code Recipe: Authenticating, then Ingesting an Earned Reward Webhook
- `auth.js`: signature verification middleware 
- `index.js`: POST endpoint

```
# auth.js

const { createHmac } = require("crypto");
const secretKey = issuer_webhook_key; //provided in postman_environment.json
 
const verifyToken = (req, res, next) => {
   // grab HMAC signature from Notify-signature header of request
   const token = req.get("notify-signature");
 
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

### [V. Transaction Reconciliation](https://docs.google.com/document/d/1QjuDORosxzqix-8gpY_vfJnonB1r8chegRI_UV6emiU/edit?usp=sharing)
There are two standard reports that Kard issues: Daily reconciliation files and end-of-month reconciliation files. Both of these files are shared via an S3 Bucket.

- [S3 Access](https://docs.google.com/document/d/18h1rsKqD8ZTeiAlR3QISziR2uMdJ8DKW/edit): For information on how to access the S3 Bucket
- Daily Reconciliation File

   At the end of each day, a daily reconciliation file of transactions is automatically generated and shared to the S3 bucket under `/upload` by the Kard team.  All previous files are automatically moved to `/backup` for your team to reference. This file will be ideal for your team to compare against received webhooks and will act as a ledger until the EOM file is generated.
  - The file is generated at 4:30 am EST. Note that this is generated at midnight UTC. That means it'll shift by an hour when we enter daylight savings.
  - Previous files move to `/backup` when the new files are generated.  This makes sure the `/upload` directory only has one of each file.
  - file naming convention: `cardlinked-reconciliation-YYMMDD`
  - file format: `.json`
 
- End of Month (EOM) Reconciliation File

   On the 15th (or following business day in case the 15th should fall on a weekend or a holiday) each month, a monthly reconciliation file will be posted to the S3 bucket. Similar to the daily reconciliation file, the new monthly file will be uploaded to `/upload` by the Kard team and the previous files will be available in `/backup`. The EOM reconciliation report contains only `SETTLED` transactions that occurred in the previous month both `PAID_IN_FULL` and `PENDING`, as well as all `PENDING` transactions from all previous months that still have yet to be paid, and `PAID_IN_FULL` transactions from previous months that are being paid out that month.
  - Previous files move to `/backup` when the new files are generated.  This makes sure the `/upload` directory only has one of each file.
  - file naming convention: `cardlinked-reconciliation-YYMM`
  - file format: `.csv`
      
- Payouts:
   - Merchants: Merchant terms are generally Net 30 across our merchant partners, but payment can be up to Net 90.
   - Issuers: the product supports the 2 following options for payouts to end users. 
      - Immediate
         - Issuers may opt to disburse payments to their end-users immediately, or shortly after a transaction occurs. By selecting this option, the Issuer agrees to front the payment amounts to the end-users. Kard will reimburse the Issuer for these payments once Kard receives the corresponding commissions from the Merchants.
      - Withheld
         - Alternatively, Issuers may choose to withhold payments to their end-users until such time as they have received the corresponding commission payments from Kard. This option allows the Issuer to avoid fronting the payment amounts.    

[# Marqeta + Kard Integration](https://www.getkard.com/docs/marqeta-kard-integration)
## Transactions
Transaction Status Mapping

Kard currently receives the following transaction event types from Marqeta: 

| Marqeta | Kard |
| ------------- | ------------- |
| authorization | Approved |
| authorization.clearing | Settled |
| pindebit | Settled |

Note: The data mapping for the transaction events below also apply to Kard’s Earned Rewards Webhook.

| Marqeta Field  | Kard Field |
| ------------- | ------------- |
| token  | transactionId  |
| user_token  | referringPartnerUserId |
| user_transaction_time | transactionDate, authorizationDate
Note: Depending on status of transaction |
| settlement_date | settledDate |
| amount | amount |
| state | status |
| preceding_related_transaction_token | transactionId
Note: valid for transactions after the first transaction event |
| card_acceptor.mcc | mcc |
| card_acceptor.name | merchantName |
| card_acceptor.street_address | merchantAddrStreet |
| card_acceptor.city | merchantAddrCity |
| card_acceptor.state | merchantAddrState |
| card_acceptor.zip | merchantAddrZipcode |
| card.last_four | cardLastFour |
| card.pan | cardBIN
Note: Kard receives a masked PAN, only showing BIN and Last 4 |
| currency_code | currency |
| card_acceptor.mid | merchantId |
| card_acceptor.name | description |
| network_reference_id | transactionId
Note: This is relevant for specific integrations. Consult your Kard Account Manager with questions |

## FIPS
FIPS State Abbreviation mappings
FIPS codes are numbers which uniquely identify geographic areas

| FIPS State Code  | State |
| ------------- | ------------- |
| 01 | ALABAMA |
| 02 | ALASKA |
| 04 | ARIZONA |
| 05 | ARKANSAS |
| 06 | CALIFORNIA |
| 08 | COLORADO |
| 09 | CONNECTICUT |
| 10 | DELAWARE |
| 11 | DISTRICT OF COLUMBIA |
| 12 | FLORIDA |
| 13 | GEORGIA |
| 15 | HAWAII | 
| 16 | IDAHO |
| 17 | ILLINOIS |
| 18 | INDIANA |
| 19 | IOWA |
| 20 | KANSAS |
| 21 | KENTUCKY |
| 22 | LOUISIANA |
| 23 | MAINE |
| 24 | MARYLAND |
| 25 | MASSACHUSETTS |
| 26 | MICHIGAN |
| 27 | MINNESOTA |
| 28 | MISSISSIPPI |
| 29 | MISSOURI |
| 30 | MONTANA |
| 31 | NEBRASKA |
| 32 | NEVADA |
| 33 | NEW HAMPSHIRE |
| 34 | NEW JERSEY |
| 35 | NEW MEXICO |
| 36 | NEW YORK |
| 37 | NORTH CAROLINA |
| 38 | NORTH DAKOTA |
| 39 | OHIO |
| 40 | OKLAHOMA |
| 41 | OREGON |
| 42 | PENNSYLVANIA |
| 44 | RHODE ISLAND |
| 45 | SOUTH CAROLINA |
| 46 | SOUTH DAKOTA |
| 47 | TENNESSEE |
| 48 | TEXAS |
| 49 | UTAH |
| 50 | VERMONT |
| 51 | VIRGINIA |
| 53 | WASHINGTON |
| 54 | WEST VIRGINIA |
| 55 | WISCONSIN |
| 56 | WYOMING |

# Recommended User Experiences
## A. Discover a New Customer CLO
Code Recipe: 
- `GET` [Eligible Rewards Offers](https://developer.getkard.com/#operation/getEligibleRewardsOffers) Endpoint
- `referringPartnerUserId` path param: `sandbox-{issuerName}-new-customer`
```
var axios = require('axios');

var config = {
  method: 'get',
  url: 'https://test-rewards-api.getkard.com/rewards/merchant/offers/user/sandbox-{issuerName}-new-customer',
  headers: { 
    'Content-Type': 'application/json', 
    'Authorization': 'redaced_token
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

## B. Discover a Lapsed Customer CLO
Code Recipe: 
- `GET` [Eligible Rewards Offers](https://developer.getkard.com/#operation/getEligibleRewardsOffers) Endpoint
- `referringPartnerUserId` path param: `sandbox-{issuerName}-lapsed-customer`
```
var axios = require('axios');

var config = {
  method: 'get',
  url: 'https://test-rewards-api.getkard.com/rewards/merchant/offers/user/sandbox-{issuerName}-lapsed-customer',
  headers: { 
    'Content-Type': 'application/json', 
    'Authorization': 'redaced_token
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

## C. Discover CLOs Near You (Map View)
Code Recipe: 
- `GET` [Eligible Locations](https://developer.getkard.com/#operation/getEligibleLocations) Endpoint
- `referringPartnerUserId` path param: `sandbox-{issuerName}-new-customer`
- query params:
    - `includeLocal`
    - `longitude=-73.9930148`
    - `latitutde=40.74201480000001`
```
var axios = require('axios');

var config = {
  method: 'get',
  url: 'https://test-rewards-api.getkard.com/rewards/merchant/locations/user/sandbox-{issuerName}-new-customer?longitude=-73.9930148&latitude=40.74201480000001&includeLocal',
  headers: { 
    'Content-Type': 'application/json', 
    'Authorization': 'redacted_token'
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

## D. Trigger an Earned Reward Webhook
The following steps provide a demo experience from the perspective of the `sandbox-{issuerName}-new-customer` cardholder.  
Code Recipe: 
1. [Discover Eligible New Customer Offers](https://github.com/kard-financial/kard-postman#a-discover-a-new-customer-clo).  
```
    {
        "_id": "6409fa6d8a2a4300083d4143",
        "isLocationSpecific": false,
        "terms": "This offer is only valid for first-time customers.",
        "redeemableOnce": false,
        "name": "BaaS Pro Shops - New Customers",
...
        "merchant": {
            "_id": "6409f118705b8a000834f23d",
            "description": "Trusted source for all products debit, credit, crypto, and more. 
                             In business since 2011, shop online or swing by our store!",
...
            "name": "BaaS Pro Shops",
            "bannerImgUrl": "https://assets.getkard.com/public/banners/kard.jpg"
        },
        "source": "NATIONAL",
        "totalCommission": 12
    },
```
2. [Submit Eligible Transaction](#c-transaction-clo-matching).  
- `POST` [Incoming Transactions](https://developer.getkard.com/#operation/incomingTransactionEndpoint) Endpoint
- Map Rewards offer `merchant.name` to Incoming Transaction `description`
```
{
   "transactionId": "sandbox-web-313",
   "referringPartnerUserId": "sandbox-{issuerName}-new-customer",
   "cardBIN": "123456",
   "cardLastFour": "4321",
   "amount": 10000,
   "mcc": "1234",
   "merchantId": "123456789101213",
   "currency": "USD",
   "description": "BaaS Pro Shops",
   "status": "APPROVED",
   "authorizationDate": "2023-03-29T17:48:06.135Z"
}
```
3. Ingest Earned Reward Webhook.   
- `POST` [Issuer Earned Reward Webhook](https://developer.getkard.com/#operation/issuerEarnedRewardWebhook) Endpoint
- Authenticate webhook using [HMAC Signature Verification](#iv-hmac-signature-verification)
- Delight your cardholder with a notification!
- Note: Cashback/Reward does not have to be given to cardholder when notification is triggered   
![example-earned-reward-notification](https://assets-global.website-files.com/6182d563d3a1261e724c788d/64076acd79c73585ead3e520_2Q4_C0BXnWNz2ldQI6lNBOcEOlIhsPPZLwhNYmLTDy7HiuopuUDLkmbkGoi3Hv80aYOY_fvKsNg5ZJx6BtfblkEQbtyZ9jgC8xpo3OiEJxpHfz4P6BhjMyM3LwRQH78i7vY2jkakjELh5JC4ENm2ezs.png)


# User Acceptance Test Cases
**As a cardholder, I should be able to successfully:**
 - Enroll in the rewards program.
 - Unenroll from the rewards program.
 - Add a card to my profile.
 - View a list of eligible ONLINE rewards.
 - View a list of eligible INSTORE rewards.
 - View a list of eligible rewards near me.
 - View Offer Details.
 - Submit a Clearing, Dual Message Transaction.
    - Submit an APPROVED(aka AUTH) event to the Incoming Transactions Endpoint 
    - Submit a SETTLED (aka CLEARED) event to the Incoming Transactions Endpoint
 - Submit a Declined, Dual Message Transaction.
    - Submit an APPROVED(aka AUTH) event to the Incoming Transactions Endpoint 
    - Submit a DECLINED event to the Incoming Transactions Endpoint
 - Submit a Reversed, Dual Message Transaction.
   - Submit an APPROVED(aka AUTH) event to the Incoming Transactions Endpoint
   - Submit a REVERSED (aka CLEARED) event to the Incoming Transactions Endpoint
 - Submit a Single Message, PIN-debit Transaction.
   - Submit a SETTLED (aka CLEARED) event to the Incoming Transactions Endpoint
 - Submit a Refund Transaction.
   - Submit a RETURNED event to the Incoming Transactions Endpoint
 - Receive an Earned Reward Webhook push notification

**As a rewards program manager, I should be able to successfully:**
 - Consume recon files.
   - Daily
   - Monthly
 - Create an Audit Request.
 - Get an Audit Request Status.
