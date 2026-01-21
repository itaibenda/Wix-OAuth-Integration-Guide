# Wix OAuth Integration Guide

A comprehensive guide to implementing OAuth with Wix for third-party applications.

## Table of Contents

- [Overview](#overview)
- [Understanding the Token Identity](#understanding-the-token-identity)
- [OAuth Flow (Simplified)](#oauth-flow-simplified)
  - [Flow Diagram](#oauth-flow-diagram)
  - [Implementation Steps](#implementation-steps)
  - [Complete Express.js Example](#complete-expressjs-example)
- [Token Management](#token-management)
  - [Token Creation Service](#token-creation-service)
  - [Getting Valid Access Tokens](#getting-valid-access-tokens)
  - [Getting Site Information](#getting-site-information-optional)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Legacy OAuth Flow](./LEGACY_OAUTH_FLOW.md)
- [References](#references)

---

## Overview

Wix provides a **simplified OAuth flow** that makes it easy to integrate your application with Wix sites. The key characteristics are:

1. **Single redirect flow**: Direct redirect to the callback URL after installation
2. **Dynamic callback URL**: Specify your `postInstallationUrl` at runtime (no pre-configuration needed)
3. **Instance-based tokens**: Uses `instance_id` + `client_credentials` grant for token retrieval
4. **Dashboard-defined scopes**: Permissions are defined in the Wix app dashboard, not per-request

> **Important Note**: Wix does not support reduced scopes. The user must approve all permissions defined in your app dashboard. You cannot request a subset of permissions during OAuth.

---

## Understanding the Token Identity

The access token generated through this OAuth flow represents an **App Identity**, not a user identity. This is an important distinction:

### What This Token Can Do (Management APIs)

The token allows your application to call Wix APIs **on behalf of the app** with the permissions granted during installation. This is suitable for:

- **Wix Stores**: Manage products, inventory, orders, collections
- **Wix Blog**: Create, update, and delete blog posts and categories
- **Wix CRM**: Manage contacts, segments, and workflows
- **Wix Bookings**: Manage services, staff, and bookings
- **Wix Events**: Create and manage events
- **Wix Data**: Read and write to site collections
- **Site Content**: Manage pages, menus, and site structure

### What This Token Cannot Do (Live Site Actions)

This app identity token is **not designed for live site visitor actions** such as:

- ❌ Add items to cart (visitor action)
- ❌ Complete checkout as a site visitor
- ❌ Submit forms as a visitor
- ❌ Create member-specific content
- ❌ Perform actions that require a logged-in site member context

> **Why?** Live site actions require a **Visitor** or **Member** identity, which represents an actual user browsing the Wix site. The app identity represents your application's server-to-server connection to the Wix site for management purposes.

---

## OAuth Flow (Simplified)

The simplified OAuth flow requires only **two steps**: initiate the installation and handle the callback. No intermediate redirects or token passing required!

### OAuth Flow Diagram

```
User clicks "Connect Wix"
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Your App Initiates Connection                           │
│  - Build the installer URL with your appId and postInstallationUrl
│  - Redirect to: https://www.wix.com/app-installer?appId=X&      │
│                 postInstallationUrl=YOUR_CALLBACK_URL           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Wix App Installer (external)                                    │
│  - User selects which Wix site to install app on                │
│  - User reviews and approves permissions                        │
│  - Wix redirects to your postInstallationUrl with:              │
│    • appId (your Wix app ID)                                    │
│    • tenantId (the Wix tenant/site that installed the app)      │
│    • instanceId (unique ID for this app installation)           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Your Callback Endpoint                                  │
│  - Use instanceId to create access token (client_credentials)   │
│  - Store instanceId securely for future token creation          │
│  - Complete the OAuth flow (close popup, redirect, etc.)        │
└─────────────────────────────────────────────────────────────────┘
```

### Key Advantages Over Legacy Flow

| Feature | Simplified Flow | Legacy Flow |
|---------|-----------------|-------------|
| Redirect steps | 1 (direct to callback) | 2 (App URL → Installer → Callback) |
| URL configuration in Dev Center | Not required | Required (App URL + Redirect URL) |
| State parameter support | Built-in via query params | Manual session management |
| Token passing | Not required | Required (must pass token back) |

---

## Implementation Steps

### 1. Wix App Setup (Wix Developer Dashboard)

1. Create a new app at [Wix Developer Dashboard](https://manage.wix.com/studio/custom-apps)
2. Note your **App ID** and **App Secret** (Develop → OAuth)
3. Define required permissions in the app dashboard (Develop → Permissions)
4. Release a version of the app (Distribute → App distribution)

> **Note**: Unlike the legacy flow, you do **not** need to configure App URL or Redirect URL in the dashboard!

### 2. Environment Configuration

```env
WIX_APP_ID=your_wix_app_id
WIX_APP_SECRET=your_wix_app_secret
```

### 3. Required Endpoints

#### 3.1 Initiate OAuth Flow

Redirect the user to the Wix App Installer with your callback URL:

```javascript
// Wix App Installer URL
const WIX_APP_INSTALLER_URL = 'https://www.wix.com/app-installer';

function initiateWixOAuth(req, res) {
  const appId = process.env.WIX_APP_ID;
  const callbackUrl = `${process.env.BASE_URL}/wix/callback`;
  
  // You can include any state information in the callback URL
  // This is useful for tracking the user or session that initiated the flow
  const state = encryptState({
    userId: req.user.id,
    initiatedAt: Date.now(),
    returnTo: req.query.returnTo || '/dashboard'
  });
  
  // Build the callback URL with your state
  const postInstallationUrl = new URL(callbackUrl);
  postInstallationUrl.searchParams.set('state', state);
  
  // Build the installer URL
  const installerUrl = new URL(WIX_APP_INSTALLER_URL);
  installerUrl.searchParams.set('appId', appId);
  installerUrl.searchParams.set('postInstallationUrl', postInstallationUrl.toString());
  
  res.redirect(installerUrl.toString());
}

// Helper function to encrypt state (implement your own encryption)
function encryptState(data) {
  // Use a proper encryption library in production (e.g., crypto-js, jose)
  return Buffer.from(JSON.stringify(data)).toString('base64url');
}
```

#### 3.2 Callback Endpoint

After the user approves permissions, Wix redirects to your `postInstallationUrl` with additional parameters:

```javascript
async function handleWixCallback(req, res) {
  const { appId, tenantId, instanceId, state } = req.query;
  
  if (!instanceId) {
    return res.status(400).send('Missing instanceId from Wix');
  }
  
  // Decrypt and validate your state (if you included one)
  let stateData = null;
  if (state) {
    try {
      stateData = decryptState(state);
      // Optionally validate state (check timestamp, etc.)
    } catch (error) {
      return res.status(400).send('Invalid state parameter');
    }
  }
  
  try {
    // Create access token using the instanceId
    const tokenData = await createWixAccessToken(instanceId);
    
    // Store the instanceId securely - you'll need it to create new tokens
    // The instanceId is permanent for this app+site combination
    await saveWixConnection({
      instanceId: instanceId,        // Store this securely (encrypted)
      tenantId: tenantId,            // The Wix site/tenant ID
      appId: appId,                  // Your app ID (for reference)
      accessToken: tokenData.accessToken,
      expiresAt: tokenData.expiresAt,
      userId: stateData?.userId      // From your encrypted state
    });
    
    // Complete the flow (close popup, redirect to success page, etc.)
    const returnTo = stateData?.returnTo || '/dashboard?connected=wix';
    
    res.send(`
      <html>
        <body>
          <script>
            if (window.opener) {
              window.opener.postMessage({ status: 'success', type: 'wix-oauth' }, '*');
              window.close();
            } else {
              window.location.href = '${returnTo}';
            }
          </script>
          <p>Connected successfully! You can close this window.</p>
        </body>
      </html>
    `);
    
  } catch (error) {
    console.error('Wix OAuth error:', error);
    res.status(500).send('Failed to complete Wix authorization');
  }
}

// Helper function to decrypt state
function decryptState(encrypted) {
  // Use a proper decryption library in production
  return JSON.parse(Buffer.from(encrypted, 'base64url').toString());
}
```

### Callback Parameters

When Wix redirects to your `postInstallationUrl`, it appends the following query parameters:

| Parameter | Description |
|-----------|-------------|
| `appId` | Your Wix application ID |
| `tenantId` | The Wix tenant (site) that installed the app |
| `instanceId` | A unique ID for this app installation, used to retrieve access tokens |

Additionally, any query parameters you included in your `postInstallationUrl` will be preserved (e.g., your `state` parameter).

---

## Complete Express.js Example

```javascript
const express = require('express');

const app = express();

const WIX_APP_INSTALLER_URL = 'https://www.wix.com/app-installer';
const WIX_TOKEN_URL = 'https://www.wixapis.com/oauth2/token';

// Step 1: Initiate OAuth
app.get('/wix/connect', (req, res) => {
  const appId = process.env.WIX_APP_ID;
  const callbackUrl = `${process.env.BASE_URL}/wix/callback`;
  
  // Include state in the callback URL
  const state = Buffer.from(JSON.stringify({
    userId: req.user?.id,
    initiatedAt: Date.now()
  })).toString('base64url');
  
  const postInstallationUrl = `${callbackUrl}?state=${state}`;
  
  const installerUrl = new URL(WIX_APP_INSTALLER_URL);
  installerUrl.searchParams.set('appId', appId);
  installerUrl.searchParams.set('postInstallationUrl', postInstallationUrl);
  
  res.redirect(installerUrl.toString());
});

// Step 2: Handle callback
app.get('/wix/callback', async (req, res) => {
  const { appId, tenantId, instanceId, state } = req.query;
  
  if (!instanceId) {
    return res.status(400).send('Missing instanceId');
  }
  
  // Decode state
  let stateData = {};
  if (state) {
    try {
      stateData = JSON.parse(Buffer.from(state, 'base64url').toString());
    } catch (e) {
      console.error('Failed to decode state:', e);
    }
  }
  
  try {
    // Create access token
    const tokenResponse = await fetch(WIX_TOKEN_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'client_credentials',
        client_id: process.env.WIX_APP_ID,
        client_secret: process.env.WIX_APP_SECRET,
        instance_id: instanceId
      })
    });
    
    const tokenData = await tokenResponse.json();
    
    // Store connection (implement your own storage)
    await saveConnection({
      instanceId,
      tenantId,
      accessToken: tokenData.access_token,
      expiresAt: new Date(Date.now() + 4 * 60 * 60 * 1000),
      userId: stateData.userId
    });
    
    res.send('<script>window.close()</script>Connected!');
    
  } catch (error) {
    console.error('Wix OAuth error:', error);
    res.status(500).send('Authorization failed');
  }
});

app.listen(3000);
```

---

## Token Management

### Token Creation Service

Wix uses the `client_credentials` grant with `instance_id`. [This is the modern OAuth 2.0 approach](https://dev.wix.com/docs/api-reference/app-management/oauth-2/create-access-token) - no authorization code exchange or refresh tokens needed:

```javascript
const WIX_TOKEN_URL = 'https://www.wixapis.com/oauth2/token';
const ACCESS_TOKEN_TTL_HOURS = 4; // Wix tokens are valid for 4 hours

async function createWixAccessToken(instanceId) {
  const response = await fetch(WIX_TOKEN_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      grant_type: 'client_credentials',
      client_id: process.env.WIX_APP_ID,      // Your app's client ID
      client_secret: process.env.WIX_APP_SECRET, // Your app's client secret
      instance_id: instanceId
    })
  });
  
  if (response.status === 400 || response.status === 401) {
    // Instance is invalid or app was disconnected from the site
    throw new WixTokenExpiredError('Wix connection invalid, re-authorization required');
  }
  
  if (!response.ok) {
    throw new Error(`Token creation failed: ${await response.text()}`);
  }
  
  const data = await response.json();
  
  if (!data.access_token) {
    throw new Error('Missing access token in response');
  }
  
  // Calculate expiration (4 hours from now)
  // Can use data.expires_in from response
  const expiresAt = new Date(Date.now() + ACCESS_TOKEN_TTL_HOURS * 60 * 60 * 1000);
  
  return {
    accessToken: data.access_token,
    expiresAt: expiresAt
  };
}

// Custom error class
class WixTokenExpiredError extends Error {
  constructor(message) {
    super(message);
    this.name = 'WixTokenExpiredError';
  }
}
```

### Getting Valid Access Tokens

Since Wix tokens expire after 4 hours, you should check expiration and create new tokens as needed:

```javascript
async function getValidAccessToken(storedConnection) {
  // Check if cached token is still valid (with 5-minute buffer)
  const bufferMs = 5 * 60 * 1000;
  const now = new Date();
  
  if (storedConnection.accessToken && storedConnection.expiresAt) {
    const expiresAt = new Date(storedConnection.expiresAt);
    if (now < new Date(expiresAt.getTime() - bufferMs)) {
      return storedConnection.accessToken;
    }
  }
  
  // Token expired or missing - create a new one using stored instanceId
  console.log('Creating new Wix access token...');
  
  try {
    const newTokenData = await createWixAccessToken(storedConnection.instanceId);
    
    // Update stored connection with new token
    await updateStoredConnection(storedConnection.id, {
      accessToken: newTokenData.accessToken,
      expiresAt: newTokenData.expiresAt
    });
    
    return newTokenData.accessToken;
    
  } catch (error) {
    if (error instanceof WixTokenExpiredError) {
      // The instanceId is no longer valid - user needs to re-authorize
      await markConnectionAsExpired(storedConnection.id);
      throw error;
    }
    throw error;
  }
}
```

### Getting Site Information (Optional)

Use the [Site Properties API](https://dev.wix.com/docs/api-reference/business-management/site-properties/properties/get-site-properties) to get information about the connected site:

```javascript
const WIX_SITE_PROPERTIES_URL = 'https://www.wixapis.com/site-properties/v4/properties';

async function getWixSiteInfo(accessToken) {
  const response = await fetch(WIX_SITE_PROPERTIES_URL, {
    method: 'GET',
    headers: {
      'Authorization': accessToken
    }
  });
  
  if (!response.ok) {
    console.error('Failed to fetch Wix site properties');
    return null;
  }
  
  const data = await response.json();
  
  return {
    siteDisplayName: data.properties?.siteDisplayName,
    locale: data.properties?.locale,
    language: data.properties?.language,
    // Additional fields available: email, phone, fax, address, categories, etc.
  };
}
```

---

## Security Considerations

1. **Secure Instance ID Storage**: The `instance_id` is essentially a permanent credential - encrypt it at rest
2. **HTTPS Only**: All endpoints must use HTTPS
3. **State Parameter Encryption**: If passing sensitive data in the state, encrypt it properly
4. **Token Expiration**: Always check token expiration before API calls
5. **Error Handling**: Handle `WixTokenExpiredError` gracefully - prompt user to re-authorize

---

## Troubleshooting

### Token Creation Fails with 401
- The `instance_id` may be invalid (user uninstalled the app)
- Verify your `WIX_APP_ID` and `WIX_APP_SECRET` are correct

### Missing instanceId in Callback
- Ensure you're using the correct installer URL: `https://www.wix.com/app-installer`
- Verify the `postInstallationUrl` is properly URL-encoded

### Permissions Not Working
- Remember that scopes are defined in the Wix dashboard, not per-request
- After changing permissions, existing users need to re-authorize

### State Parameter Lost
- Ensure your state is properly URL-encoded when adding to `postInstallationUrl`
- Check that your callback correctly parses both Wix parameters and your custom parameters

---

## Legacy OAuth Flow

For existing integrations using the legacy Custom Authentication flow (two-step redirect with token passing), see the dedicated documentation:

📄 **[Legacy OAuth Flow Documentation](./LEGACY_OAUTH_FLOW.md)**

The legacy flow uses `https://www.wix.com/installer/install` and requires:
- Configuring App URL and Redirect URL in the Wix Dev Center
- Handling an intermediate redirect with a `token` parameter
- Manual session management for state

---

## References

- [Wix OAuth 2.0 Create Access Token](https://dev.wix.com/docs/api-reference/app-management/oauth-2/create-access-token)
- [Wix Custom Authentication Documentation (Legacy)](https://dev.wix.com/docs/build-apps/develop-your-app/access/authentication/custom-authentication-legacy)
- [Wix Developer Portal](https://dev.wix.com/)
