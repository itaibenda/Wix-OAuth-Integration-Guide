# Wix OAuth Integration Guide

A comprehensive guide to implementing OAuth with Wix for third-party applications.

## Overview

Wix uses a unique OAuth flow called **Custom Authentication (Legacy)** that differs from standard OAuth providers. The key differences are:

1. **Two-step redirect flow**: Wix requires an intermediate redirect through your "App URL" before the final authorization
2. **Token parameter**: Wix sends a `token` parameter to your App URL that must be passed back to the installer
3. **Instance-based tokens**: Wix uses `instance_id` + `client_credentials` grant instead of traditional refresh tokens
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

## OAuth Flow Diagram

```
User clicks "Connect Wix"
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Your App Initiates Connection                           │
│  - Store pending connection info (app context, user, etc.)      │
│  - Redirect to: https://www.wix.com/installer/install?appId=X   │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Wix Installer (external)                                        │
│  - User selects which Wix site to install app on                │
│  - Wix redirects to your "App URL" with a `token` parameter     │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Your App URL Endpoint                                   │
│  - Receive `token` parameter from Wix                           │
│  - Redirect back to Wix installer with:                         │
│    • token (received from Wix)                                  │
│    • appId (your Wix app ID)                                    │
│    • redirectUrl (your final callback URL)                      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Wix Installer (external)                                        │
│  - User reviews and approves permissions                        │
│  - Wix redirects to your redirectUrl with:                      │
│    • code (authorization code)                                  │
│    • instanceId (unique site identifier)                        │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Your Callback Endpoint                                  │
│  - Use instanceId to create access token (client_credentials)   │
│  - Store instanceId securely for future token creation          │
│  - Complete the OAuth flow (close popup, redirect, etc.)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Steps

### 1. Wix App Setup (Wix Developer Dashboard)

1. Create a new app at [Wix Developer Dashboard](https://manage.wix.com/studio/custom-apps)
2. Configure OAuth settings (Develop -> OAuth):
   - **App URL**: Your endpoint that receives the initial redirect with `token` (e.g., `https://yourapp.com/wix/install`)
   - **Redirect URL**: Your final callback endpoint (e.g., `https://yourapp.com/wix/callback`)
   - Note your **App ID** and **App Secret**
3. Define required permissions in the app dashboard (Develop -> Permissions). These cannot be reduced per-request
4. Release a version of the app (Distribute -> App distribution)

### 2. Environment Configuration

```env
WIX_APP_ID=your_wix_app_id
WIX_APP_SECRET=your_wix_app_secret
```

### 3. Required Endpoints

#### 3.1 Initiate OAuth Flow

Redirect the user to the Wix installer:

```javascript
// Wix OAuth endpoints
const WIX_INSTALLER_URL = 'https://www.wix.com/installer/install';

function initiateWixOAuth(req, res) {
  const appId = process.env.WIX_APP_ID;
  
  // Store any context you need (user session, app context, etc.)
  // This could be in a session, database, or signed cookie
  req.session.pendingWixConnection = {
    userId: req.user.id,
    initiatedAt: Date.now()
  };
  
  // Redirect to Wix installer
  const installerUrl = `${WIX_INSTALLER_URL}?appId=${appId}`;
  res.redirect(installerUrl);
}
```

#### 3.2 App URL Endpoint (Receives Token from Wix)

This is the endpoint configured as "App URL" in your Wix dashboard. Wix redirects here with a `token` parameter after the user selects a site:

```javascript
function handleWixInstallInit(req, res) {
  const { token } = req.query;
  
  if (!token) {
    return res.status(400).send('Missing token from Wix');
  }
  
  // Verify you have a pending connection (optional but recommended)
  if (!req.session.pendingWixConnection) {
    return res.status(400).send('No pending connection found');
  }
  
  const appId = process.env.WIX_APP_ID;
  const callbackUrl = `${process.env.BASE_URL}/wix/callback`;
  
  // Build redirect URL back to Wix with all required parameters
  const params = new URLSearchParams({
    token: token,           // The token received from Wix - REQUIRED
    appId: appId,           // Your Wix app ID
    redirectUrl: callbackUrl // Where Wix will redirect after approval
  });
  
  const redirectUrl = `${WIX_INSTALLER_URL}?${params.toString()}`;
  res.redirect(redirectUrl);
}
```

> **Critical**: The `token` parameter received from Wix must be passed back to the installer. This token identifies the installation session and is required for the flow to complete.

#### 3.3 Final Callback Endpoint

After the user approves permissions, Wix redirects to your callback URL with `code` and `instanceId`:

```javascript
async function handleWixCallback(req, res) {
  const { code, instanceId } = req.query;
  
  if (!code || !instanceId) {
    return res.status(400).send('Missing authorization parameters');
  }
  
  try {
    // Create access token using the instanceId
    const tokenData = await createWixAccessToken(instanceId);
    
    // Store the instanceId securely - you'll need it to create new tokens
    // The instanceId is permanent for this app+site combination
    await saveWixConnection({
      instanceId: instanceId,  // Store this securely (encrypted)
      accessToken: tokenData.accessToken,
      expiresAt: tokenData.expiresAt,
      userId: req.session.pendingWixConnection?.userId
    });
    
    // Clear pending connection
    delete req.session.pendingWixConnection;
    
    // Complete the flow (close popup, redirect to success page, etc.)
    res.send(`
      <html>
        <body>
          <script>
            if (window.opener) {
              window.opener.postMessage({ status: 'success', type: 'wix-oauth' }, '*');
              window.close();
            } else {
              window.location.href = '/dashboard?connected=wix';
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
```

### 4. Token Creation Service

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
      client_id: process.env.WIX_APP_ID, // the integration's client id
      client_secret: process.env.WIX_APP_SECRET, // the integration's client secret
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

### 5. Getting Valid Access Tokens

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

### 6. Getting Site Information (Optional)

Use the [Token Info endpoint](https://dev.wix.com/docs/api-reference/app-management/oauth-2/token-info) to get information about the connected site:

```javascript
const WIX_TOKEN_INFO_URL = 'https://www.wixapis.com/oauth2/token-info';

async function getWixSiteInfo(accessToken) {
  const response = await fetch(WIX_TOKEN_INFO_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ token: accessToken })
  });
  
  if (!response.ok) {
    console.error('Failed to fetch Wix token info');
    return null;
  }
  
  const data = await response.json();
  
  return {
    siteDisplayName: data.site_display_name,
    instanceId: data.instance_id,
    // Additional fields may be available
  };
}
```

---

## Complete Express.js Example

```javascript
const express = require('express');
const session = require('express-session');

const app = express();

// Session middleware (use a proper store in production)
app.use(session({
  secret: 'your-session-secret',
  resave: false,
  saveUninitialized: false
}));

const WIX_INSTALLER_URL = 'https://www.wix.com/installer/install';
const WIX_TOKEN_URL = 'https://www.wixapis.com/oauth2/token';

// Step 1: Initiate OAuth
app.get('/wix/connect', (req, res) => {
  // Store pending connection info
  req.session.pendingWixConnection = {
    userId: req.user?.id,
    initiatedAt: Date.now()
  };
  
  const installerUrl = `${WIX_INSTALLER_URL}?appId=${process.env.WIX_APP_ID}`;
  res.redirect(installerUrl);
});

// Step 2: App URL - receives token from Wix
app.get('/wix/install', (req, res) => {
  const { token } = req.query;
  
  if (!token) {
    return res.status(400).send('Missing token');
  }
  
  const params = new URLSearchParams({
    token: token,
    appId: process.env.WIX_APP_ID,
    redirectUrl: `${process.env.BASE_URL}/wix/callback`
  });
  
  res.redirect(`${WIX_INSTALLER_URL}?${params.toString()}`);
});

// Step 3: Final callback
app.get('/wix/callback', async (req, res) => {
  const { code, instanceId } = req.query;
  
  if (!instanceId) {
    return res.status(400).send('Missing instanceId');
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
      accessToken: tokenData.access_token,
      expiresAt: new Date(Date.now() + 4 * 60 * 60 * 1000),
      userId: req.session.pendingWixConnection?.userId
    });
    
    delete req.session.pendingWixConnection;
    
    res.send('<script>window.close()</script>Connected!');
    
  } catch (error) {
    console.error('Wix OAuth error:', error);
    res.status(500).send('Authorization failed');
  }
});

app.listen(3000);
```

---

## Key Differences from Standard OAuth

| Aspect | Standard OAuth | Wix OAuth |
|--------|---------------|-----------|
| Redirect flow | Single redirect to callback | Two-step: App URL → Installer → Callback |
| Token parameter | N/A | Wix sends `token` to App URL, must pass back |
| Token refresh | Refresh token grant | `client_credentials` with `instance_id` |
| Token lifetime | Usually 1 hour | 4 hours |
| Scopes | Per-request | Fixed in app dashboard (cannot be reduced) |
| State parameter | Standard support | Not passed through initial redirect |

---

## Security Considerations

1. **Secure Instance ID Storage**: The `instance_id` is essentially a permanent credential - encrypt it at rest
2. **HTTPS Only**: All endpoints must use HTTPS
3. **Session Validation**: Verify pending connection exists before processing callbacks
4. **Token Expiration**: Always check token expiration before API calls
5. **Error Handling**: Handle `WixTokenExpiredError` gracefully - prompt user to re-authorize

---

## Troubleshooting

### "Missing token" Error
- Ensure your App URL is correctly configured in the Wix dashboard
- The App URL must exactly match what Wix redirects to

### Token Creation Fails with 401
- The `instance_id` may be invalid (user uninstalled the app)
- Verify your `WIX_APP_ID` and `WIX_APP_SECRET` are correct

### Permissions Not Working
- Remember that scopes are defined in the Wix dashboard, not per-request
- After changing permissions, existing users need to re-authorize

---

## References

- [Wix Custom Authentication Documentation](https://dev.wix.com/docs/build-apps/develop-your-app/access/authentication/custom-authentication-legacy)
- [Wix OAuth 2.0 Create Access Token](https://dev.wix.com/docs/api-reference/app-management/oauth-2/create-access-token)
- [Wix Developer Portal](https://dev.wix.com/)

