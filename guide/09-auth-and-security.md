# 09 Auth and Security: Passwords, OAuth2, and Not Getting Hacked 🛡️
### building auth that doesn't embarrass you in a breach notification email

---

authentication is one of those areas where the "obvious" approach is almost always wrong. hashing passwords with SHA-256? wrong (we'll explain why). storing JWTs in localStorage? wrong. implementing OAuth2 yourself from scratch? PLEASE no.

there's a lot of accumulated knowledge here that prevents you from reinventing the wheel poorly and shipping auth that a bored security researcher destroys in an afternoon. this chapter covers all of it.

---

## why you can't just use SHA-256 for passwords

from the previous chapter: SHA-256 is a great hash function. so why shouldn't you hash passwords with it?

```go
// THIS IS WRONG. DO NOT DO THIS.
hash := sha256.Sum256([]byte(password))
// store hash[:] in database
```

**the attack:** SHA-256 is designed to be *fast*. it's fast for legitimate uses (data integrity, digital signatures). it's also fast for attackers.

on a single modern GPU, an attacker can compute approximately **10-20 billion SHA-256 hashes per second**. your entire leaked password database could be cracked against a rainbow table (precomputed hash-to-password mapping) in hours.

**the solution:** use password hashing algorithms that are *deliberately slow* and *memory-hard*. these are specifically designed to be:
- slow: you can set a "work factor" that makes each hash computation take 100-500ms
- memory-hard: require significant RAM to compute, making GPU attacks expensive
- resistant to pre-computation: include a salt so rainbow tables are useless

---

## the contenders: bcrypt, scrypt, Argon2, PBKDF2

### bcrypt (1999)

designed by Niels Provos and David Mazières specifically for password hashing. still the most widely used password hasher.

```go
import "golang.org/x/crypto/bcrypt"

// hash a password
func hashPassword(password string) (string, error) {
    // cost factor: 10-12 is typical, 2^cost iterations
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    if err != nil {
        return "", err
    }
    return string(hash), nil
}

// verify a password
func checkPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// the hash looks like:
// $2a$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
// $2a$ = bcrypt version
// 12   = cost factor
// R9h/... = 22 chars of base64-encoded salt
// OPST9/... = remaining chars = the hash
```

bcrypt has a few limitations:
- maximum password length of 72 bytes (silently truncates longer passwords)
- 128-bit internal state limits maximum memory hardness
- not winner of the Password Hashing Competition

**bcrypt at cost 12:** ~100-200ms per hash on modern hardware. slow enough to defeat GPU cracking, fast enough for user logins. OWASP recommends cost 12 as the current minimum.

### Argon2: the winner (2015 Password Hashing Competition)

Argon2 won the 2015 Password Hashing Competition, beating 23 other candidates. it has three variants:
- **Argon2i:** optimized for resistance to side-channel attacks (smart card operations)
- **Argon2d:** optimized against GPU attacks
- **Argon2id:** hybrid, recommended for most applications

**OWASP's 2024+ recommendation: Argon2id.** this is the current gold standard.

```go
import "golang.org/x/crypto/argon2"

type Argon2Params struct {
    Memory      uint32 // in KiB
    Iterations  uint32
    Parallelism uint8
    SaltLength  uint32
    KeyLength   uint32
}

// OWASP recommended minimum parameters
var defaultParams = &Argon2Params{
    Memory:      64 * 1024, // 64 MiB
    Iterations:  3,
    Parallelism: 4,
    SaltLength:  16,
    KeyLength:   32,
}

func hashPasswordArgon2(password string) (string, error) {
    // generate random salt
    salt := make([]byte, defaultParams.SaltLength)
    if _, err := rand.Read(salt); err != nil {
        return "", err
    }
    
    // compute hash
    hash := argon2.IDKey(
        []byte(password),
        salt,
        defaultParams.Iterations,
        defaultParams.Memory,
        defaultParams.Parallelism,
        defaultParams.KeyLength,
    )
    
    // encode as: $argon2id$v=19$m=65536,t=3,p=4$salt_b64$hash_b64
    b64Salt := base64.RawStdEncoding.EncodeToString(salt)
    b64Hash := base64.RawStdEncoding.EncodeToString(hash)
    
    encoded := fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version,
        defaultParams.Memory,
        defaultParams.Iterations,
        defaultParams.Parallelism,
        b64Salt,
        b64Hash,
    )
    
    return encoded, nil
}
```

OWASP's recommended Argon2id parameters as of 2026:
- Memory: **64 MB** (64 × 1024 KiB)
- Iterations: **3**
- Parallelism: **4**
- Salt: **16 bytes minimum** (random, unique per hash)
- Output: **32 bytes**

why memory-hardness matters: a GPU has thousands of cores but limited memory bandwidth. an algorithm requiring 64MB per hash computation can't be parallelized across thousands of GPU cores, there simply isn't enough memory. bcrypt only uses ~4KB, which is why GPUs can run millions of bcrypt instances simultaneously. Argon2id with 64MB per hash can typically run only a few hundred instances per GPU.

### PBKDF2: only for FIPS compliance

PBKDF2 (Password-Based Key Derivation Function 2) is not memory-hard, it's just slow (many iterations of HMAC-SHA256). it doesn't resist GPU attacks as well as Argon2 but is FIPS 140-2 compliant. use it **only** when FIPS compliance is a requirement (some government/healthcare contexts).

### the order of preference (2026)

1. **Argon2id**, new systems, best choice
2. **scrypt**, memory-hard, also good, less library support
3. **bcrypt** at cost 12+, if you need broad library support or already using bcrypt
4. **PBKDF2-SHA256**, only for FIPS compliance
5. **SHA-256**, not for passwords, ever

---

## JWT: what it is and where it goes wrong

**JWT (JSON Web Token)** is a compact, URL-safe token format for transmitting claims between parties. it's commonly used for authentication.

structure: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ik1vZSIsImlhdCI6MTUxNjIzOTAyMn0.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

base64url decode each part:

```json
// header
{ "alg": "HS256", "typ": "JWT" }

// payload
{
  "sub": "1234567890",    // subject (user ID)
  "name": "Moe",
  "iat": 1516239022,      // issued at (Unix timestamp)
  "exp": 1516325422       // expiry
}

// signature: HMAC-SHA256(base64url(header) + "." + base64url(payload), secret)
```

the server verifies the signature to confirm it issued this token and the payload hasn't been tampered with. **the payload is not encrypted, it's just base64 encoded.** anyone can decode it. don't put sensitive data in JWT payloads.

### where JWT goes wrong

**1. storing in localStorage (the classic mistake):**

```javascript
// WRONG: localStorage is accessible to JavaScript
// XSS vulnerability = attacker can steal your tokens
localStorage.setItem("token", jwt);
```

XSS (Cross-Site Scripting) attacks inject malicious JavaScript into your page. if an attacker can run JS, they can read `localStorage` and steal tokens.

**right approach:** `HttpOnly` cookies. they're sent automatically with requests, but JavaScript cannot access them. XSS can't steal them.

```go
// set HttpOnly cookie in Go
http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    jwt,
    HttpOnly: true,   // JavaScript cannot access this
    Secure:   true,   // HTTPS only
    SameSite: http.SameSiteStrictMode, // CSRF protection
    Path:     "/",
    MaxAge:   86400,  // 24 hours
})
```

**2. not validating `alg: none`:**

the JWT spec allows `alg: none` for unsigned tokens. some early libraries accepted this by default, allowing attackers to forge tokens by setting `alg: none` and omitting the signature. always validate the algorithm in the header and reject `none`.

```go
import "github.com/golang-jwt/jwt/v5"

// always specify the expected algorithm explicitly
token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
    // reject unexpected algorithms
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
    }
    return []byte(secret), nil
})
```

**3. long expiry without refresh tokens:**

if your JWT never expires (or expires in 30 days), a stolen token is useful for 30 days. there's no way to invalidate a JWT before its expiry without a token denylist, which defeats the stateless nature of JWT.

**the solution: short-lived access tokens + refresh tokens:**

```
Access Token:  15 minutes expiry, used for API calls
Refresh Token: 7-30 days expiry, used only to get new access tokens
```

when the access token expires, the client uses the refresh token to get a new one. if the user logs out, revoke the refresh token (store it in DB, check on each refresh request). the access token doesn't need revocation, it expires quickly.

**refresh token rotation:** issue a new refresh token with each refresh request and invalidate the old one. if a stolen refresh token is used, the original refresh (by the attacker) invalidates the token the real user has, who then can't refresh and must log in again, triggering a token reuse detection alert.

---

## OAuth2 and OIDC: authentication you don't build yourself

**OAuth2** is an authorization framework. it lets users grant your application limited access to their account on another service, without giving you their password for that service.

"Login with Google" uses OAuth2. you never see the user's Google password. Google redirects back to you with a token saying "this user authenticated, here's their email."

**OIDC (OpenID Connect)** is an identity layer on top of OAuth2. OAuth2 says "this token lets you do X." OIDC adds "this token belongs to user Y." it defines the ID Token (a JWT) and the `/userinfo` endpoint.

- **OAuth2:** authorization ("what can this app do?")
- **OIDC:** authentication ("who is this user?")

### the authorization code flow + PKCE (what you actually implement)

as of 2025, the IETF Best Current Practice for OAuth2 Security mandates **PKCE (Proof Key for Code Exchange)** for all clients, including server-side.

```
1. User clicks "Login with Google"

2. Your server generates:
   code_verifier = random(32 bytes, base64url)
   code_challenge = base64url(SHA256(code_verifier))

3. Redirect to Google:
   https://accounts.google.com/o/oauth2/auth
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &scope=openid email profile
     &state=random_csrf_token
     &code_challenge=base64url(sha256(verifier))
     &code_challenge_method=S256

4. User authenticates with Google, grants permission

5. Google redirects to your callback:
   https://yourapp.com/callback?code=AUTH_CODE&state=same_csrf_token

6. Verify state matches (CSRF protection)

7. Your server exchanges code for tokens:
   POST https://oauth2.googleapis.com/token
   {
     grant_type: "authorization_code",
     code: AUTH_CODE,
     redirect_uri: "https://yourapp.com/callback",
     client_id: YOUR_CLIENT_ID,
     client_secret: YOUR_SECRET,
     code_verifier: original_verifier  // PKCE: proves it's you
   }

8. Google returns:
   {
     "access_token": "...",
     "id_token": "eyJ...",   // JWT with user identity
     "refresh_token": "...",
     "expires_in": 3600
   }

9. Validate the id_token:
   - verify signature using Google's JWKS (public keys from jwks_uri)
   - verify issuer (iss) matches
   - verify audience (aud) matches your client_id
   - verify token hasn't expired (exp)
   - verify nonce if you sent one

10. Extract user identity from id_token claims:
    { "sub": "google-user-id-123", "email": "user@gmail.com", "name": "..." }
```

PKCE prevents interception attacks: even if someone intercepts the auth code, they can't exchange it without the `code_verifier` that only your server knows.

```go
// Go OAuth2 implementation with golang.org/x/oauth2
import "golang.org/x/oauth2"
import "golang.org/x/oauth2/google"

var googleOAuth = &oauth2.Config{
    ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
    ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
    RedirectURL:  "https://yourapp.com/auth/callback",
    Scopes:       []string{"openid", "email", "profile"},
    Endpoint:     google.Endpoint,
}

func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
    // generate PKCE verifier
    verifier := oauth2.GenerateVerifier()
    
    // store verifier in session (need it for the callback)
    session := getSession(r)
    session.Values["pkce_verifier"] = verifier
    session.Values["oauth_state"] = generateState()
    session.Save(r, w)
    
    // redirect to Google
    url := googleOAuth.AuthCodeURL(
        session.Values["oauth_state"].(string),
        oauth2.S256ChallengeOption(verifier),
    )
    http.Redirect(w, r, url, http.StatusFound)
}

func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
    session := getSession(r)
    
    // verify state (CSRF protection)
    if r.URL.Query().Get("state") != session.Values["oauth_state"] {
        http.Error(w, "invalid state", http.StatusBadRequest)
        return
    }
    
    // exchange code for tokens with PKCE
    verifier := session.Values["pkce_verifier"].(string)
    token, err := googleOAuth.Exchange(r.Context(),
        r.URL.Query().Get("code"),
        oauth2.VerifierOption(verifier),
    )
    
    // extract ID token and validate
    // use a proper OIDC library like coreos/go-oidc
}
```

### the Backend For Frontend (BFF) pattern

the IETF now recommends the BFF pattern for web applications. instead of the frontend handling OAuth2 directly, a dedicated server component (the BFF) handles all authentication:

```
Frontend (browser) ←→ BFF (your server) ←→ Identity Provider
                         ↑
                    handles OAuth2,
                    stores tokens server-side,
                    issues HttpOnly session cookies to browser
```

the browser never touches access tokens or refresh tokens directly. the BFF validates requests using the session cookie and makes API calls using the stored tokens. XSS attacks can't steal what the JavaScript can't access.

---

## common auth vulnerabilities you must know

### CSRF (Cross-Site Request Forgery)

a malicious site tricks a user's browser into making a request to your site with the user's cookies.

```html
<!-- attacker's site -->
<img src="https://bank.example.com/transfer?to=attacker&amount=10000">
```

the bank's server receives a GET request with the user's session cookie and processes the transfer.

**mitigation:**
- `SameSite=Strict` cookies (browser won't send them on cross-site requests)
- CSRF tokens (hidden form field or header, validated server-side)
- check `Origin` and `Referer` headers

```go
// CSRF token generation
token := generateCSRFToken()
w.Header().Set("X-CSRF-Token", token)
// client sends this header back with state-changing requests
// server validates it matches the session's token
```

### SQL injection

classic. still happens in 2026 because people still build queries by string concatenation.

```go
// CATASTROPHICALLY WRONG - allows SQL injection
query := "SELECT * FROM users WHERE email = '" + email + "'"

// RIGHT - parameterized queries
row := db.QueryRowContext(ctx, "SELECT * FROM users WHERE email = $1", email)
```

the user can set `email = ' OR '1'='1` and the first query becomes `SELECT * FROM users WHERE email = '' OR '1'='1'`, returning all users. the parameterized version treats the input as data, not as SQL code.

use parameterized queries. always. no exceptions.

### timing attacks in authentication

we mentioned this in the crypto chapter. comparing passwords/tokens in a way that leaks timing information:

```go
// VULNERABLE: returns false immediately on first character mismatch
if storedHash != computedHash {
    return false
}

// CORRECT: constant-time comparison
if !hmac.Equal([]byte(storedHash), []byte(computedHash)) {
    return false
}
```

an attacker making thousands of requests can measure response time differences of microseconds and determine how many characters of their guess match the stored value. constant-time comparison eliminates this.

### rate limiting authentication endpoints

```go
// without rate limiting, an attacker can try millions of passwords
// simple in-memory rate limiter per IP (use Redis for distributed systems)

type RateLimiter struct {
    attempts map[string]*attemptRecord
    mu       sync.Mutex
}

type attemptRecord struct {
    count    int
    lastSeen time.Time
}

func (rl *RateLimiter) CheckLogin(ip string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    record, exists := rl.attempts[ip]
    if !exists || time.Since(record.lastSeen) > 15*time.Minute {
        rl.attempts[ip] = &attemptRecord{count: 1, lastSeen: time.Now()}
        return true
    }
    
    if record.count >= 10 {  // max 10 attempts per 15 minutes
        return false
    }
    
    record.count++
    record.lastSeen = time.Now()
    return true
}
```

also: exponential backoff, CAPTCHA after N failures, account lockout with unlock-via-email.

---

## session management: the complete picture

```go
// session token generation
func generateSessionToken() (string, error) {
    bytes := make([]byte, 32)  // 256 bits
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}

// store in Redis (not in JWT payload, because sessions can be revoked)
type Session struct {
    UserID    int64
    CreatedAt time.Time
    ExpiresAt time.Time
    IPAddress string
    UserAgent string
}

func (s *SessionStore) Create(ctx context.Context, session *Session) (string, error) {
    token, err := generateSessionToken()
    if err != nil {
        return "", err
    }
    
    key := "session:" + token
    data, _ := json.Marshal(session)
    ttl := time.Until(session.ExpiresAt)
    
    return token, s.redis.Set(ctx, key, data, ttl).Err()
}

func (s *SessionStore) Get(ctx context.Context, token string) (*Session, error) {
    key := "session:" + token
    data, err := s.redis.Get(ctx, key).Bytes()
    if err != nil {
        return nil, ErrSessionNotFound
    }
    
    var session Session
    return &session, json.Unmarshal(data, &session)
}

func (s *SessionStore) Delete(ctx context.Context, token string) error {
    return s.redis.Del(ctx, "session:" + token).Err()
}
```

server-side sessions can be revoked instantly (just delete from Redis). JWTs cannot be revoked before expiry without a denylist (which means you're doing server-side state anyway). for long-lived sessions (stay logged in for 30 days), server-side sessions with Redis are usually the right choice.

---

## security headers: the free wins

these HTTP response headers provide significant security improvements for virtually zero implementation cost:

```go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // prevent clickjacking
        w.Header().Set("X-Frame-Options", "DENY")
        
        // prevent MIME type sniffing
        w.Header().Set("X-Content-Type-Options", "nosniff")
        
        // force HTTPS for 2 years, include subdomains
        w.Header().Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")
        
        // content security policy (customize per your needs)
        w.Header().Set("Content-Security-Policy",
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'")
        
        // disable referrer info for privacy
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        // restrict browser APIs for security
        w.Header().Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        next.ServeHTTP(w, r)
    })
}
```

check your headers against [securityheaders.com](https://securityheaders.com), it grades your headers and explains what each one does.

---

*next: [10 Payments: Stripe, Crypto, and Why Bitcoin Isn't Ready](10-payments.md)*

*sources: [OWASP password cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) | [Argon2 vs bcrypt 2026](https://guptadeepak.com/the-complete-guide-to-password-hashing-argon2-vs-bcrypt-vs-scrypt-vs-pbkdf2-2026/) | [OAuth2 IETF best practices 2025](https://dev.to/kimmaida/oauth-20-security-best-practices-for-developers-2ba5) | [OIDC deep dive](https://medium.com/@QuarkAndCode/oauth-2-0-vs-openid-connect-oidc-best-practices-security-in-2025-0c82f071a9a9)*
