+++
title = 'Making Refresh Tokens Safer with Rotation and Reuse Detection'
date = 2026-05-05T20:54:36+01:00
draft = false
+++

When implementing authentication with OAuth/OIDC, it’s common to receive both an access token and a refresh token from the token endpoint.

- The **access token** is short-lived and used to authenticate API requests.
- The **refresh token** is longer-lived and used to obtain new access tokens once the original expires.

At first glance, this seems straightforward. But there’s a subtle security problem baked into this model.

### The Problem with Refresh Tokens

If a refresh token is ever stolen, it becomes a long-lived credential that an attacker can use to continuously mint new access tokens. Unlike access tokens, which expire quickly, a compromised refresh token can extend an attack indefinitely.
So the question becomes: **how do you detect and respond to refresh token theft?**

### Refresh Token Rotation

The solution is **refresh token rotation**.
Each time a client uses a refresh token to obtain a new access token, the identity provider also returns a _new_ refresh token. The previous refresh token is immediately invalidated.
This creates an important invariant:

> At any given time, only the _latest_ refresh token is valid.

### Detecting Token Reuse
This is where things get interesting.
If an attacker steals a refresh token and uses it:
- The identity provider rotates the token as expected.
- The legitimate user, still holding the now-invalid refresh token, attempts to use it.
- The provider rejects the request.

This rejection is a signal: **the token has already been used**.
That’s your breach detection mechanism.
From the application’s perspective, this likely means:
- Two parties are attempting to use the same session
- One of them is not legitimate

### Response Strategy
When reuse is detected, the safest response is simple:
- Invalidate the session
- Log the user out
- Require re-authentication

This ensures:
- The attacker loses access
- The legitimate user regains control by logging back in

It’s a small inconvenience for the user, but a strong containment strategy.

---

## Implementation Considerations

In our case, the access token was already stored in an HTTP-only cookie, but the refresh token was being returned by the token endpoint and not persisted anywhere.

To support rotation and reuse detection, we need to store and manage refresh tokens explicitly.

We evaluated three approaches.

---

### Option A: Server-Side Sessions (PostgreSQL)

Store the refresh token in a database and issue the browser an opaque session ID.

**Pros:**
- Refresh tokens never leave the server
- Sessions can be explicitly revoked (logout actually works)
- Rotation and reuse detection are handled cleanly via the identity provider

**Cons:**
- Each authenticated request requires a database lookup
- The system is no longer stateless

---

### Option B: Refresh Token in HttpOnly Cookie

Store the refresh token directly in a second secure cookie.

**Pros:**
- No server-side storage required
- Maintains a stateless architecture

**Cons:**
- Reuse detection becomes effectively impossible
- The refresh token (a high-value credential) is now exposed to the client
- Logout cannot truly invalidate the token—it remains valid at the provider

This approach breaks one of the key benefits of rotation: **knowing when something has gone wrong**.

---

### Option C: Redis / In-Memory Store

Similar to Option A, but using an in-memory store like Redis.

**Pros:**
- Faster reads compared to a traditional database

**Cons:**
- Introduces new infrastructure
- No meaningful security improvement over PostgreSQL for this use case

---

## Final Decision

We chose **server-side sessions backed by PostgreSQL**.
The deciding factor was **preserving reuse detection as a first-class security signal**.
With this approach:

- The identity provider remains the source of truth for token validity
- Any attempt to reuse an old refresh token results in a clear failure
- That failure cleanly propagates into a forced re-authentication

### Why Not Cookies?

Storing refresh tokens client-side would have undermined the entire rotation model.

If both an attacker and a legitimate user present what appear to be valid tokens, your application has no way to distinguish between them. Only the identity provider can make that call.

By keeping refresh tokens server-side, we allow the provider’s decision to flow through the system and trigger appropriate action.

---

## Closing Thought

This is one of those areas where a “stateless” architecture can quietly erode your security model.

Adding a small amount of state—just enough to track sessions and refresh tokens—unlocks a powerful capability:

> The ability to detect and respond to credential theft in real time.

And that’s a trade-off worth making.