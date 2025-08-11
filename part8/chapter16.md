# Chapter 16: Building Modern Web Services

## Beyond Hello World

Go's reputation for web services is well-earned. But modern Go web development has evolved far beyond simple HTTP handlers. With enhanced routing, middleware patterns, real-time capabilities, and cloud-native features built into the standard library, Go rivals any web framework while maintaining its simplicity. This chapter shows how to build production-ready web services using modern Go.

## Modern HTTP Server Architecture

### The Service Pattern

```go
package main

import (
    "context"
    "embed"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

//go:embed static/* templates/*
var assets embed.FS

type Server struct {
    router *http.ServeMux
    logger *slog.Logger
    config *Config
    db     *Database
    cache  *Cache
    
    // Services
    userService    *UserService
    authService    *AuthService
    productService *ProductService
}

func NewServer(config *Config) (*Server, error) {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        AddSource: true,
    }))
    
    db, err := NewDatabase(config.DatabaseURL)
    if err != nil {
        return nil, fmt.Errorf("database connection: %w", err)
    }
    
    cache := NewCache(config.RedisURL)
    
    s := &Server{
        router: http.NewServeMux(),
        logger: logger,
        config: config,
        db:     db,
        cache:  cache,
    }
    
    // Initialize services
    s.userService = NewUserService(db, cache)
    s.authService = NewAuthService(db, config.JWTSecret)
    s.productService = NewProductService(db, cache)
    
    // Setup routes
    s.setupRoutes()
    
    return s, nil
}

func (s *Server) setupRoutes() {
    // Static files
    s.router.Handle("GET /static/", http.FileServerFS(assets))
    
    // API routes with method-based routing (Go 1.22+)
    s.router.HandleFunc("GET /api/health", s.handleHealth)
    
    // User routes
    s.router.HandleFunc("POST /api/users", s.handleCreateUser)
    s.router.HandleFunc("GET /api/users/{id}", s.handleGetUser)
    s.router.HandleFunc("PUT /api/users/{id}", s.handleUpdateUser)
    s.router.HandleFunc("DELETE /api/users/{id}", s.handleDeleteUser)
    
    // Auth routes
    s.router.HandleFunc("POST /api/auth/login", s.handleLogin)
    s.router.HandleFunc("POST /api/auth/refresh", s.handleRefresh)
    s.router.HandleFunc("POST /api/auth/logout", s.handleLogout)
    
    // Product routes with wildcard
    s.router.HandleFunc("GET /api/products/{category}/{subcategory...}", s.handleProducts)
    
    // WebSocket endpoint
    s.router.HandleFunc("GET /ws", s.handleWebSocket)
    
    // Catch-all for SPA
    s.router.HandleFunc("GET /{path...}", s.handleSPA)
}

func (s *Server) Run(ctx context.Context) error {
    srv := &http.Server{
        Addr:         s.config.Port,
        Handler:      s.middleware(s.router),
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    // Server run context
    serverCtx, serverStopCtx := context.WithCancel(ctx)
    
    // Listen for syscall signals for graceful shutdown
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    
    go func() {
        <-sig
        
        // Shutdown signal with grace period
        shutdownCtx, cancel := context.WithTimeout(serverCtx, 30*time.Second)
        defer cancel()
        
        go func() {
            <-shutdownCtx.Done()
            if shutdownCtx.Err() == context.DeadlineExceeded {
                s.logger.Error("graceful shutdown timed out, forcing exit")
                os.Exit(1)
            }
        }()
        
        // Trigger graceful shutdown
        if err := srv.Shutdown(shutdownCtx); err != nil {
            s.logger.Error("server shutdown error", "error", err)
        }
        serverStopCtx()
    }()
    
    // Run server
    s.logger.Info("starting server", "addr", srv.Addr)
    err := srv.ListenAndServe()
    if err != nil && err != http.ErrServerClosed {
        return err
    }
    
    // Wait for server context to be done
    <-serverCtx.Done()
    
    return nil
}
```

### Middleware Chain

```go
func (s *Server) middleware(next http.Handler) http.Handler {
    return s.logging(
        s.recovery(
            s.cors(
                s.rateLimit(
                    s.authentication(
                        s.requestID(
                            s.metrics(next)))))))
}

// Request ID middleware
func (s *Server) requestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = generateRequestID()
        }
        
        ctx := context.WithValue(r.Context(), "request_id", id)
        w.Header().Set("X-Request-ID", id)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Logging middleware
func (s *Server) logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        wrapped := &responseWriter{
            ResponseWriter: w,
            status:        200,
        }
        
        next.ServeHTTP(wrapped, r)
        
        s.logger.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", wrapped.status,
            "duration", time.Since(start),
            "ip", r.RemoteAddr,
            "request_id", r.Context().Value("request_id"),
        )
    })
}

// Recovery middleware
func (s *Server) recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                s.logger.Error("panic recovered",
                    "error", err,
                    "stack", string(debug.Stack()),
                )
                
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

// CORS middleware
func (s *Server) cors(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", s.config.CORSOrigin)
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

// Rate limiting middleware
func (s *Server) rateLimit(next http.Handler) http.Handler {
    limiter := rate.NewLimiter(rate.Every(time.Second), 100)
    
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

// Authentication middleware
func (s *Server) authentication(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Skip auth for public endpoints
        if strings.HasPrefix(r.URL.Path, "/api/auth/") ||
           strings.HasPrefix(r.URL.Path, "/static/") {
            next.ServeHTTP(w, r)
            return
        }
        
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        claims, err := s.authService.ValidateToken(token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        ctx := context.WithValue(r.Context(), "user", claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

type responseWriter struct {
    http.ResponseWriter
    status int
    written int64
}

func (w *responseWriter) WriteHeader(status int) {
    w.status = status
    w.ResponseWriter.WriteHeader(status)
}

func (w *responseWriter) Write(b []byte) (int, error) {
    n, err := w.ResponseWriter.Write(b)
    w.written += int64(n)
    return n, err
}
```

## RESTful API Design

### Resource Handlers

```go
// User handlers following REST conventions
func (s *Server) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        s.respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }
    
    // Validate request
    if err := req.Validate(); err != nil {
        s.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    user, err := s.userService.Create(r.Context(), req)
    if err != nil {
        if errors.Is(err, ErrUserExists) {
            s.respondError(w, http.StatusConflict, "User already exists")
            return
        }
        s.logger.Error("create user failed", "error", err)
        s.respondError(w, http.StatusInternalServerError, "Internal error")
        return
    }
    
    s.respondJSON(w, http.StatusCreated, user)
}

func (s *Server) handleGetUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    
    // Try cache first
    if cached, err := s.cache.Get(r.Context(), "user:"+id); err == nil {
        w.Header().Set("X-Cache", "HIT")
        s.respondJSON(w, http.StatusOK, cached)
        return
    }
    
    user, err := s.userService.GetByID(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            s.respondError(w, http.StatusNotFound, "User not found")
            return
        }
        s.logger.Error("get user failed", "error", err)
        s.respondError(w, http.StatusInternalServerError, "Internal error")
        return
    }
    
    // Cache for next time
    s.cache.Set(r.Context(), "user:"+id, user, 5*time.Minute)
    
    w.Header().Set("X-Cache", "MISS")
    s.respondJSON(w, http.StatusOK, user)
}

func (s *Server) handleUpdateUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    
    // Check if user can update this resource
    claims := r.Context().Value("user").(*Claims)
    if claims.UserID != id && !claims.IsAdmin {
        s.respondError(w, http.StatusForbidden, "Forbidden")
        return
    }
    
    var req UpdateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        s.respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }
    
    user, err := s.userService.Update(r.Context(), id, req)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            s.respondError(w, http.StatusNotFound, "User not found")
            return
        }
        s.logger.Error("update user failed", "error", err)
        s.respondError(w, http.StatusInternalServerError, "Internal error")
        return
    }
    
    // Invalidate cache
    s.cache.Delete(r.Context(), "user:"+id)
    
    s.respondJSON(w, http.StatusOK, user)
}

// Response helpers
func (s *Server) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    
    if err := json.NewEncoder(w).Encode(data); err != nil {
        s.logger.Error("encode response failed", "error", err)
    }
}

func (s *Server) respondError(w http.ResponseWriter, status int, message string) {
    s.respondJSON(w, status, map[string]string{
        "error": message,
    })
}
```

## WebSocket Support

### Real-time Communication

```go
import (
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        // Configure origin checking
        return true
    },
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

func NewHub() *Hub {
    return &Hub{
        broadcast:  make(chan []byte),
        register:   make(chan *Client),
        unregister: make(chan *Client),
        clients:    make(map[*Client]bool),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
            
        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()
            
        case message := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
    
    userID string
    roomID string
}

func (s *Server) handleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        s.logger.Error("websocket upgrade failed", "error", err)
        return
    }
    
    claims := r.Context().Value("user").(*Claims)
    
    client := &Client{
        hub:    s.hub,
        conn:   conn,
        send:   make(chan []byte, 256),
        userID: claims.UserID,
    }
    
    client.hub.register <- client
    
    go client.writePump()
    go client.readPump()
}

func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()
    
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })
    
    for {
        var msg Message
        err := c.conn.ReadJSON(&msg)
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("websocket error: %v", err)
            }
            break
        }
        
        // Process message
        c.handleMessage(msg)
    }
}

func (c *Client) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()
    
    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            
            c.conn.WriteMessage(websocket.TextMessage, message)
            
        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

## Server-Sent Events (SSE)

### Real-time Updates Without WebSockets

```go
func (s *Server) handleSSE(w http.ResponseWriter, r *http.Request) {
    // Set headers for SSE
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")
    
    // Create event channel
    events := make(chan Event)
    
    // Register client
    s.eventBus.Subscribe(events)
    defer s.eventBus.Unsubscribe(events)
    
    // Send events
    for {
        select {
        case event := <-events:
            fmt.Fprintf(w, "id: %s\n", event.ID)
            fmt.Fprintf(w, "event: %s\n", event.Type)
            fmt.Fprintf(w, "data: %s\n\n", event.Data)
            
            if f, ok := w.(http.Flusher); ok {
                f.Flush()
            }
            
        case <-r.Context().Done():
            return
        }
    }
}

type EventBus struct {
    subscribers map[chan Event]bool
    mu          sync.RWMutex
}

func (eb *EventBus) Publish(event Event) {
    eb.mu.RLock()
    defer eb.mu.RUnlock()
    
    for ch := range eb.subscribers {
        select {
        case ch <- event:
        default:
            // Don't block if subscriber is slow
        }
    }
}
```

## GraphQL Integration

### GraphQL Server

```go
import (
    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/playground"
)

func (s *Server) setupGraphQL() {
    resolver := &Resolver{
        UserService:    s.userService,
        ProductService: s.productService,
    }
    
    srv := handler.NewDefaultServer(generated.NewExecutableSchema(generated.Config{
        Resolvers: resolver,
    }))
    
    // Add middleware
    srv.Use(extension.Introspection{})
    srv.Use(extension.AutomaticPersistedQuery{
        Cache: lru.New(100),
    })
    
    // GraphQL endpoint
    s.router.Handle("/graphql", srv)
    
    // Playground
    s.router.Handle("/playground", playground.Handler("GraphQL playground", "/graphql"))
}

// Resolver implementation
type Resolver struct {
    UserService    *UserService
    ProductService *ProductService
}

func (r *Resolver) Query() generated.QueryResolver {
    return &queryResolver{r}
}

type queryResolver struct{ *Resolver }

func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    return r.UserService.GetByID(ctx, id)
}

func (r *queryResolver) Products(ctx context.Context, filter *model.ProductFilter) ([]*model.Product, error) {
    return r.ProductService.List(ctx, filter)
}
```

## gRPC Services

### gRPC Server Implementation

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

type GRPCServer struct {
    pb.UnimplementedUserServiceServer
    userService *UserService
}

func (s *GRPCServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.userService.GetByID(ctx, req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user not found: %v", err)
    }
    
    return &pb.User{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}

func (s *Server) RunGRPC(ctx context.Context) error {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        return fmt.Errorf("failed to listen: %w", err)
    }
    
    grpcServer := grpc.NewServer(
        grpc.UnaryInterceptor(grpcMiddleware),
        grpc.StreamInterceptor(grpcStreamMiddleware),
    )
    
    pb.RegisterUserServiceServer(grpcServer, &GRPCServer{
        userService: s.userService,
    })
    
    // Register reflection service
    reflection.Register(grpcServer)
    
    go func() {
        <-ctx.Done()
        grpcServer.GracefulStop()
    }()
    
    return grpcServer.Serve(lis)
}

// gRPC middleware
func grpcMiddleware(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()
    
    // Call handler
    resp, err := handler(ctx, req)
    
    // Log request
    slog.Info("grpc request",
        "method", info.FullMethod,
        "duration", time.Since(start),
        "error", err,
    )
    
    return resp, err
}
```

## API Documentation

### OpenAPI/Swagger Integration

```go
import (
    "github.com/swaggo/http-swagger"
    _ "yourproject/docs" // swagger docs
)

// @title           API Documentation
// @version         1.0
// @description     Modern Go Web Service API
// @host            localhost:8080
// @BasePath        /api

// @securityDefinitions.apikey Bearer
// @in header
// @name Authorization

func (s *Server) setupSwagger() {
    s.router.HandleFunc("GET /swagger/*", httpSwagger.Handler(
        httpSwagger.URL("/swagger/doc.json"),
    ))
}

// handleCreateUser creates a new user
// @Summary      Create user
// @Description  Create a new user account
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        user body CreateUserRequest true "User data"
// @Success      201  {object}  User
// @Failure      400  {object}  ErrorResponse
// @Failure      409  {object}  ErrorResponse
// @Router       /users [post]
// @Security     Bearer
func (s *Server) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    // Implementation
}
```

## Testing Web Services

### Integration Testing

```go
func TestAPI(t *testing.T) {
    // Setup test server
    config := &Config{
        DatabaseURL: "postgres://test",
        Port:        ":0", // Random port
    }
    
    server, err := NewServer(config)
    require.NoError(t, err)
    
    // Create test server
    ts := httptest.NewServer(server.router)
    defer ts.Close()
    
    client := ts.Client()
    
    t.Run("create user", func(t *testing.T) {
        user := CreateUserRequest{
            Name:  "Test User",
            Email: "test@example.com",
        }
        
        body, _ := json.Marshal(user)
        resp, err := client.Post(ts.URL+"/api/users", "application/json", bytes.NewReader(body))
        require.NoError(t, err)
        defer resp.Body.Close()
        
        assert.Equal(t, http.StatusCreated, resp.StatusCode)
        
        var created User
        err = json.NewDecoder(resp.Body).Decode(&created)
        require.NoError(t, err)
        assert.NotEmpty(t, created.ID)
    })
}

// WebSocket testing
func TestWebSocket(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(handleWebSocket))
    defer server.Close()
    
    // Convert http:// to ws://
    u := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"
    
    // Connect
    ws, _, err := websocket.DefaultDialer.Dial(u, nil)
    require.NoError(t, err)
    defer ws.Close()
    
    // Send message
    err = ws.WriteJSON(Message{Type: "ping"})
    require.NoError(t, err)
    
    // Read response
    var resp Message
    err = ws.ReadJSON(&resp)
    require.NoError(t, err)
    assert.Equal(t, "pong", resp.Type)
}
```

## Performance Optimization

### Connection Pooling

```go
func setupHTTPClient() *http.Client {
    return &http.Client{
        Transport: &http.Transport{
            MaxIdleConns:        100,
            MaxIdleConnsPerHost: 10,
            MaxConnsPerHost:     10,
            IdleConnTimeout:     90 * time.Second,
            DisableCompression:  false,
            ForceAttemptHTTP2:   true,
        },
        Timeout: 30 * time.Second,
    }
}
```

### Response Caching

```go
func (s *Server) cacheMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Only cache GET requests
        if r.Method != "GET" {
            next.ServeHTTP(w, r)
            return
        }
        
        key := r.URL.Path
        
        // Check cache
        if cached, err := s.cache.Get(r.Context(), key); err == nil {
            w.Header().Set("X-Cache", "HIT")
            w.Header().Set("Content-Type", "application/json")
            w.Write(cached)
            return
        }
        
        // Capture response
        rec := httptest.NewRecorder()
        next.ServeHTTP(rec, r)
        
        // Copy to real response
        for k, v := range rec.Header() {
            w.Header()[k] = v
        }
        w.Header().Set("X-Cache", "MISS")
        w.WriteHeader(rec.Code)
        
        body := rec.Body.Bytes()
        w.Write(body)
        
        // Cache successful responses
        if rec.Code == http.StatusOK {
            s.cache.Set(r.Context(), key, body, 5*time.Minute)
        }
    })
}
```

## Best Practices

### 1. Structured Logging
```go
// Use structured logging throughout
slog.Info("request processed",
    "method", r.Method,
    "path", r.URL.Path,
    "duration", time.Since(start),
)
```

### 2. Graceful Shutdown
```go
// Always implement graceful shutdown
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
server.Shutdown(ctx)
```

### 3. Health Checks
```go
func (s *Server) handleHealth(w http.ResponseWriter, r *http.Request) {
    health := map[string]string{
        "status": "healthy",
        "db":     s.db.Ping(),
        "cache":  s.cache.Ping(),
    }
    s.respondJSON(w, http.StatusOK, health)
}
```

### 4. Request Validation
```go
func (req *CreateUserRequest) Validate() error {
    if req.Email == "" {
        return errors.New("email required")
    }
    if !isValidEmail(req.Email) {
        return errors.New("invalid email")
    }
    return nil
}
```

## Exercises

1. **Build a REST API**: Create a complete REST API with CRUD operations, authentication, and validation.

2. **WebSocket Chat**: Implement a real-time chat application using WebSockets.

3. **GraphQL Server**: Build a GraphQL server with subscriptions for real-time updates.

4. **Service Mesh**: Implement service discovery and load balancing for microservices.

5. **API Gateway**: Create an API gateway with rate limiting, authentication, and routing.

## Summary

Modern Go web services leverage powerful standard library features and established patterns to build production-ready applications. From the enhanced HTTP router to WebSocket support, from structured logging to graceful shutdown, Go provides everything needed for robust web services without heavyweight frameworks.

Key takeaways:
- The standard library is powerful enough for most web services
- Middleware chains provide clean separation of concerns
- Modern routing with method matching and wildcards
- WebSockets and SSE enable real-time features
- Proper shutdown and health checks are essential

Next, we'll explore deploying these services in cloud-native environments.

---

*Continue to [Chapter 17: Cloud-Native Go](chapter17.md)*