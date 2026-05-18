+++
title = 'One Login Is Not Enough'
date = 2026-05-07T14:13:18Z
+++

One Login Is Not Enough: Adding Multi-Provider OIDC to Kaleb

The Problem

Kaleb's original authentication was exactly what you'd expect from a project where auth was "milestone 1": one OIDC provider, hard-coded through five environment variables,
lazily initialised on the first request, and storing raw tokens directly in the session row.

That design made sense to ship quickly. It stopped making sense the moment we needed a second provider.

The specific pain points:

No concept of "identity". The sessions table stored user_sub, id_token, refresh_token, and id_token_expiry. The session was the identity. Adding a second provider meant     
either a second set of those same columns (bad) or a completely different sessions table per provider (worse).

Secrets in environment variables. OIDC_CLIENT_ID, OIDC_CLIENT_SECRET, OIDC_ISSUER_URL — any second provider would require new env vars with different names, with no         
principled convention for naming them.

Lazy initialisation was a footgun. The handler called h.init() on every request until OIDC discovery succeeded. This meant the server could start and appear healthy while   
silently refusing to authenticate anyone. With multiple providers that failure mode gets worse — one bad provider would leave users guessing which one was broken.

Session cookies held UUIDs. The kaleb_session cookie value was the raw session UUID. If an attacker read the DB (SQL injection, backups without encryption at rest), they'd  
find a column that could be replayed directly into the cookie.

No account linking. A user who has both a personal and a work IdP account had no path to merge them. Two providers would mean two separate Kaleb accounts.
   
---                                                                                                                                                                          
What We Considered

Option 1: One sessions table row per provider

Add a provider column to sessions, keep everything else the same. Simple migration, minimal code change.

Why we rejected it: It doesn't solve account linking. Two sessions from two providers are still two unrelated users. The sessions table continues to store raw tokens, which
is a security smell. We'd be back here in six months.

Option 2: Single provider_accounts join table

Add provider_accounts(user_id, provider, sub) and keep the session storing tokens directly.

This gets us closer — providers are associated with users — but tokens are still stored in plaintext on the session, and every refresh still has to update the session row   
rather than a dedicated token store.

Option 3: Full identity layer (what we built)

Introduce identities(provider, issuer, subject) as the stable identity anchor, identity_tokens(identity_id, id_token, encrypted_refresh_token) as the token store, and       
restructure sessions to reference an identity and store only a hashed opaque token.

┌───────────────────────────┬───────────────────────────────────────────────┐
│                           │                    Detail                     │
├───────────────────────────┼───────────────────────────────────────────────┤
│ Implementation complexity │ High                                          │
├───────────────────────────┼───────────────────────────────────────────────┤
│ Runtime performance       │ +1 DB round trip per auth'd request           │
├───────────────────────────┼───────────────────────────────────────────────┤                                                                                                
│ Operational burden        │ New encryption key to manage                  │
├───────────────────────────┼───────────────────────────────────────────────┤                                                                                                
│ Reversibility             │ Moderate — schema change, but well-structured │
└───────────────────────────┴───────────────────────────────────────────────┘

We chose this. The extra DB round trip per request is real, but the security properties are worth it and the read path is indexed.
   
---                                                                                                                                                                          
The Architecture

Before

graph TD
subgraph Config                                                                                                                                                          
ENV[OIDC_ISSUER_URL\nOIDC_CLIENT_ID\nOIDC_CLIENT_SECRET]
end                                                                                                                                                                      
subgraph Session
S[sessions\nuser_sub | id_token | refresh_token | id_token_expiry]                                                                                                   
end                                                                                                                                                                      
subgraph Cookie
C[kaleb_session = session UUID]                                                                                                                                      
end                                                                                                                                                                      
ENV --> Handler
Handler -->|lazy init on first request| Provider[Single OIDC Provider]                                                                                                   
Handler --> S                                                                                                                                                            
C -->|UUID lookup| S

After

graph TD                                                  
subgraph Config
YAML[providers.yaml\nname, issuer_url, scopes]
ENVK[TOKEN_ENCRYPTION_KEY\nOIDC_CLIENT_ID env refs]                                                                                                                  
end                                                                                                                                                                      
subgraph Registry                                                                                                                                                        
R[auth.Registry\nmap provider_name → ProviderRecord\nbuilt at startup]                                                                                               
end                                                                                                                                                                      
subgraph Schema
U[users]                                                                                                                                                             
I[identities\nprovider | issuer | subject]        
IT[identity_tokens\nid_token | encrypted_refresh]                                                                                                                    
S[sessions\ntoken_hash | user_id | identity_id | expires_at]                                                                                                         
end                                                                                                                                                                      
subgraph Cookie                                                                                                                                                          
C[kaleb_session = random token]                                                                                                                                      
end                                                   
YAML --> Registry
ENVK --> Registry
Registry --> Handler
Handler --> U                                                                                                                                                            
Handler --> I
Handler --> IT                                                                                                                                                           
C -->|SHA-256| S                                      
S -->|identity_id| IT

Token refresh flow (per request)

sequenceDiagram                                           
participant Browser
participant Middleware as JWTMiddleware                                                                                                                                  
participant SessionRepo
participant IdentityTokenRepo                                                                                                                                            
participant IdentityRepo                                                                                                                                                 
participant OIDCProvider

      Browser->>Middleware: GET /grpc/... (cookie: raw_token)                                                                                                                  
      Middleware->>SessionRepo: GetByTokenHash(sha256(raw_token))
      SessionRepo-->>Middleware: Session{user_id, identity_id, expires_at}                                                                                                     
      Middleware->>IdentityTokenRepo: GetByIdentityID(identity_id)
      IdentityTokenRepo-->>Middleware: IdentityToken{id_token, encrypted_refresh, expiry}                                                                                      
      alt ID token not expired                                                                                                                                                 
          Middleware->>OIDCProvider: Verify(id_token)
          OIDCProvider-->>Middleware: ok                                                                                                                                       
      else ID token expired                                 
          Middleware->>Middleware: Decrypt(encrypted_refresh)                                                                                                                  
          Middleware->>OIDCProvider: Refresh(refresh_token)                                                                                                                    
          OIDCProvider-->>Middleware: new_id_token + new_refresh_token
          Middleware->>IdentityTokenRepo: Upsert(identity_id, new_id_token, encrypt(new_refresh))                                                                              
      end                                                                                                                                                                      
      Middleware->>Next: context{user_id}                                                                                                                                      

Account linking flow

sequenceDiagram                                                                                                                                                              
participant Browser                                   
participant Handler
participant IdentityRepo
participant IdentityTokenRepo                                                                                                                                            
participant OIDCProvider2

      Browser->>Handler: GET /auth/link/second (session cookie)                                                                                                                
      Handler->>Handler: pkceStore[state] = {providerName: "second", linkUserID: <current_user_id>}
      Handler->>Browser: Redirect to provider2/authorize                                                                                                                       
      Browser->>OIDCProvider2: Auth flow                                                                                                                                       
      OIDCProvider2->>Browser: /auth/second/callback?code=...                                                                                                                  
      Browser->>Handler: GET /auth/second/callback                                                                                                                             
      Handler->>IdentityRepo: FindByProviderSub(provider, issuer, sub)
      alt identity already linked to THIS user                                                                                                                                 
          Handler-->>Browser: 400 already_linked                                                                                                                               
      else identity linked to DIFFERENT user                                                                                                                                   
          Handler-->>Browser: 409 conflict                                                                                                                                     
      else new identity                                                                                                                                                        
          Handler->>IdentityRepo: Create(linkUserID, provider, issuer, sub)                                                                                                    
          Handler->>IdentityTokenRepo: Upsert(identity_id, id_token, encrypt(refresh))
          Handler-->>Browser: redirect /                                                                                                                                       
      end                                                   
                                                                                                                                                                               
---                                                       
The Implementation

Provider configuration moves to YAML

The old internal/config/oidc.go is gone. In its place is providers.yaml and internal/config/providers.go.

# providers.yaml
providers:                                                                                                                                                                   
- name: mock
issuer_url: http://mock-oauth-server:8081/default                                                                                                                        
internal_issuer_url: http://mock-oauth-server:8080/default                                                                                                               
client_id_env: OIDC_CLIENT_ID
client_secret_env: OIDC_CLIENT_SECRET                                                                                                                                    
scopes: [openid, profile, email, offline_access]      
email_trust: false                                                                                                                                                       
enabled: true

Secrets are never in the YAML file. Each provider declares which environment variable holds its credential. The config loader reads the YAML, then injects from env:

func injectSecrets(p *ProviderConfig) {                                                                                                                                      
p.clientID = os.Getenv(p.ClientIDEnv)                 
p.clientSecret = os.Getenv(p.ClientSecretEnv)                                                                                                                            
}

clientID and clientSecret are unexported fields. The YAML tags don't exist for them. This isn't security-by-obscurity — it's making it structurally impossible to            
accidentally serialize credentials back into config.

Eager discovery at startup

The old handler called h.init() on every request until the first successful OIDC discovery. We replaced this with auth.BuildRegistry at startup time:

registry, err := auth.BuildRegistry(ctx, providerCfgs, serverConfig.BaseURL)                                                                                                 
if err != nil {                                                                                                                                                              
return fmt.Errorf("build OIDC registry: %w", err)
}

If any enabled provider's OIDC discovery endpoint is unreachable at startup, the server refuses to start. This is the correct failure mode for a stateless web server —      
better a clear startup error than runtime auth failures that look like database issues.

The identities table

The core data model change. Previously the session stored user_sub — the OIDC sub claim from the provider. That is both provider-specific and tied to the session's lifetime.

CREATE TABLE identities (                                                                                                                                                    
id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),                                                                                                                   
user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
provider   TEXT NOT NULL,                                                                                                                                                
issuer     TEXT NOT NULL,                             
subject    TEXT NOT NULL,                                                                                                                                                
UNIQUE(provider, issuer, subject)
);

The (provider, issuer, subject) triple is the stable identity anchor. issuer is the OIDC issuer URL from the ID token — not just a name we assign, but what the provider     
itself claims. This matters because two providers can both issue sub = "12345" without collision.

Session restructure: store a hash, not a value

Previously: session cookie = session UUID. Anyone with read access to the sessions table could forge a valid cookie.

Now:

rawCookieToken, _ := generateRandomToken()                                                                                                                                   
h.sessionRepo.Create(ctx, hashToken(rawCookieToken), userID, identityID, expiresAt)

http.SetCookie(w, &http.Cookie{                                                                                                                                              
Name:  sessionCookie,                                                                                                                                                    
Value: rawCookieToken,                                
})

The cookie holds the plaintext token. The database stores SHA-256(token). Reading the database doesn't give you the cookie value. This is the same pattern used by Django's  
session framework and GitHub's personal access tokens.

func hashToken(token string) string {                     
sum := sha256.Sum256([]byte(token))
return hex.EncodeToString(sum[:])
}

Encrypted refresh tokens

Refresh tokens are long-lived credentials that can obtain new access. Storing them plaintext is indefensible. We added AES-256-GCM encryption:

// EncryptToken encrypts plaintext with AES-256-GCM.                                                                                                                         
// Returns a base64url-encoded string of nonce||ciphertext.                                                                                                                  
func EncryptToken(plaintext string, key []byte) (string, error) {                                                                                                            
block, _ := aes.NewCipher(key)                                                                                                                                           
gcm, _ := cipher.NewGCM(block)                                                                                                                                           
nonce := make([]byte, gcm.NonceSize())                                                                                                                                   
io.ReadFull(rand.Reader, nonce)
ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)                                                                                                             
return base64.URLEncoding.EncodeToString(ciphertext), nil                                                                                                                
}

GCM provides authenticated encryption — a tampered ciphertext returns an error at decrypt time, not garbled plaintext. The nonce is prepended to the ciphertext and stored   
with it; there's no nonce management overhead.

The encryption key is a 32-byte value loaded from TOKEN_ENCRYPTION_KEY (base64-encoded), validated at startup:

if len(key) != 32 {                                                                                                                                                          
return nil, fmt.Errorf("TOKEN_ENCRYPTION_KEY must decode to exactly 32 bytes (got %d)", len(key))
}

32 bytes = AES-256. The check is explicit because a 16-byte key silently gives you AES-128, and we don't want that to happen accidentally.

Account linking with email collision detection

When a user logs in via a new provider for the first time, we check whether their verified email address matches an existing account — but only when the provider is         
explicitly configured to be trusted (email_trust: true):

if record.Config.EmailTrust && claims.EmailVerified && claims.Email != "" {
_, emailErr := h.userRepo.GetByEmail(r.Context(), claims.Email)                                                                                                          
if emailErr == nil {                                                                                                                                                     
http.Redirect(w, r, "/login?error=link_required&provider="+entry.providerName, http.StatusFound)                                                                     
return                                                                                                                                                               
}                                                     
}

This redirects the user to the login page with an error rather than silently creating a duplicate account. The email_trust flag is off by default — email claims from most   
providers are not verified in a way we'd want to rely on for account merging.

Unlinking an identity enforces two constraints in the repository layer:

func (r *identityRepository) Delete(ctx context.Context, id uuid.UUID) error {                                                                                               
// ...                                                                                                                                                                   
count, _ := r.queries.CountIdentitiesByUserID(ctx, userID)
if count <= 1 {                                                                                                                                                          
return ErrCannotDeleteLastIdentity                                                                                                                                   
}                                                                                                                                                                        
// ...                                                                                                                                                                   
}

A user cannot unlink their last identity — that would make the account permanently inaccessible.

Context carries user ID, not the token

The old middleware put an *oidc.IDToken into the request context. Every handler that wanted the user's ID had to call idToken.Claims(&c) and inspect c.Sub. The new          
middleware puts a uuid.UUID — the user_id from the session row — directly into context:

ctx := context.WithValue(r.Context(), UserIDKey, session.UserID)

Handlers now call auth.UserIDFromContext(r.Context()). No token parsing at the handler layer, no OIDC coupling outside the auth package.
                                                                                                                                                                               
---                                                                                                                                                                          
Tradeoffs & What We Gave Up

+1 DB round trip per authenticated request. The middleware now reads: session (by token hash), identity token (by identity_id), identity (by identity_id). That's up from one
read. We have appropriate indexes (idx_sessions_token_hash, idx_identities_user_id) but under heavy load this will be visible in the query plan.

The encryption key is now a required runtime secret. Rotating it requires re-encrypting every refresh_token_ciphertext in identity_tokens. There's no key versioning here —  
if you rotate the key, old rows become unreadable until re-encrypted. This is a known limitation.

Lazy OIDC discovery is gone. If you deploy and an OIDC provider's discovery endpoint is down, the server won't start. This is intentional, but it means you can't do rolling
deploys if providers are flaky. In practice this has been a non-issue with any well-operated IdP.

The email_trust flag is a footgun if misunderstood. Setting email_trust: true for a provider that does not enforce email verification could allow account takeover. The      
default is false and the YAML comment warns about it, but there's no machine-readable enforcement.

PKCE state lives in-memory. The pkceStore is a map[string]pkceEntry guarded by a mutex. In a multi-instance deployment, the auth flow must land on the same instance for both
the login redirect and the callback. There's a sweep that removes expired entries, but no distributed store. Single-instance for now; this would need Redis or a DB table to
scale horizontally.
                                                            
---
Testing & Validation

Unit tests cover:
- BuildRegistry with enabled/disabled/invalid providers
- LoadProviderConfigs validation (missing issuer, missing credentials, duplicate names)
- EncryptToken/DecryptToken round-trips and tamper detection
- Handler login/callback flows with mock repositories
- Account linking and conflict paths
- Unlink guard (last identity protection, fresh-auth check)

Integration tests (repository/integration/) cover the new IdentityRepository and the restructured SessionRepository against a real PostgreSQL container via testcontainers.

The middleware tests were updated to reflect that context now carries user_id (UUID) rather than *oidc.IDToken.
                                                                                                                                                                               
---                                                                                                                                                                          
Rollout Considerations

This branch includes two migrations:

1. create_identities_and_identity_tokens — additive, safe to apply before code deployment
2. restructure_sessions — begins with TRUNCATE sessions, which invalidates all existing sessions

All existing users will be logged out on deploy. This is intentional. The session schema is incompatible with the old format and there's no sensible migration path for the  
session contents — the old columns stored raw tokens we no longer want stored that way.

Monitoring to add after deploy:
- identity_tokens rows with expired id_token_expiry that aren't rotating — would indicate the refresh flow is broken
- identities count per user — if nobody is linking providers, the feature isn't being used

  ---                                                                                                                                                                          
What's Next

- PKCE store distribution — move pkceStore to Redis or a short-TTL DB table to support horizontal scaling
- Token encryption key rotation — build a re-encryption job that accepts old and new key, rewrites all refresh_token_ciphertext rows, then sets the new key as active
- Provider management UI — the frontend has a LinkProvidersPage wired up but it's basic; showing which identities are linked and surfacing the unlink flow with fresh-auth   
  re-prompting needs polish
- email_trust audit — the flag is correct but needs documentation in the operator guide explaining exactly when to enable it and what verification guarantees are required   
  from the provider
- Session expiry cleanup job — expired sessions accumulate in the table; a background worker or scheduled query should prune them