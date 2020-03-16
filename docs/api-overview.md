# API Introduction

The LianLianPay API is organized around REST. Our API has predictable resource-oriented URLs, accepts JSON-encoded request bodies, returns JSON-encoded responses, and uses standard HTTP response codes, authentication, and verbs.

# Versioning

When we make backwards-incompatible changes to the API, we release new versions.  The version to use is specified in the URL. For example, the current version for our EU VAT KYC api is:

```
https://global-api.lianlianpay.com/kyc/v1...
```


# Authentication

LianLian Pay API uses an API key to authenticate requests. To request Lianlian Pay developer account, send the following two information to `fx_support@lianlianpay.com`:
<!-- theme: info -->
>**developer_name**
>
>**developer_public_key**

After process, Lianlian Pay will replay with:
<!-- theme: info -->
>**developer_master_token** - api authentication
>
>**developer_refresh_token** - refresh developer master token, valid for a year. Can be updated by sending request to `fx_support@lianlianpay.com`
>
>**LianlianPay_public_key**

The `Authorization` header must be included in all requests with the following format:

```
Authorization: Basic {master_token}
```

Your API keys carry many privileges, so be sure to keep them secure! Do not share your API tokens in publicly accessible areas such as GitHub, client-side code, and so forth.

All API requests must be made over HTTPS. Calls made over plain HTTP will fail. API requests without authentication will fail.

# Preventing replay attacks

A replay attack is when an attacker intercepts a valid payload and its signature, then re-transmits them. To mitigate such attacks, LianLian Pay includes a timestamp in the **LLPAY-Signature** header for requests and for responses. Because this timestamp is part of the signed payload, it is also verified by the signature, so an attacker cannot change the timestamp without invalidating the signature. If the signature is valid but the timestamp is too old, the call will be rejected.

### Verifying signatures

The LLPAY-Signature header contains a timestamp and one or more signatures. The timestamp is prefixed by t=, and each signature is prefixed by a scheme. Schemes start with v, potentially followed by an integer.

```
LLPAY-Signature: t=1492774577, v=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
```

Note that newlines have been added in the example above for clarity, but a real LLPAY-Signature header will be all one line.

Lianlian Pay and clients shall exchange each party’s public key.  The LLPAY-Signature will be signed by with the RSA private key.

**Step 1:** Determine the signature payload

Create a `payload` string which is the following connected with `&`

* HTTP_METHOD: The actual method defined for specific API endpoint `POST`
* URI: The request's URL Path, such as `/profiles/{profileId}`
* REQUEST_EPOCH: The seconds elapsed since 1970/1/1 00:00:00 GMT, same as value of `t`
* REQUEST_PAYLOAD: The contents of request body in JSON format. 
* QUERY_STRING: The query string of the URL, URL Encoded, such as `attr1%3Dvalue1%26attr2%3Dvalue2`

<!-- theme: info -->
>Case 1: QUERY_STRING is empty:
>
>**Formula is**: HTTP_METHOD&URI&REQUEST_EPOCH&REQUEST_PAYLOAD
>
>---
>Case 2: REQUEST_PAYLOAD is empty:
>
>**Formula is**: HTTP_METHOD&URI&REQUEST_EPOCH&&QUERY_STRING
>
>---
>Case 3: REQUEST_PAYLOAD and QUERY_STRING both are empty:
>
>**Formula is**: HTTP_METHOD&URI&REQUEST_EPOCH&


Put typical GET and POST examples here.

Example POST `payload`:
```
POST&kyc/v1/profiles&19879234&{"profileId":"string","countryOfOrigin":"CN","type":"COMPANY","email":"abc@xyz.com","mobile":"string","indicator":"string","kyc":{"name":"string","idNumber":"string","idFrontCopy":"string","idBackCopy":"string","address":{"lineOne":"string","lineTwo":"string","city":"string","province":"string","region":"string","postalCode":"string"}}}
```

Example GET `payload`:
```
GET&events/v1&19879234&&amp;status%3DMAXIMUM_RETRIES_REACHED
```



**Step 2:** Prepare the `LLPAY-Signature` string

Achieve this by concatenating:
* The timestamp (Seconds elapsed since 1970/1/1 00:00:00 GMT as a string)
* Use your RSA private key and calculate the string `BASE64(RSAwithSHA256(STEP_1_RESULT)))` from Step 1

### Result Signature

LianLian Pay result headers also contain the `LLPAY-Signature` header as described above but signed with Lianlian Pay's private RSA key.  You may verify the signature corresponds to the Result body by calculating the signature and comparing it to the unencrypted version of `LLPAY-Signature` using the Lianlian Pay's public RSA key.  Here are the steps:

**Step 1:** Determine the signature `payload`

Create a `payload` string which is the following separated by `&`

* Response Timestamp: Extract t from LLPay-Signature and use it as timestamp
* Response Payload: The contents of response body in JSON format

Example `payload`:
```
19879234&{"profileId":"string","countryOfOrigin":"CN","type":"COMPANY","email":"abc@xyz.com","mobile":"string","indicator":"string","kyc":{"name":"string","idNumber":"string","idFrontCopy":"string","idBackCopy":"string","address":{"lineOne":"string","lineTwo":"string","city":"string","province":"string","region":"string","postalCode":"string"}}}
```

**Step 2:** Use RSA Verify to compare signatures

Compare the LLPay-Signature received with the `payload` calculated in Step 1 using the Lianlian Pay's Public Key using the following algorithm:

```
RSA.verify( LLPAY-Signature, 
            '19879234&{"profileId":"string","countryOfOrigin":"CN","type":"COMPANY","email":"abc@xyz.com","mobile":"string","indicator":"string","kyc":{"name":"string","idNumber":"string","idFrontCopy":"string","idBackCopy":"string","address":{"lineOne":"string","lineTwo":"string","city":"string","province":"string","region":"string","postalCode":"string"}}}', 
            Lianlian Pay Public Key )
```


# Results

#### Success

A successful API call returns a conventional HTTP response with status 2xx indicating success.  The result objects are standard REST resources.  Example:

```
{
  "profileId": "SMAPLE_PROFILE_ID",
  "countryOfOrigin": "US",
  "type": "COMPANY", 
  "email": "sample@abc.com",
  "mobile":"1234567890123",
  "kyc": {
    "name": "SAMPLE_COMPANY_ABC",
    "registrationNumber":"123456789", 
    "registrationCertificateCopyType":"BUSINESS_LICENSE",
    "registrationCertificateCopy":"SAMPLE_DOCUMENT_ID", 
    "address": {
      "lineOne": "line 1",
      "lineTwo": "line 2",
      "city": "town",
      "province": "state",
      "region": "US",
      "postalCode": "12345"
    },
    "legalRepresentative":{}
  },
  "status":"ACTIVE"
}
```

#### Errors

LianLian Pay API uses conventional HTTP response codes to indicate the success or failure of an API request. In general: Codes in the 2xx range indicate success. Codes in the 4xx range indicate an error that failed given the information provided (e.g., a required parameter was omitted, a charge failed, etc.). Codes in the 5xx range indicate an error with LianLian's servers (these are rare).

Some 4xx errors that could be handled programmatically (e.g., a validation error) include an error code that briefly explains the error reported.


### HTTP Status Table

|CODE | DESCRIPTION|
|---------|----------|
|200 | Success. |
|400 | API Error. Detailed error codes & error message are included in each API method.|
|401 | Unauthorized.	No valid API key provided.|
|403 | Forbidden.	The API key doesn't have permissions to perform the request.|
|404 | Resource Not Found.	The requested resource doesn't exist.|
|500 | Internal Server Error. Contact Lianlian Pay for technical support|

### Error Result

When a result other than 2xx is returned, an Error Object is returned with further information including Error Code and Message.  There are some general error codes that can happen across APIs but most are particular to the API and defined within that API's documentation.  

See definition of Error Model in API Reference.

#### Body

```
{
  "code": "108001",
  "message": "`profileId` already exists"
 }
 ```

# Request IDs

Each API request has an associated request identifier. You can find this value in the response headers, under `Request-Id`. If you need to contact us about a specific request, providing the request identifier will ensure the fastest possible resolution.


# Idempotent Requests

Some requests such as the payment POST request are idempotent.  When a repeat request is sent with the same Client ID, the service responds with the same success or failure response as it is assumed the client never received it to begin with.  In any repeat responses, however, a new header `Repeat-Id` is addd and contains the Request Id Of the original response.  In this way the client knows the response is a repeat - this helps in debugging issues.



