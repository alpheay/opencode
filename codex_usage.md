# Codex Integration: Reverse Engineering Guide

This document explains how the ChatGPT Pro/Plus Codex integration works based on the implementation in this repository (`opencode`), specifically in `packages/opencode/src/plugin/codex.ts`.

This guide details everything an agent or developer needs to build a working, custom provider to utilize ChatGPT Codex models. It covers the specific device authorization flow, the required hardcoded Client ID, and the necessary HTTP headers to interact with the Codex API.

## 1. Authentication: Device Authorization Flow

The integration leverages a Device Authorization flow targeting the internal OpenAI accounts API to obtain a token using a ChatGPT Pro/Plus account.

### Key Details & The Hardcoded Client ID
- **Client ID:** `app_EMoamEEZ73f0CkXaXp7hrann`
  - **Important Note:** This Client ID is **not user-specific**. It is hardcoded into the repository. To successfully emulate this specific Codex flow and implement a working provider, you **must** use this exact Client ID (`app_EMoamEEZ73f0CkXaXp7hrann`).
- **Issuer / Auth Base URL:** `https://auth.openai.com`
- **Device Code Endpoint:** `https://auth.openai.com/api/accounts/deviceauth/usercode`
- **Token Polling Endpoint:** `https://auth.openai.com/api/accounts/deviceauth/token`
- **OAuth Exchange Endpoint:** `https://auth.openai.com/oauth/token`

### The Flow
1. **Initiate Device Authorization:**
   Send a `POST` request to the Device Code Endpoint to retrieve a `user_code`, `device_auth_id`, and `interval`.
   ```json
   {
     "client_id": "app_EMoamEEZ73f0CkXaXp7hrann"
   }
   ```
2. **User Authorization:**
   The application displays the `user_code` and directs the user to visit `https://auth.openai.com/codex/device` to enter the code.
3. **Poll for Authorization Code:**
   The application polls the Token Polling Endpoint, sending the `device_auth_id` and `user_code`:
   ```json
   {
     "device_auth_id": "<device_auth_id>",
     "user_code": "<user_code>"
   }
   ```
4. **Exchange for Access Token:**
   When the user completes the flow, the polling endpoint returns an `authorization_code` and a `code_verifier`.
   These are then exchanged for standard OAuth tokens (`access_token`, `refresh_token`, `expires_in`) via the OAuth Exchange Endpoint (`/oauth/token`) using standard `application/x-www-form-urlencoded` format.

## 2. API Requests: Endpoints and Required Headers

Once authenticated, requests are made to the internal ChatGPT backend for Codex. The payloads generally mimic standard OpenAI chat requests.

### Endpoint
- **Codex API Endpoint:** `https://chatgpt.com/backend-api/codex/responses` (or `/codex`)

### Required Headers
When sending requests to the Codex API, the following custom headers must be injected:

- `Authorization`: `Bearer <access_token>` (The access token obtained from the token exchange)
- `ChatGPT-Account-Id`: `<account_id>` *(Optional/Conditional)*. If the token claims include a `chatgpt_account_id` or an organization ID, it is explicitly passed in this header.
- `User-Agent`: e.g., `opencode/1.0.0` (Must be present for authorization requests)

*Note: Like the Copilot integration, the standard `authorization` headers attached by standard AI SDKs are stripped out and replaced with the actual Bearer token derived from the OAuth flow.*

## 3. Creating a Working Provider: Example Implementation

Here is a complete, conceptual example of how to implement the authentication and API request flow in a custom application using Node.js/TypeScript. An agent can use this directly as a blueprint to build a provider.

```typescript
const CLIENT_ID = "app_EMoamEEZ73f0CkXaXp7hrann"; // Mandatory hardcoded Client ID
const ISSUER = "https://auth.openai.com";
const CODEX_API_ENDPOINT = "https://chatgpt.com/backend-api/codex/responses";

async function authenticateCodex() {
  // 1. Initiate Device Flow using the hardcoded Client ID
  const deviceResponse = await fetch(`${ISSUER}/api/accounts/deviceauth/usercode`, {
    method: "POST",
    headers: { "Content-Type": "application/json", "User-Agent": "my-custom-app/1.0.0" },
    body: JSON.stringify({ client_id: CLIENT_ID })
  });

  const deviceData = await deviceResponse.json();
  console.log(`Please visit ${ISSUER}/codex/device and enter code: ${deviceData.user_code}`);

  // 2. Poll for token authorization
  while (true) {
    const interval = Math.max(parseInt(deviceData.interval) || 5, 1) * 1000;
    await new Promise(resolve => setTimeout(resolve, interval + 3000)); // Sleep + safety margin

    const tokenRes = await fetch(`${ISSUER}/api/accounts/deviceauth/token`, {
      method: "POST",
      headers: { "Content-Type": "application/json", "User-Agent": "my-custom-app/1.0.0" },
      body: JSON.stringify({
        device_auth_id: deviceData.device_auth_id,
        user_code: deviceData.user_code
      })
    });

    if (tokenRes.ok) {
        const data = await tokenRes.json();

        // 3. Exchange authorization code for access token
        const exchangeRes = await fetch(`${ISSUER}/oauth/token`, {
          method: "POST",
          headers: { "Content-Type": "application/x-www-form-urlencoded" },
          body: new URLSearchParams({
            grant_type: "authorization_code",
            code: data.authorization_code,
            redirect_uri: `${ISSUER}/deviceauth/callback`,
            client_id: CLIENT_ID, // Must be the hardcoded Client ID
            code_verifier: data.code_verifier,
          }).toString()
        });

        const tokens = await exchangeRes.json();
        console.log("Authentication successful!");
        return tokens.access_token; // Return the token for Bearer auth
    }

    if (tokenRes.status !== 403 && tokenRes.status !== 404) {
        throw new Error("Authentication failed");
    }
  }
}

async function queryCodex(accessToken: string, prompt: string, modelId: string = "gpt-5.3-codex", accountId?: string) {
  // Required Headers
  const headers: Record<string, string> = {
    "Authorization": `Bearer ${accessToken}`,
    "Content-Type": "application/json"
  };

  if (accountId) {
      headers["ChatGPT-Account-Id"] = accountId;
  }

  // OpenAI Compatible Payload (mostly)
  const body = {
    model: modelId, // e.g., "gpt-5.3-codex", "gpt-5.1-codex-max", etc.
    messages: [{ role: "user", content: prompt }]
  };

  const response = await fetch(CODEX_API_ENDPOINT, {
    method: "POST",
    headers,
    body: JSON.stringify(body)
  });

  if (!response.ok) {
    throw new Error(`API error: ${response.status} ${response.statusText}`);
  }

  return await response.json();
}

// Usage Example
// const token = await authenticateCodex();
// const response = await queryCodex(token, "Write a bubble sort in Python");
// console.log(response);
```

## 4. Known Supported Models

In this integration, the models are identified with specific internal IDs (prefixed with `gpt-` and suffixed with `codex`). When configuring a provider to use this reverse-engineered endpoint, you should target one of the following `modelId` parameters:

- `gpt-5.3-codex`
- `gpt-5.2-codex`
- `gpt-5.1-codex`
- `gpt-5.1-codex-max`
- `gpt-5.1-codex-mini`
- `gpt-5-codex`
