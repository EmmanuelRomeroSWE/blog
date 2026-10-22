+++
date = '2026-10-04T15:16:32-04:00'
draft = true
title = 'Introduction to JWTs'
+++

## Objective
Some time ago, I was chatting with a colleague about JWTs. We were investigating an issue that forced us to reason about how they work.
After some back and forth, it was obvious to me that I needed to sharpen my understanding. This blog is an effort to do that. By the end of this blog you will have a general understanding of JWTs.

## What is a JWT?
According to Google a JSON Web Token (JWT) is:
: a compact, open standard [RFC 7519](https://www.rfc-editor.org/info/rfc7519/) used to securely transmit information between parties as a JSON object.

At first, this definition felt really vague. I learned that is because a JWT is just a specification term that does not attempt to define the concrete implementation of a json web token. Instead, the specification defines and reserves a [set of claims](https://www.rfc-editor.org/info/rfc7519/#section-4.1), explains how a [JWT should be verified](https://www.rfc-editor.org/info/rfc7519/#section-7.2), and lays out the [requirements for the implementation](https://www.rfc-editor.org/info/rfc7519/#section-8). In other words, a JWT is just a standard that lays the foundation for the concrete implementation of tokens. This translates to [two concrete implementations](https://www.rfc-editor.org/info/rfc7519/#section-7.1) which we will go over in the next sections.

## JSON Web Signature (JWS)

This is the first concrete implementation of a JWT. It's the one that most developers think about whenever JWTs are discussed.

What is a JSON Web Signature (JWS)?
: A JSON Web Signature (JWS) is a standardized data structure defined by the IETF in [RFC 7515](https://www.rfc-editor.org/info/rfc7515/) that represents a digitally signed or Message Authentication Code (MAC) protected message. 

The key word in that definition is data structure. So lets focus on the anatomy (structure) of the message.


### Anatomy of a JWT:

```json
{header}.{payload}.signature
```

Before we dive into the header, payload and signature I want to call out a few things. Although I represented the payload as a JSON in thee structure above, the standard does not require it to be a JSON. In fact, it can be any arbitrary data, you can even put a picture of me there if you'd like.

### The ```{header}``` aka JOSE Header:

The header contains the information that describes the type of token and the algorithm used to sign it. There are keys (aka Header Parameters) that are strictly reserved by the standard and only one that is required.

```json
{
  "alg": "string", \\ required
  "kid": "string",
  "jwk": "JSON",
  "jku": "string",
  "typ": "string",
  "cty": "string",
  "crit": "array",
  "x5u": "string",
  "x5c": "array",
  "x5t": "string",
  "x5t#s256": "string",
}
```


The most common headers are the following:

- `alg`: is a string that represents the algorithm used to sign the token
- `kid`: is a string that represents the name of the key (usually name of public key) that the verification step will use to validate the token
- `typ`: is a string that represents the type of token being handled so that token consumers do not have to guess what type of token it is.

> **Note:** If you are interested in knowing what the other values do or when they are used, reach out and I can write a deep dive on it.

The header also supports custom header parameters like `app_version` or `gateway_route`. When using custom header parameters you have to be careful that it will not collide with other parameters that are out there, the more you can scope it the better.


### The {payload} aka JWS Header

The payload is usually a JSON object defined by the creator of the token. The JWT standard defines and reserves the following keys for the payload.

```json
{
  "iss": "string",
  "sub": "string",
  "aud": "string | array",
  "exp": "number",
  "nbf": "number",
  "iat": "number",
  "jti": "string",
}
```

All of the claims reserved by the standard are optional. Here is the description for each claim:

- iss: Issuer identifies the principal (auth server/service) that issued the JWT
- sub: Subject identifies the principal that is the subject of the JWT (ex: user id)
- aud: Audience identifies the recipients that the JWT is intended for (consuming server/service)
- exp: Expiration Time identifies the exact time on or after which the JWT must not be accepted
- nbf: Not before identifies the exact time before which the JWT must not be accepted
- iat: Issued At identifies the time at which the token was minted/issued
- jti: JWT ID is a unique identifier for an individual token

For the most part the fields are self explanatory. I will move forward to the next section to keep this blog byte sized.

> **Note:** If you are interested in a deep dive on how each field integrates with a real system, reach out and I can write a follow up.

The payload also support Public and Private claims. 

- Public Claims: A custom claim designed to be shared across multiple organizations or systems, which requires a collision-resistant name (such as a URI) or [IANA registration](https://www.iana.org/assignments/jwt) to prevent naming conflicts

```json
{
  "email": "string",
  "email_verified": "boolean",
  "name": "string",
}
```

- Private Claims: A custom claim created exclusively for internal use between a specific token issuer and consumer that agree on its meaning without needing public standardization.

```json
{
  "roles": "array",
  "tenant_id": "string",
  "is_beta_tester": "boolean",
}
```

The main distinction here is public claims require some sort of public agreement and registration whereas private claims are simply adhoc conventions established between providers and consumers of tokens.

### The signature

The signature is the cryptographic proof that secures the entire token. The main purpose of the signature is to provide integrity to the header values and the claims made in the payload. It is used to prove that the appropriate party (provider) signed the token.

Something important to highlight is that the signature is not a JSON structure but it is Base64 URL encoded (similar to the other structures in the JWS)


So how is the signature generated?
It uses the following formula

```
Signing Input = Base64URL(Header).Base64URL(Payload)
```

Then the token issuer takes this combined string and passes it through a cryptographic signing algorithm (specified in the alg header parameter) using a secret key (known as a private key). Finally, the resulting cryptographic bytes are Base64URL-encoded to become the third segment of the JWS.

Here are some common cryptographic algorithms (alg):

1. Symmetric Algorithms (aka Shared Secrets)
HS256 (HMAC using SHA-256): uses a single secret key that both the issuer and consumer of the token hold and use to verify the token. Usually used in closed, internal microservice networks.

2. Asymmetric Algorithms (Public/Private Key Pairs)
RS256 (RSA Signature with SHA-256) or ES256 (ECDSA using P-256 and SHA-256): The issuer server holds the private key and uses it to sign the token. The private key is kept secure and is only known by the issuer server. The consuming servers use the public key to verify the token. It is impossible to produce a valid token with only the public key and therefore its secure.

## Validating a JWS

### How JWS Validation Works

* **Splits** the token into header, payload, and signature by periods (`.`).
* **Decodes** the header and payload from Base64URL.
* **Inspects** the header for the algorithm (`alg`) and key ID (`kid`).
* **Recomputes** the signature using the matching public key or secret.
* **Compares** the recomputed signature against the token's attached signature.


### What IS Enforced

* **Integrity:** Guarantees the payload and header haven't been altered.
* **Authenticity:** Verifies the token was signed by the legitimate key holder.
* **Algorithm Safety:** Blocks unauthorized or dangerous algorithms (like `"none"`).
* **Temporal Checks:** Enforces expiration (`exp`) and active (`nbf`) windows *(if configured)*.
* **Source Checks:** Validates issuer (`iss`) and audience (`aud`) matches *(if configured)*.


### What IS NOT Enforced

* **Required Claims:** The base spec does not require any specific claims; an empty payload `{}` can have a valid signature.
* **Custom Claim Meaning:** Validators don't interpret application logic (like `roles` or `permissions`).
* **Revocation/Logout:** Stateless tokens remain valid until they expire, unless checked against an external blocklist.
* **Clock Skew:** Small time differences between servers will cause failures unless a grace period is explicitly programmed.

## Bringing it Full Circle (Questions)

1. When do we use custom headers vs custom payload key-value pairs?
If the data is processing instructions, use custom headers.
If the data is business logic, user attributes, assertions, or content, use custom payload.

2. Can anyone see the content in my JWS?
Yes, the header and the payload are both public in a JWS, meaning that anyone can decode the Base64 content into the valid content.

## JSON Web Encryption (JWE)

This is the second concrete implementation of a JWT. This is less discussed and used when claims are invalid.

What is a JSON Web Encryption (JWE)?
: 

Why is it that this case is the least discussed?

## Conclusion


