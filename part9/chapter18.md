# Chapter 18: Security in Modern Go

## Security by Design

Go was designed with security in mind. Memory safety without garbage collection overhead, built-in cryptography packages, and a compiler that catches many vulnerabilities make Go inherently more secure than many alternatives. But modern threats require modern defenses. This chapter explores security best practices, tools, and patterns for building secure Go applications.

## Input Validation and Sanitization

### SQL Injection Prevention

```go
package main

import (
    "database/sql"
    "fmt"
    "regexp"
    "strings"
    
    _ "github.com/lib/pq"
)

// NEVER do this - vulnerable to SQL injection
func badQuery(db *sql.DB, userInput string) error {
    query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userInput)
    _, err := db.Query(query)  // SQL injection vulnerability!
    return err
}

// ALWAYS use parameterized queries
func safeQuery(db *sql.DB, userInput string) (*sql.Rows, error) {
    query := "SELECT * FROM users WHERE name = $1"
    return db.Query(query, userInput)  // Safe from SQL injection
}

// Safe dynamic query building
type QueryBuilder struct {
    query  strings.Builder
    args   []interface{}
    argIdx int
}

func (qb *QueryBuilder) Where(column string, value interface{}) *QueryBuilder {
    if qb.argIdx > 0 {
        qb.query.WriteString(" AND ")
    } else {
        qb.query.WriteString(" WHERE ")
    }
    
    qb.argIdx++
    qb.query.WriteString(fmt.Sprintf("%s = $%d", column, qb.argIdx))
    qb.args = append(qb.args, value)
    
    return qb
}

func (qb *QueryBuilder) Build() (string, []interface{}) {
    return qb.query.String(), qb.args
}

// Usage
func searchUsers(db *sql.DB, filters map[string]interface{}) (*sql.Rows, error) {
    qb := &QueryBuilder{}
    qb.query.WriteString("SELECT * FROM users")
    
    for column, value := range filters {
        // Validate column name against whitelist
        if !isValidColumn(column) {
            return nil, fmt.Errorf("invalid column: %s", column)
        }
        qb.Where(column, value)
    }
    
    query, args := qb.Build()
    return db.Query(query, args...)
}

func isValidColumn(column string) bool {
    validColumns := map[string]bool{
        "id": true, "name": true, "email": true, "created_at": true,
    }
    return validColumns[column]
}
```

### Command Injection Prevention

```go
import (
    "os/exec"
    "path/filepath"
    "regexp"
)

// NEVER do this - command injection vulnerability
func badCommand(userInput string) error {
    cmd := exec.Command("sh", "-c", "echo "+userInput)  // Vulnerable!
    return cmd.Run()
}

// Safe command execution
func safeCommand(userInput string) error {
    // Use exec.Command with separate arguments
    cmd := exec.Command("echo", userInput)  // Arguments are safely escaped
    return cmd.Run()
}

// Safe file operations
func safeFileOperation(userPath string) error {
    // Clean and validate path
    cleanPath := filepath.Clean(userPath)
    
    // Ensure path doesn't escape base directory
    basePath := "/var/app/data"
    fullPath := filepath.Join(basePath, cleanPath)
    
    // Verify the path is still within base directory
    if !strings.HasPrefix(fullPath, basePath) {
        return fmt.Errorf("invalid path: attempted directory traversal")
    }
    
    // Additional validation
    if strings.Contains(fullPath, "..") {
        return fmt.Errorf("invalid path: contains ..")
    }
    
    return processFile(fullPath)
}

// Input validation with regex
var (
    emailRegex    = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    usernameRegex = regexp.MustCompile(`^[a-zA-Z0-9_-]{3,32}$`)
    uuidRegex     = regexp.MustCompile(`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`)
)

type UserInput struct {
    Email    string `json:"email"`
    Username string `json:"username"`
    UserID   string `json:"user_id"`
}

func (u *UserInput) Validate() error {
    if !emailRegex.MatchString(u.Email) {
        return fmt.Errorf("invalid email format")
    }
    
    if !usernameRegex.MatchString(u.Username) {
        return fmt.Errorf("invalid username: must be 3-32 alphanumeric characters")
    }
    
    if !uuidRegex.MatchString(u.UserID) {
        return fmt.Errorf("invalid user ID: must be valid UUID")
    }
    
    return nil
}
```

### XSS Prevention

```go
import (
    "html/template"
    "github.com/microcosm-cc/bluemonday"
)

// HTML template automatically escapes
func safeTemplate(userInput string) string {
    tmpl := template.Must(template.New("test").Parse(`
        <div>User said: {{.}}</div>
    `))
    
    var buf bytes.Buffer
    tmpl.Execute(&buf, userInput)  // Automatically escaped
    return buf.String()
}

// For user-generated HTML content
func sanitizeHTML(userHTML string) string {
    // Create a strict policy
    p := bluemonday.StrictPolicy()
    
    // Or create a custom policy
    p = bluemonday.NewPolicy()
    p.AllowElements("p", "br", "strong", "em")
    p.AllowAttrs("href").OnElements("a")
    p.RequireParseableURLs(true)
    p.AllowRelativeURLs(false)
    p.RequireNoFollowOnLinks(true)
    
    // Sanitize the HTML
    return p.Sanitize(userHTML)
}

// JSON output safety
func jsonResponse(w http.ResponseWriter, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Content-Type-Options", "nosniff")  // Prevent MIME sniffing
    
    if err := json.NewEncoder(w).Encode(data); err != nil {
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
    }
}
```

## Authentication and Authorization

### JWT Implementation

```go
import (
    "time"
    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID   string   `json:"user_id"`
    Email    string   `json:"email"`
    Roles    []string `json:"roles"`
    jwt.RegisteredClaims
}

type TokenService struct {
    secretKey []byte
    issuer    string
}

func NewTokenService(secretKey string, issuer string) *TokenService {
    return &TokenService{
        secretKey: []byte(secretKey),
        issuer:    issuer,
    }
}

func (ts *TokenService) GenerateToken(userID, email string, roles []string) (string, error) {
    claims := &Claims{
        UserID: userID,
        Email:  email,
        Roles:  roles,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    ts.issuer,
            Subject:   userID,
            ID:        generateTokenID(),
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(ts.secretKey)
}

func (ts *TokenService) ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Verify signing method
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return ts.secretKey, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, fmt.Errorf("invalid token")
}

// Refresh token with rotation
type RefreshToken struct {
    Token     string
    UserID    string
    ExpiresAt time.Time
    Used      bool
}

func (ts *TokenService) GenerateRefreshToken(userID string) (*RefreshToken, error) {
    token := &RefreshToken{
        Token:     generateSecureToken(32),
        UserID:    userID,
        ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
        Used:      false,
    }
    
    // Store in database
    if err := storeRefreshToken(token); err != nil {
        return nil, err
    }
    
    return token, nil
}

func (ts *TokenService) RefreshAccessToken(refreshToken string) (string, *RefreshToken, error) {
    // Retrieve and validate refresh token
    token, err := getRefreshToken(refreshToken)
    if err != nil {
        return "", nil, err
    }
    
    if token.Used {
        // Token reuse detected - potential attack
        revokeAllUserTokens(token.UserID)
        return "", nil, fmt.Errorf("refresh token reuse detected")
    }
    
    if time.Now().After(token.ExpiresAt) {
        return "", nil, fmt.Errorf("refresh token expired")
    }
    
    // Mark old token as used
    token.Used = true
    updateRefreshToken(token)
    
    // Generate new tokens
    user, _ := getUserByID(token.UserID)
    accessToken, _ := ts.GenerateToken(user.ID, user.Email, user.Roles)
    newRefreshToken, _ := ts.GenerateRefreshToken(user.ID)
    
    return accessToken, newRefreshToken, nil
}
```

### OAuth 2.0 Implementation

```go
import (
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

func setupOAuth() *oauth2.Config {
    return &oauth2.Config{
        ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
        ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
        RedirectURL:  "https://example.com/auth/callback",
        Scopes:       []string{"email", "profile"},
        Endpoint:     google.Endpoint,
    }
}

func handleOAuthLogin(w http.ResponseWriter, r *http.Request) {
    config := setupOAuth()
    
    // Generate state token for CSRF protection
    state := generateSecureToken(32)
    
    // Store state in session
    session, _ := store.Get(r, "auth-session")
    session.Values["state"] = state
    session.Save(r, w)
    
    // Redirect to provider
    url := config.AuthCodeURL(state, oauth2.AccessTypeOffline)
    http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

func handleOAuthCallback(w http.ResponseWriter, r *http.Request) {
    config := setupOAuth()
    
    // Verify state token
    session, _ := store.Get(r, "auth-session")
    expectedState := session.Values["state"].(string)
    
    if r.FormValue("state") != expectedState {
        http.Error(w, "Invalid state token", http.StatusBadRequest)
        return
    }
    
    // Exchange code for token
    token, err := config.Exchange(r.Context(), r.FormValue("code"))
    if err != nil {
        http.Error(w, "Failed to exchange token", http.StatusInternalServerError)
        return
    }
    
    // Get user info
    client := config.Client(r.Context(), token)
    resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
    if err != nil {
        http.Error(w, "Failed to get user info", http.StatusInternalServerError)
        return
    }
    defer resp.Body.Close()
    
    var userInfo struct {
        Email         string `json:"email"`
        VerifiedEmail bool   `json:"verified_email"`
        Name          string `json:"name"`
        Picture       string `json:"picture"`
    }
    
    json.NewDecoder(resp.Body).Decode(&userInfo)
    
    // Create or update user
    user := findOrCreateUser(userInfo.Email, userInfo.Name)
    
    // Generate session
    generateUserSession(w, r, user)
}
```

## Cryptography Best Practices

### Password Hashing

```go
import (
    "golang.org/x/crypto/bcrypt"
    "golang.org/x/crypto/argon2"
    "crypto/rand"
    "encoding/base64"
)

// Bcrypt for passwords
func hashPassword(password string) (string, error) {
    // Use cost of 12 or higher
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    if err != nil {
        return "", err
    }
    return string(hash), nil
}

func verifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// Argon2 for higher security requirements
type Argon2Params struct {
    Memory      uint32
    Iterations  uint32
    Parallelism uint8
    SaltLength  uint32
    KeyLength   uint32
}

var defaultArgon2Params = &Argon2Params{
    Memory:      64 * 1024,  // 64 MB
    Iterations:  3,
    Parallelism: 2,
    SaltLength:  16,
    KeyLength:   32,
}

func hashPasswordArgon2(password string) (string, error) {
    params := defaultArgon2Params
    
    // Generate salt
    salt := make([]byte, params.SaltLength)
    if _, err := rand.Read(salt); err != nil {
        return "", err
    }
    
    // Hash password
    hash := argon2.IDKey([]byte(password), salt, params.Iterations, 
        params.Memory, params.Parallelism, params.KeyLength)
    
    // Encode parameters, salt, and hash
    b64Salt := base64.RawStdEncoding.EncodeToString(salt)
    b64Hash := base64.RawStdEncoding.EncodeToString(hash)
    
    return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version, params.Memory, params.Iterations, params.Parallelism,
        b64Salt, b64Hash), nil
}
```

### Encryption

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "crypto/x509"
    "encoding/pem"
)

// AES-GCM encryption
func encrypt(plaintext []byte, key []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    
    nonce := make([]byte, gcm.NonceSize())
    if _, err := rand.Read(nonce); err != nil {
        return nil, err
    }
    
    ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
    return ciphertext, nil
}

func decrypt(ciphertext []byte, key []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    
    nonceSize := gcm.NonceSize()
    if len(ciphertext) < nonceSize {
        return nil, fmt.Errorf("ciphertext too short")
    }
    
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return nil, err
    }
    
    return plaintext, nil
}

// RSA encryption for key exchange
func generateRSAKeyPair() (*rsa.PrivateKey, *rsa.PublicKey, error) {
    privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        return nil, nil, err
    }
    
    return privateKey, &privateKey.PublicKey, nil
}

func encryptRSA(publicKey *rsa.PublicKey, plaintext []byte) ([]byte, error) {
    hash := sha256.New()
    ciphertext, err := rsa.EncryptOAEP(hash, rand.Reader, publicKey, plaintext, nil)
    if err != nil {
        return nil, err
    }
    
    return ciphertext, nil
}

func decryptRSA(privateKey *rsa.PrivateKey, ciphertext []byte) ([]byte, error) {
    hash := sha256.New()
    plaintext, err := rsa.DecryptOAEP(hash, rand.Reader, privateKey, ciphertext, nil)
    if err != nil {
        return nil, err
    }
    
    return plaintext, nil
}
```

## Secure Communication

### TLS Configuration

```go
import (
    "crypto/tls"
    "net/http"
)

func setupSecureServer() *http.Server {
    tlsConfig := &tls.Config{
        MinVersion:               tls.VersionTLS13,
        PreferServerCipherSuites: true,
        CipherSuites: []uint16{
            tls.TLS_AES_256_GCM_SHA384,
            tls.TLS_CHACHA20_POLY1305_SHA256,
            tls.TLS_AES_128_GCM_SHA256,
        },
        CurvePreferences: []tls.CurveID{
            tls.X25519,
            tls.CurveP256,
        },
    }
    
    server := &http.Server{
        Addr:      ":443",
        Handler:   secureRouter(),
        TLSConfig: tlsConfig,
        
        // Timeouts
        ReadTimeout:       15 * time.Second,
        ReadHeaderTimeout: 5 * time.Second,
        WriteTimeout:      15 * time.Second,
        IdleTimeout:       60 * time.Second,
    }
    
    return server
}

// mTLS (Mutual TLS)
func setupMutualTLS() *tls.Config {
    // Load CA cert
    caCert, err := os.ReadFile("ca-cert.pem")
    if err != nil {
        log.Fatal(err)
    }
    
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)
    
    // Load server cert and key
    serverCert, err := tls.LoadX509KeyPair("server-cert.pem", "server-key.pem")
    if err != nil {
        log.Fatal(err)
    }
    
    return &tls.Config{
        Certificates: []tls.Certificate{serverCert},
        ClientAuth:   tls.RequireAndVerifyClientCert,
        ClientCAs:    caCertPool,
        MinVersion:   tls.VersionTLS13,
    }
}

// Certificate pinning
func setupCertificatePinning(expectedHash string) *http.Client {
    return &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
                VerifyPeerCertificate: func(certificates [][]byte, _ [][]*x509.Certificate) error {
                    for _, cert := range certificates {
                        hash := sha256.Sum256(cert)
                        actualHash := base64.StdEncoding.EncodeToString(hash[:])
                        
                        if actualHash == expectedHash {
                            return nil
                        }
                    }
                    
                    return fmt.Errorf("certificate pinning failed")
                },
            },
        },
    }
}
```

### API Security

```go
// API key authentication
func apiKeyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        apiKey := r.Header.Get("X-API-Key")
        
        if apiKey == "" {
            http.Error(w, "Missing API key", http.StatusUnauthorized)
            return
        }
        
        // Validate API key (constant-time comparison)
        hashedKey := hashAPIKey(apiKey)
        validKey, err := getAPIKeyFromDB(hashedKey)
        if err != nil || !constantTimeCompare(hashedKey, validKey.Hash) {
            http.Error(w, "Invalid API key", http.StatusUnauthorized)
            return
        }
        
        // Check rate limits for this key
        if !checkRateLimit(apiKey) {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        // Add key info to context
        ctx := context.WithValue(r.Context(), "api_key", validKey)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// HMAC request signing
func verifyRequestSignature(r *http.Request, secret []byte) bool {
    signature := r.Header.Get("X-Signature")
    timestamp := r.Header.Get("X-Timestamp")
    
    // Check timestamp to prevent replay attacks
    ts, _ := strconv.ParseInt(timestamp, 10, 64)
    if time.Now().Unix()-ts > 300 { // 5 minutes
        return false
    }
    
    // Read body
    body, _ := io.ReadAll(r.Body)
    r.Body = io.NopCloser(bytes.NewReader(body))
    
    // Compute expected signature
    message := fmt.Sprintf("%s%s%s%s", r.Method, r.URL.Path, timestamp, string(body))
    h := hmac.New(sha256.New, secret)
    h.Write([]byte(message))
    expectedSignature := base64.StdEncoding.EncodeToString(h.Sum(nil))
    
    // Constant-time comparison
    return hmac.Equal([]byte(signature), []byte(expectedSignature))
}
```

## Supply Chain Security

### Dependency Verification

```go
// go.mod with checksums
module github.com/example/secure-app

go 1.22

require (
    github.com/lib/pq v1.10.9
    golang.org/x/crypto v0.17.0
)

// Verify dependencies
// $ go mod verify
// $ go list -m all | nancy sleuth  // Check for vulnerabilities
```

### SBOM Generation

```bash
# Generate Software Bill of Materials
$ go mod vendor
$ cyclonedx-gomod mod -json -output sbom.json

# Or with syft
$ syft packages . -o cyclonedx-json > sbom.json
```

## Runtime Security

### Seccomp and AppArmor

```go
import (
    "github.com/opencontainers/runtime-spec/specs-go"
    "github.com/containers/common/pkg/seccomp"
)

func setupSeccomp() *specs.LinuxSeccomp {
    return &specs.LinuxSeccomp{
        DefaultAction: specs.ActErrno,
        Architectures: []specs.Arch{specs.ArchX86_64},
        Syscalls: []specs.LinuxSyscall{
            {
                Names:  []string{"read", "write", "open", "close"},
                Action: specs.ActAllow,
            },
            // Add other required syscalls
        },
    }
}
```

### Security Headers

```go
func securityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Security headers
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        w.Header().Set("Content-Security-Policy", "default-src 'self'; script-src 'self' 'unsafe-inline'")
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        w.Header().Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        next.ServeHTTP(w, r)
    })
}
```

## Secure Coding Patterns

### Secrets Management

```go
type SecretStore interface {
    GetSecret(ctx context.Context, key string) ([]byte, error)
    SetSecret(ctx context.Context, key string, value []byte) error
    DeleteSecret(ctx context.Context, key string) error
}

// Zero out secrets in memory
func clearSecret(secret []byte) {
    for i := range secret {
        secret[i] = 0
    }
}

// Use defer to ensure cleanup
func useSecret() {
    secret := getSecretFromStore()
    defer clearSecret(secret)
    
    // Use secret
}
```

### Error Handling

```go
// Don't leak sensitive information in errors
func authenticate(username, password string) error {
    user, err := getUserByUsername(username)
    if err != nil {
        // Don't reveal if user exists
        return fmt.Errorf("authentication failed")
    }
    
    if !verifyPassword(password, user.PasswordHash) {
        // Same error as above
        return fmt.Errorf("authentication failed")
    }
    
    return nil
}

// Log security events
func logSecurityEvent(event string, details map[string]interface{}) {
    slog.Warn("security event",
        "event", event,
        "details", details,
        "timestamp", time.Now(),
        "source_ip", getClientIP(),
    )
}
```

## Best Practices

### 1. Principle of Least Privilege
```go
// Run with minimal permissions
// Drop privileges after startup
// Use separate service accounts
```

### 2. Defense in Depth
```go
// Multiple layers of security
// Input validation → Authentication → Authorization → Audit logging
```

### 3. Fail Securely
```go
// Default to deny
if !isAuthorized {
    return errors.New("access denied")
}
```

### 4. Regular Updates
```bash
# Keep dependencies updated
$ go get -u ./...
$ go mod tidy
$ govulncheck ./...
```

## Exercises

1. **Implement OAuth 2.0**: Build a complete OAuth 2.0 provider with PKCE support.

2. **Create a WAF**: Build a Web Application Firewall middleware for Go applications.

3. **Secure API**: Design and implement a secure REST API with authentication, rate limiting, and audit logging.

4. **Encryption Service**: Build a service that handles encryption/decryption with key rotation.

5. **Security Scanner**: Create a tool that scans Go code for common security vulnerabilities.

## Summary

Security in modern Go requires a comprehensive approach: from input validation to cryptography, from secure communication to supply chain security. Go provides excellent tools and libraries, but security ultimately depends on proper implementation and continuous vigilance. The key is defense in depth—multiple layers of security working together.

Key takeaways:
- Always validate and sanitize input
- Use parameterized queries for database access
- Implement proper authentication and authorization
- Use strong, modern cryptography
- Keep dependencies updated and verified
- Monitor and log security events

Next, we'll explore experimental features that might shape the future of Go.

---

*Continue to [Chapter 19: Experimental Features](chapter19.md)*