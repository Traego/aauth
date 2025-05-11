authorization
> IPR: trust200902
> Area: Security
> Version: 00
> Date: 2025-05-11
> Authors: Pat White

# Abstract

This document defines the **Agent Authorization Grant**, an OAuth 2.1 extension for **confidential agent clients**—software or AI agents with long-lived identities—to request end-user consent and obtain access tokens via HTTP polling, Server-Sent Events (SSE), or WebSocket. It is heavily inspired by the core dance of the OAuth 2.0 Device Authorization Grant (RFC 8628) but is tailored for agents either long lived identities, and introduces scoped, natural-language descriptions and a `reason` parameter provided by the agent.

# Status of This Memo

This Internet-Draft is submitted in full conformance with BCP 78 and BCP 79. Internet-Drafts are working documents of the IETF and may be updated, replaced, or obsoleted at any time. This draft will expire on 2025-11-11.

# Table of Contents

1. [Introduction](#introduction)
2. [Conventions and Requirements Language](#conventions-and-requirements-language)
3. [Terminology](#terminology)
4. [Agent Authorization Grant](#agent-authorization-grant)
   4.1. [Agent Authorization Request](#agent-authorization-request)
   4.2. [User Approval](#user-approval)
   4.3. [Token Retrieval](#token-retrieval)
   4.4. [Error Handling & Back-Off](#error-handling--back-off)
5. [Security Considerations](#security-considerations)
6. [IANA Considerations](#iana-considerations)
7. [References](#references)

## 1. Introduction

OAuth 2.1 provides a framework for delegated access via bearer tokens. In modern AI architectures, **agents**—autonomous software processes—act on behalf of users or other agents. This extension defines the **Agent Authorization Grant**, enabling agents with long lived identities to request a delegated token with human-in-the-loop consent, and to receive tokens asynchronously via polling, SSE, or WebSocket, while leveraging natural-language scope descriptions published by resource servers.

## 2. Conventions and Requirements Language

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” are to be interpreted as described in \[RFC2119].

## 3. Terminology

* **Agent**: A confidential client (software or AI) with a stable, long-lived identity (`client_id`).
* **Authorization Server (AS)**: Issues `request_code` handles, manages user consent, and issues access tokens.
* **Resource Server (RS)**: API server that protects resources via OAuth tokens and publishes human-readable scope descriptions at `/.well-known/aauth.json`.
* **request\_code**: Opaque handle returned by the AS to correlate an agent’s request with subsequent token retrieval.
* **reason**: Human-readable explanation provided by the agent, shown to the user during consent.
* **scope\_descriptions**: Natural-language text for each requested scope, fetched from the RS’s discovery document.

## 4. Agent Authorization Grant

> This flow is modeled on the OAuth 2.0 Device Authorization Grant (RFC 8628) but is tailored for **confidential agent clients** with long-lived identities, and optimized for performant delivery of tokens to the agent.

### 4.1 Agent Authorization Request

An agent requests user approval by authenticating and POSTing to `/agent_authorization`:

```http
POST /agent_authorization HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=urn:ietf:params:oauth:grant-type:agent_authorization&
scope=urn:example:resource.read urn:example:resource.write&
reason="<human-readable reason>"
```

* **grant\_type**: MUST be `urn:ietf:params:oauth:grant-type:agent_authorization`.
* **scope**: Space-delimited list of scope URIs.
* **reason**: Concise, human-readable explanation provided by the agent; MUST be echoed verbatim by the AS.

Upon receipt, the AS:

1. **Validates** client credentials.
2. **Fetches** each scope’s description from the target RS’s `https://{rs}/.well-known/aauth.json#scope_descriptions`.
3. **Returns:**

```json
{
  "request_code": "GhiJkl-QRstuVwxyz",
  "token_endpoint": "https://auth.example.com/token",
  "poll_interval": 5,
  "expires_in": 600,
  "poll_sse_endpoint": "https://auth.example.com/agent_authorization/sse",
  "poll_ws_endpoint": "wss://auth.example.com/agent_authorization/ws"
}
```

### 4.2 User Approval User Approval

The AS is fully responsible for user interaction:

1. **Initiate** the consent review flow with the user. This spec does not specify how this occurs, it could be through an app, a chat client, or some other mechanism.
2. **Display** the `reason` and each `scope_descriptions` entry to ensure the user has all the information necessary to assess the consent request.
3. **Authenticate** the user (including MFA) and, upon consent, bind approval to `request_code`.

> *Note:* The agent does **not** handle redirects or UI rendering—it passively awaits token availability.

### 4.3 Token Retrieval

After user approval, the agent obtains its access token via one of:

1. **HTTP Polling**

   ```http
   POST /token HTTP/1.1
   Host: auth.example.com
   Content-Type: application/x-www-form-urlencoded
   Authorization: Basic <base64(client_id:client_secret)>

   grant_type=urn:ietf:params:oauth:grant-type:device_code&
   device_code=GhiJkl-QRstuVwxyz
   ```

   *Pending*: `{"error":"authorization_pending"}`
   *Success*: Standard OAuth 2.x token response.

2. **Server-Sent Events (SSE)**

   ```http
   GET /agent_authorization/sse?request_code=GhiJkl-QRstuVwxyz HTTP/1.1
   Host: auth.example.com
   Authorization: Bearer <agent_jwt>
   Accept: text/event-stream
   ```

   On approval:

   ```
   event: token_response
   data: {"access_token":"<JWT>","expires_in":900,"issued_token_type":"urn:ietf:params:oauth:token-type:jwt"}
   ```

3. **WebSocket**

   ```http
   GET /agent_authorization/ws?request_code=GhiJkl-QRstuVwxyz HTTP/1.1
   Host: auth.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Protocol: aauth.agent-flow
   Authorization: Bearer <agent_jwt>
   ```

   On open:

   ```json
   {
     "type": "token_response",
     "access_token": "<JWT>",
     "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
     "expires_in": 900
   }
   ```

### 4.4 Error Handling & Back-Off

* **HTTP Polling**: MUST honor `error="slow_down"` with `Retry-After` headers; return standard OAuth error codes such as `authorization_pending`, `access_denied`, or `expired_token`.

* **Server-Sent Events (SSE)**:

  * On token response:

    ```
    event: token_response
    data: {"access_token":"<JWT>","expires_in":900,"issued_token_type":"urn:ietf:params:oauth:token-type:jwt"}
    ```
  * On user rejection:

    ```
    event: error
    data: {"error":"access_denied","error_description":"The user denied the request."}
    ```
  * On request expiration:

    ```
    event: error
    data: {"error":"expired_token","error_description":"The request_code has expired."}
    ```

* **WebSocket**:

  * On token response:

    ```json
    {"type":"token_response","access_token":"<JWT>","issued_token_type":"urn:ietf:params:oauth:token-type:jwt","expires_in":900}
    ```
  * On user rejection or expiration:

    ```json
    {"type":"error","error":"access_denied","error_description":"The user denied the request."}
    ```

    ```json
    {"type":"error","error":"expired_token","error_description":"The request_code has expired."}
    ```

* **Security**: All endpoints SHOULD enforce TLS and require client authentication.

## 5. Security Considerations. Security Considerations

* Agents MUST protect their long-lived credentials.
* Short-lived `request_code` and tokens limit replay risk.
* RS scope descriptions should not expose sensitive system details.
* Implement rate-limiting on polling and push channels to prevent abuse.

## 6. IANA Considerations

This document registers the following new OAuth grant type:

```
urn:ietf:params:oauth:grant-type:agent_authorization
```

## 7. References

### Normative References

* \[RFC2119] Bradner, "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119.
* \[RFC8628] Wahl, "OAuth 2.0 Device Authorization Grant", RFC 8628.
* \[RFC8693] Jones & Campbell, "OAuth 2.0 Token Exchange", RFC 8693.

### Informative References

* OAuth 2.1, draft-ietf-oauth-v2-1.
* OpenID Connect Core 1.0.
