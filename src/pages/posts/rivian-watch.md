---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Rivian Command Flow'
pubDate: 2024-12-05
description: 'On how to connect to the Rivian API'
image:
    url: 'khttps://docs.astro.build/assets/rose.webp'
    alt: 'The Astro logo on a dark background with a pink glow.'
tags: ["astro", "blogging", "learning in public"]
---

This documentation is a synthesized version of the unofficial [Rivian API](https://github.com/kaedenbrinkman/rivian-api)
and the (also unofficial) [Rivian Python Client](https://github.com/bretterer/rivian-python-client)

Disclaimer: this documentation is a hack job and I take no responsibility for
anything that happens to your car if you decide to use it, distribute it, etc.

## Overview
,
The tl;dr is, for a given user you must:

1. Authenticate
    1. Request CSRF Token
    2. Request Access Token

And for each vehicle that user has you have to:

2. Enlist Key (over the internet)
    1. Generate a P-256 key pair (U<sub>priv</sub> U<sub>pub</sub>)
    2. Get shared secret
        1. Send U<sub>pub</sub> to Rivian
            - They send you their public key (R<sub>pub</sub>)
        2. Perform Elliptic Point Multiplication on U<sub>priv</sub> * R<sub>pub</sub>
            - This is equal to the ECPM U<sub>pub</sub> * R<sub>priv</sub> (because math is amazing)
3. Pair Key (over bluetooth)
        
To clarify: You need to do both the "enlisting" of the key over the internet
and the "pairing" of the key over bluetooth to be able to send *any* command
to the car over the internet and BLE

- Account authentication
    - Create CSRF Token
    - Request Access Token

Let us go into each one of these steps in detail:

## Account Authentication

You need a CSRF token and an access token to be able to make any requests to
the Rivian API; We need to get those first

## Create CSRF Token

https://rivian.com/api/gql/gateway/graphql

All requests to the Rivian API are POST requests with the following headers:

```json
{
    "User-Agent": "RivianApp/707 CFNetwork/1237 Darwin/20.4.0",
    "Accept": "application/json",
    "Content-Type": "application/json",
    "Apollographql-Client-Name": "com.rivian.ios.consumer-apollo-ios"
}
```

To create a CSRF set this as the body:
 
```json
{
    "operationName": "CreateCSRFToken",
    "query": "mutation CreateCSRFToken {\n  createCsrfToken {\n    __typename\n    csrfToken\n    appSessionToken\n  }\n}",
    "variables": []
}
```

The response should look something like this:

```json
{
    "data": {
        "createCsrfToken": {
            "__typename": "CreateCsrfTokenResponse",
            "csrfToken": "<your csrf token>",
            "appSessionToken": "<your app session token>"
        }
    }
}
```

Get the csrf token and the app session token.

## Request Access Token

https://rivian.com/api/gql/gateway/graphql

### Headers

```json
{
    "a-sess": "<SESSION TOKEN>",
    "csrf-token": "<CSRF TOKEN>"
}
```

### Body

```json
{
    "operationName": "Login",
    "variables": {
        "email": "<USER EMAIL>",
        "password": "<USER PASSWORD>"
    },
    "query": "mutation Login($email: String!, $password: String!) { login(email: $email, password: $password) { __typename ... on MobileLoginResponse { accessToken refreshToken userSessionToken } ... on MobileMFALoginResponse { otpToken } } }"
}
```

### Response

If the user does not have MFA, the API will respond with:

```json
{
    "data": {
        "loginWithOTP": {
            "__typename": "MobileLoginResponse",
            "accessToken": "<your access token>",
            "refreshToken": "<your refresh token>",
            "userSessionToken": "<your user session token>"
        }
    }
}
```

Otherwise, it will respond with:

```json
{
    "data": {
        "login": {
            "__typename": "MobileMFALoginResponse",
            "otpToken": "<A OTP TOKEN>"
        }
    }
}
```

In that case, send the following request with the same headers as above:

```json
{
    "operationName": "LoginWithOTP",
    "variables": {
        "email": "<your email>",
        "otpCode": "<your otp code>",
        "otpToken": "<otp token>"
    },
    "query": "mutation LoginWithOTP($email: String!, $otpCode: String!, $otpToken: String!) { loginWithOTP(email: $email, otpCode: $otpCode, otpToken: $otpToken) { __typename accessToken refreshToken userSessionToken } }"
}
```

and the server should respond with:

```json
{
    "data": {
        "loginWithOTP": {
            "__typename": "MobileLoginResponse",
            "accessToken": "<your access token>",
            "refreshToken": "<your refresh token>",
            "userSessionToken": "<your user session token>"
        }
    }
}
```

Congrats! You're in.

## Enroll Phone/Key

## Generate a SECP256r1 (P-256) key pair

## Enlist Key 

## Pair Key
Bluetooth command:
- get HKDF derived secret key from key pair
