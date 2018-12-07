# REST authenticator

This authenticator permits authentication of Tinode users and creation of Tinode accounts using a separate process as a source of truth.

This authenticator calls a REST service by HTTP POST. A skeleton implementation of a server is provided for reference at [rest-auth](../../../rest-auth/).

Request and response payloads are formatted as JSON. Some of the request or response fields are context-dependent and may be skipped.


## Request

```js
{
  "endpoint": "auth", // string, one of the endpoints as described below.
  "secret": "Ym9iOmJvYjEyMw==", // authentication secret as provided by the client.
  "rec": {            // authentication record
    {
      "uid": "LELEQHDWbgY", // user ID, int64 base64-encoded
      "authlvl": "auth",    // authentication level
      "lifetime": "10000s", // lifetime of this record
                            // see https://golang.org/pkg/time/#Duration for format.
    	"features": 2,        // bitmap of features
    	"tags": ["email:alice@example.com"] // Tags associated with this authentication record.
    }
  }
}
```

## Response

```js
{
  "err": "internal", // string, error message in case of an error.
  "rec": {           // authentication record.
    ...              // the same as `request.rec`
  },
  "byteval": "Ym9iOmJvYjEyMw==",    // array of bytes, optional
  "ts": "2018-12-04T15:17:02.627Z", // time stamp, optional
  "boolval": true,                  // boolean value, optional
  "newacc": {        // data to use for creating a new account.
    // Default access mode
    "auth": "JRWPS",
    "anon": "N",
    "public": {...}, // user's public data, see /docs/API.md#public-and-private-fields
    "private": {...} // user's private data, see /docs/API.md#public-and-private-fields
  }
}
```

## Recognized error responses

The error is returned as json:

```json
{ "err": "error-message" }
```

See [here](../../store/types/types.go#L24) for an up to date list of supported error messages.

* "internal": database failure or other internal failure.
* "malformed": request cannot be parsed or otherwise wrong.
* "failed": authentication failed (wrong login or password, etc).
* "duplicate value": duplicate credential, i.e. attempt to create a record with a non-unique login.
* "unsupported": the operation is not supported.
* "expired": the secret has expired.
* "policy": policy violation, e.g. password too weak.
* "credentials": credentials like email or captcha must be validated.
* "not found": the object was not found.
* "denied": the operation is not permitted.

## The server must implement the following endpoints:

### `add` Add new authentication record

This endpoint requests server to add a new authentication record. This endpoint is generally used for account creation. If accounts are managed externally, it's likely to be unused and should generally return an error `"unsupported"`.

#### Sample request:
```json
{
  "endpoint": "add",
  "secret": "Ym9iOmJvYjEyMw==",
  "rec": {
    "uid": "LELEQHDWbgY",
    "lifetime": "10000s",
    "features": 2,
    "tags": ["email:alice@example.com"]
  }
}
```

#### Sample response (rec values may change):
```json
{
  "rec": {
    "uid": "LELEQHDWbgY",
    "authlvl": "auth",
    "lifetime": "5000s",
    "features": 1,
    "tags": ["email:alice@example.com", "uname:alice"]
  }
}
```

### `auth` Request for authentication

Request to authenticate a user. Client (Tinode) provides a secret, authentication server responds with a user record. If this is a very first login and the server manages the accounts, the server may return `newacc` object which will be used by client (Tinode) to create the account.
The server may optionally return a challenge as `byteval`.

#### Sample request:
```json
{
  "endpoint": "auth",
  "secret": "Ym9iOmJvYjEyMw==",
}
```

#### Sample response when the account already exists (optional challenge included):
```json
{
  "rec": {
    "uid": "LELEQHDWbgY",
    "authlvl": "auth"
  },
  "byteval": "9X6m3tWeBEMlDxlcFAABAAEAbVs"
}
```

#### Sample response when the account needs to be created by Tinode:
```js
{
  "rec": {
    "authlvl": "auth",
    "lifetime": "5000s",
    "features": 1,
    "tags": ["email:alice@example.com", "uname:alice"]
  },
  "newacc": {
    "auth": "JRWPS",
    "anon": "N",
    "public": {/* see /docs/API.md#public-and-private-fields */},
    "private": {/* see /docs/API.md#public-and-private-fields */}
  }  
}
```

### `checkunique` Checks if provided authentication record is unique.

Request is used for account creation. If accounts are managed by the server, the server should respond with an error `"unsupported"`.

#### Sample request
```json
{
  "endpoint": "checkunique",
  "secret": "Ym9iOmJvYjEyMw==",
}
```

#### Sample response
```json
{
  "boolval": true
}
```

### `del` Requests to delete authentication record.

If accounts are managed by the server, the server should respond with an error `"unsupported"`.

#### Sample request
```json
{
  "endpoint": "del",
  "rec": {
    "uid": "LELEQHDWbgY",
  },
}
```

#### Sample response
```json
{}
```


### `gen` Generate authentication secret.

If accounts are managed by the server, the server should respond with an error `"unsupported"`.

#### Sample request
```json
{
  "endpoint": "gen",
  "rec": {
    "uid": "LELEQHDWbgY",
    "authlvl": "auth",
  },
}
```

#### Sample response
```json
{
  "byteval": "9X6m3tWeBEMlDxlcFAABAAEAbVs",
  "ts": "2018-12-04T15:17:02.627Z",
}
```


### `link` Requests server to link new account ID to authentication record.

If server requested Tinode to create a new account, this endpoint is used to link the new Tinode user ID with the server's authentication record. If linking was successful, the server should respond with a non-empty json.

#### Sample request
```json
{
  "endpoint": "link",
  "secret": "Ym9iOmJvYjEyMw==",
  "rec": {
    "uid": "LELEQHDWbgY",
    "authlvl": "auth",
  },
}
```

#### Sample response
```json
{}
```


### `upd` Update authentication record.

If accounts are managed by the server, the server should respond with an error `"unsupported"`.

#### Sample request
```json
{
  "endpoint": "upd",
  "secret": "Ym9iOmJvYjEyMw==",
  "rec": {
    "uid": "LELEQHDWbgY",
    "authlvl": "auth",
  },
}
```

#### Sample response
```json
{}
```