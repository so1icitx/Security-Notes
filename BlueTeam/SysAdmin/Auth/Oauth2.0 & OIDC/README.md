# OAuth 2.0 and OpenID Connect (OIDC): 

> **Note:** This documentation is based on the technical deep-dive by Nate Barbettini (Okta). It cuts through the jargon to explain exactly how these protocols work under the hood, why they were invented, and how to implement them in various architectures.

---


##  The History: Why do we need this?

To understand OAuth, you must understand the problem it solved.

### 1. Simple Login (Forms Authentication)

In the beginning, apps used **Simple Login**. You enter a username/password, the backend checks a database, hashes the password, and drops a **session cookie** in your browser. This works for a single site but fails for distributed systems.

### 2. The "Yelp" Problem (Delegated Authorization)

Around 2006, the "Delegated Authorization" problem emerged.

* **The Scenario:** A new startup (e.g., Yelp) wants to find your friends.
* **The Anti-Pattern:** Yelp would ask for your **Gmail Email AND Gmail Password**.
* **The Execution:** Yelp would log in as you, scrape your contacts, and then (hopefully) delete your password.
* **The Risk:** You just gave the keys to your digital kingdom (Bank resets, private emails) to a third party.

**OAuth** was invented to allow a user to grant a third-party application access to *specific* data (e.g., "Read Contacts") without sharing their password.

---

##  OAuth 2.0: The Authorization Protocol

**OAuth is for Authorization (Permissions).** It is not originally for Authentication (Identity), though it was often abused for that.

### Terminology Dictionary

If you read the specs, the jargon is dense. Here is the translation:

| OAuth Term | Translation | Example |
| --- | --- | --- |
| **Resource Owner** | The User | You (sitting at the keyboard) |
| **Client** | The Application | Yelp, Spotify, a Web App |
| **Authorization Server** | The Security System | Google Accounts, Okta, Facebook Auth |
| **Resource Server** | The API | Google Contacts API, Gmail API |
| **Authorization Grant** | Proof of Consent | A temporary code proving you clicked "Yes" |
| **Redirect URI** | Callback URL | Where the user returns after logging in |
| **Access Token** | The Key | The token used to call the API |
| **Scope** | Permissions | `contacts.read`, `profile.write` |

### The Problem with "Front Channel" Security

To understand the architecture, you must distinguish between channels.

* **Front Channel (Less Secure):** The Browser. URLs, Query Parameters, Source Code. Anyone looking over your shoulder or a malicious browser extension can see this data. We **cannot** trust the browser with long-lived secrets.
* **Back Channel (Highly Secure):** Server-to-Server communication (e.g., Python backend to Google API). Encrypted, hidden from the user, highly trusted.

**The Golden Rule:** Never let the `Client Secret` or permanent access tokens touch the Front Channel if you can avoid it.

### Under the Hood: The Authorization Code Flow

This is the "Gold Standard" flow for web applications with a backend. It utilizes both channels for maximum security.

#### Step 1: The Setup

The App (Client) registers with the Auth Server and gets a `Client ID` (public) and `Client Secret` (private).

#### Step 2: The Authorization Request (Front Channel)

The User clicks "Connect with Google". The browser redirects to the Auth Server with specific query parameters:

```http
GET https://accounts.google.com/o/oauth2/v2/auth?
 response_type=code
 &client_id=YOUR_CLIENT_ID
 &redirect_uri=https://yoursite.com/callback
 &scope=contacts.read profile
 &state=random_security_string

```

* `response_type=code`: Tells the server "Don't give me the token yet, give me a temporary code."
* `scope`: The specific permissions requested.

#### Step 3: User Consent

The User logs in (if not already) and sees the consent screen: *"Yelp wants to read your contacts. Allow?"*

#### Step 4: The Callback (Front Channel)

If the user clicks "Allow", Google redirects the browser back to the `redirect_uri` with a code:

```http
GET https://yoursite.com/callback?code=AUTH_CODE_xyz123

```

* **Crucial:** This code is visible in the browser URL. If a hacker steals it, they can't use it yet because they don't have the Client Secret.

#### Step 5: The Exchange (Back Channel)

The Client's backend server takes the code and makes a POST request to Google. This does **not** happen in the browser.

```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

code=AUTH_CODE_xyz123
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET  <-- The Secret Key!
&grant_type=authorization_code
&redirect_uri=https://yoursite.com/callback

```

#### Step 6: Access Token Granted

The Auth Server verifies the code and the secret. It returns the Access Token.

```json
{
  "access_token": "ya29.a0AfH6S...",
  "expires_in": 3600,
  "scope": "contacts.read profile",
  "token_type": "Bearer"
}

```

#### Step 7: API Access

The Client uses the token to get data.

```http
GET https://people.googleapis.com/v1/people/me/connections
Authorization: Bearer ya29.a0AfH6S...

```

---

##  OpenID Connect (OIDC): The Authentication Layer

OAuth 2.0 doesn't care *who* you are; it cares *what permissions* you have.
Developers hacked OAuth to do login (e.g., "Login with Facebook"), but implementation varied wildly.

**OpenID Connect** is a thin layer on top of OAuth 2.0 that standardizes Authentication.

### The Key Differences

1. **Scope:** You add `openid` to your scope list.
2. **Artifact:** In addition to an Access Token, you get an **ID Token**.
3. **UserInfo Endpoint:** A standard endpoint to get user details.

### The ID Token (JWT)

The ID Token is a **JSON Web Token (JWT)**. It is a signed piece of data that proves the user's identity.

**Structure of a JWT:**

1. **Header:** Algorithm used (e.g., HS256).
2. **Payload (Claims):**
* `sub`: Subject (User ID).
* `name`: "Nate Barbettini".
* `exp`: Expiration time.
* `iss`: Issuer (who created this token).


3. **Signature:** A crypto-hash ensuring the token wasn't tampered with.

**The Flow with OIDC:**
The flow looks identical to the OAuth flow above, but in **Step 6**, the JSON response includes:

```json
{
  "access_token": "ya29...",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFh...",  <-- The User's ID Card
  "scope": "openid profile email"
}

```

---

---

##  Resources

* **https://www.google.com/search?q=OAuth.com:** Free eBook and exhaustive documentation.
* **https://www.google.com/search?q=OAuthDebugger.com:** A playground to test flows and see parameters in real-time.
* **JWT.io:** Decode and inspect ID Tokens.
* **https://www.google.com/search?q=Developer.okta.com:** Free developer accounts to spin up your own Authorization Server for testing.
