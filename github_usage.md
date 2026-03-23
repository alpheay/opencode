# GitHub Copilot Integration: Reverse Engineering Guide

This document explains how the GitHub Copilot integration works based on the implementation in this repository (`opencode`), specifically in `packages/opencode/src/plugin/copilot.ts` and `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts`.

This guide details the authentication flow and the specific HTTP headers required to make API requests to GitHub Copilot's models, allowing you to use it in your own custom application or harness.

## 1. Authentication: GitHub OAuth Device Flow

The integration uses the **OAuth Device Authorization Grant** flow to authenticate the user and obtain an access token.

### Key Details
- **Client ID:** `Ov23li8tweQw6odWQebz`
- **Device Code URL:** `https://github.com/login/device/code`
- **Access Token URL:** `https://github.com/login/oauth/access_token`
- **Scope:** `read:user`

*Note: For GitHub Enterprise, the domain `github.com` is replaced with the enterprise domain.*

### The Flow
1. **Initiate Device Authorization:**
   Send a `POST` request to the Device Code URL to get a user code, device code, and a verification URI.
   ```json
   {
     "client_id": "Ov23li8tweQw6odWQebz",
     "scope": "read:user"
   }
   ```
2. **User Authorization:**
   The user visits the `verification_uri` and enters the `user_code` to authorize the application.
3. **Poll for Access Token:**
   While the user is authorizing, the application polls the Access Token URL using the `device_code`.
   ```json
   {
     "client_id": "Ov23li8tweQw6odWQebz",
     "device_code": "<device_code_from_step_1>",
     "grant_type": "urn:ietf:params:oauth:grant-type:device_code"
   }
   ```
   If successful, GitHub returns an `access_token` (which is used as the refresh/access token).

## 2. API Requests: Endpoints and Required Headers

Once authenticated, you communicate with the Copilot API. The API is generally OpenAI-compatible but requires specific headers and payload structures.

### Required Headers
When sending requests to the Copilot models, the following custom headers must be injected:

- `Authorization`: `Bearer <access_token>` (The token obtained from the OAuth flow)
- `x-initiator`: `agent` or `user` (Indicates whether an AI agent or a human user initiated the request)
- `User-Agent`: e.g., `opencode/1.0.0` (A valid user agent string is required)
- `Openai-Intent`: `conversation-edits`

### Optional/Conditional Headers
- `Copilot-Vision-Request`: `true` (Required if the request payload contains images/vision content)
- `anthropic-beta`: `interleaved-thinking-2025-05-14` (Added specifically when using Claude models via the Copilot API)

*Note: The integration explicitly strips out standard `x-api-key` and `authorization` headers to replace them with the correct GitHub `Bearer` token.*

## 3. Example Implementation (TypeScript)

Here is a conceptual example of how to implement the authentication and API request flow in a custom application using Node.js/TypeScript.

```typescript
const CLIENT_ID = "Ov23li8tweQw6odWQebz";
const DEVICE_CODE_URL = "https://github.com/login/device/code";
const ACCESS_TOKEN_URL = "https://github.com/login/oauth/access_token";
const COPILOT_API_BASE = "https://api.githubcopilot.com/chat/completions"; // Approximate base URL

async function authenticateCopilot() {
  // 1. Initiate Device Flow
  const deviceRes = await fetch(DEVICE_CODE_URL, {
    method: "POST",
    headers: { "Accept": "application/json", "Content-Type": "application/json" },
    body: JSON.stringify({ client_id: CLIENT_ID, scope: "read:user" })
  });

  const deviceData = await deviceRes.json();
  console.log(`Please visit ${deviceData.verification_uri} and enter code: ${deviceData.user_code}`);

  // 2. Poll for token
  while (true) {
    await new Promise(resolve => setTimeout(resolve, deviceData.interval * 1000 + 3000)); // Sleep + safety margin

    const tokenRes = await fetch(ACCESS_TOKEN_URL, {
      method: "POST",
      headers: { "Accept": "application/json", "Content-Type": "application/json" },
      body: JSON.stringify({
        client_id: CLIENT_ID,
        device_code: deviceData.device_code,
        grant_type: "urn:ietf:params:oauth:grant-type:device_code"
      })
    });

    const tokenData = await tokenRes.json();

    if (tokenData.access_token) {
      console.log("Authentication successful!");
      return tokenData.access_token;
    }

    if (tokenData.error !== "authorization_pending") {
        console.error("Auth error:", tokenData.error);
        throw new Error("Authentication failed");
    }
  }
}

async function queryCopilot(accessToken: string, prompt: string, isAgent: boolean = false) {
  const headers = {
    "Authorization": `Bearer ${accessToken}`,
    "x-initiator": isAgent ? "agent" : "user",
    "User-Agent": "my-custom-app/1.0.0",
    "Openai-Intent": "conversation-edits",
    "Content-Type": "application/json"
  };

  const body = {
    messages: [{ role: "user", content: prompt }]
    // Depending on the model, specify model ID here
  };

  const response = await fetch(COPILOT_API_BASE, {
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
// const token = await authenticateCopilot();
// const response = await queryCopilot(token, "Write a bubble sort in Python");
// console.log(response);
```

## 4. API Emulation (OpenAI Compatible)

The repository implements an `OpenaiCompatibleProvider` (`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts`).
This means that once the authentication is complete and the correct headers are appended (as shown in the `queryCopilot` example above), the Copilot API largely accepts the standard OpenAI Chat Completions payload format (`{ messages: [...] }`).
