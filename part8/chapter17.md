# Chapter 17: Cloud-Native Go

## Born for the Cloud

Go wasn't just adopted by the cloud-native movement—it helped define it. Docker, Kubernetes, Prometheus, Terraform, and countless other cloud tools are written in Go. This isn't coincidence. Go's fast compilation, single-binary deployment, excellent concurrency, and low resource usage make it ideal for cloud environments. This chapter explores building cloud-native applications with modern Go.

## Container Optimization

### Multi-Stage Docker Builds

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder

# Install certificates for HTTPS
RUN apk --no-cache add ca-certificates

WORKDIR /build

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build with optimizations
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath \
    -ldflags="-s -w -extldflags '-static' \
    -X main.version=${VERSION:-dev} \
    -X main.commit=${COMMIT:-unknown} \
    -X main.buildTime=$(date -u +%Y%m%d-%H%M%S)" \
    -o app ./cmd/api

# Runtime stage - distroless for security
FROM gcr.io/distroless/static:nonroot

# Copy certificates
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /build/app /app

# Run as non-root
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/app"]
```

### Minimal Alpine Image

```dockerfile
# For when you need a shell
FROM golang:1.22-alpine AS builder

RUN apk add --no-cache git

WORKDIR /src
COPY . .

RUN go mod download
RUN CGO_ENABLED=0 go build -o /app ./cmd/api

# Final stage
FROM alpine:3.19

RUN apk --no-cache add ca-certificates tzdata

# Create non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser

COPY --from=builder /app /app

USER appuser

EXPOSE 8080

CMD ["/app"]
```

### Container-Aware Go Application

```go
package main

import (
    "context"
    "fmt"
    "os"
    "runtime"
    "runtime/debug"
)

func configureForContainer() {
    // Detect if running in container
    if _, err := os.Stat("/.dockerenv"); err == nil {
        fmt.Println("Running in Docker container")
        optimizeForContainer()
    }
    
    // Check for Kubernetes
    if os.Getenv("KUBERNETES_SERVICE_HOST") != "" {
        fmt.Println("Running in Kubernetes")
        configureForKubernetes()
    }
}

func optimizeForContainer() {
    // Set GOMAXPROCS to container CPU limit
    if cpuQuota := getCGroupCPUQuota(); cpuQuota > 0 {
        runtime.GOMAXPROCS(cpuQuota)
    }
    
    // Set memory limit from cgroup
    if memLimit := getCGroupMemoryLimit(); memLimit > 0 {
        debug.SetMemoryLimit(memLimit)
    }
    
    // Tune GC for container
    debug.SetGCPercent(50) // More aggressive GC in containers
}

func getCGroupCPUQuota() int {
    // Read from /sys/fs/cgroup/cpu/cpu.cfs_quota_us and cpu.cfs_period_us
    quotaFile := "/sys/fs/cgroup/cpu/cpu.cfs_quota_us"
    periodFile := "/sys/fs/cgroup/cpu/cpu.cfs_period_us"
    
    quota, err := readIntFromFile(quotaFile)
    if err != nil || quota <= 0 {
        return 0
    }
    
    period, err := readIntFromFile(periodFile)
    if err != nil || period <= 0 {
        return 0
    }
    
    return int(quota / period)
}

func getCGroupMemoryLimit() int64 {
    // Read from /sys/fs/cgroup/memory/memory.limit_in_bytes
    limitFile := "/sys/fs/cgroup/memory/memory.limit_in_bytes"
    
    limit, err := readIntFromFile(limitFile)
    if err != nil {
        return 0
    }
    
    // Check if it's the default unlimited value
    if limit > 1<<60 {
        return 0
    }
    
    return limit
}
```

## Kubernetes Integration

### Kubernetes Operator

```go
package main

import (
    "context"
    "fmt"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
)

// Custom Resource Definition
type MyApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   MyAppSpec   `json:"spec,omitempty"`
    Status MyAppStatus `json:"status,omitempty"`
}

type MyAppSpec struct {
    Replicas int32  `json:"replicas"`
    Image    string `json:"image"`
    Version  string `json:"version"`
}

type MyAppStatus struct {
    AvailableReplicas int32  `json:"availableReplicas"`
    LastUpdate        string `json:"lastUpdate"`
}

// Reconciler
type MyAppReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    // Fetch the MyApp instance
    var myApp MyApp
    if err := r.Get(ctx, req.NamespacedName, &myApp); err != nil {
        log.Error(err, "unable to fetch MyApp")
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // Create or update Deployment
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      myApp.Name,
            Namespace: myApp.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &myApp.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app": myApp.Name,
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app":     myApp.Name,
                        "version": myApp.Spec.Version,
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "app",
                            Image: myApp.Spec.Image + ":" + myApp.Spec.Version,
                            Ports: []corev1.ContainerPort{
                                {
                                    ContainerPort: 8080,
                                },
                            },
                            Env: []corev1.EnvVar{
                                {
                                    Name:  "VERSION",
                                    Value: myApp.Spec.Version,
                                },
                            },
                            LivenessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    HTTPGet: &corev1.HTTPGetAction{
                                        Path: "/health",
                                        Port: intstr.FromInt(8080),
                                    },
                                },
                                InitialDelaySeconds: 30,
                                PeriodSeconds:       10,
                            },
                            ReadinessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    HTTPGet: &corev1.HTTPGetAction{
                                        Path: "/ready",
                                        Port: intstr.FromInt(8080),
                                    },
                                },
                                InitialDelaySeconds: 5,
                                PeriodSeconds:       5,
                            },
                            Resources: corev1.ResourceRequirements{
                                Requests: corev1.ResourceList{
                                    corev1.ResourceCPU:    resource.MustParse("100m"),
                                    corev1.ResourceMemory: resource.MustParse("128Mi"),
                                },
                                Limits: corev1.ResourceList{
                                    corev1.ResourceCPU:    resource.MustParse("500m"),
                                    corev1.ResourceMemory: resource.MustParse("512Mi"),
                                },
                            },
                        },
                    },
                },
            },
        },
    }
    
    // Set MyApp instance as the owner
    ctrl.SetControllerReference(&myApp, deployment, r.Scheme)
    
    // Create or Update the deployment
    if err := r.Create(ctx, deployment); err != nil {
        if !errors.IsAlreadyExists(err) {
            return ctrl.Result{}, err
        }
        // Update existing deployment
        if err := r.Update(ctx, deployment); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // Update status
    myApp.Status.LastUpdate = time.Now().Format(time.RFC3339)
    if err := r.Status().Update(ctx, &myApp); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: time.Minute}, nil
}

func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        MetricsBindAddress:     ":8081",
        Port:                   9443,
        HealthProbeBindAddress: ":8082",
        LeaderElection:         true,
        LeaderElectionID:       "myapp-operator",
    })
    if err != nil {
        log.Fatal(err)
    }
    
    if err = (&MyAppReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        log.Fatal(err)
    }
    
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        log.Fatal(err)
    }
}
```

### Kubernetes Client

```go
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
)

func getKubernetesClient() (*kubernetes.Clientset, error) {
    // In-cluster config
    config, err := rest.InClusterConfig()
    if err != nil {
        // Fall back to kubeconfig
        kubeconfig := filepath.Join(os.Getenv("HOME"), ".kube", "config")
        config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
        if err != nil {
            return nil, err
        }
    }
    
    return kubernetes.NewForConfig(config)
}

func watchPods(ctx context.Context, client *kubernetes.Clientset) {
    watcher, err := client.CoreV1().Pods("default").Watch(ctx, metav1.ListOptions{})
    if err != nil {
        log.Fatal(err)
    }
    
    for event := range watcher.ResultChan() {
        pod, ok := event.Object.(*corev1.Pod)
        if !ok {
            continue
        }
        
        switch event.Type {
        case watch.Added:
            fmt.Printf("Pod added: %s\n", pod.Name)
        case watch.Modified:
            fmt.Printf("Pod modified: %s\n", pod.Name)
        case watch.Deleted:
            fmt.Printf("Pod deleted: %s\n", pod.Name)
        }
    }
}
```

## Observability

### OpenTelemetry Integration

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/metric"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
    oteltrace "go.opentelemetry.io/otel/trace"
)

func initTelemetry(ctx context.Context) (*trace.TracerProvider, error) {
    // Create resource
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("myapp"),
            semconv.ServiceVersion("1.0.0"),
            attribute.String("environment", os.Getenv("ENVIRONMENT")),
        ),
        resource.WithHost(),
        resource.WithContainer(),
    )
    if err != nil {
        return nil, err
    }
    
    // Create OTLP exporter
    exporter, err := otlptrace.New(
        ctx,
        otlptracegrpc.NewClient(
            otlptracegrpc.WithEndpoint(os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")),
            otlptracegrpc.WithInsecure(),
        ),
    )
    if err != nil {
        return nil, err
    }
    
    // Create trace provider
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(res),
        trace.WithSampler(trace.AlwaysSample()),
    )
    
    // Set global provider
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
    
    return tp, nil
}

// Instrumented HTTP handler
func instrumentedHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        
        // Extract trace context from headers
        ctx = otel.GetTextMapPropagator().Extract(ctx, propagation.HeaderCarrier(r.Header))
        
        // Start span
        tracer := otel.Tracer("http-server")
        ctx, span := tracer.Start(ctx, r.URL.Path,
            oteltrace.WithAttributes(
                semconv.HTTPMethod(r.Method),
                semconv.HTTPTarget(r.URL.String()),
                semconv.HTTPUserAgent(r.UserAgent()),
            ),
        )
        defer span.End()
        
        // Wrap response writer to capture status code
        wrapped := &statusRecorder{ResponseWriter: w, status: 200}
        
        // Call next handler
        next.ServeHTTP(wrapped, r.WithContext(ctx))
        
        // Set span attributes
        span.SetAttributes(
            semconv.HTTPStatusCode(wrapped.status),
        )
        
        if wrapped.status >= 500 {
            span.RecordError(fmt.Errorf("HTTP %d", wrapped.status))
        }
    })
}

// Metrics
func initMetrics() {
    meter := otel.Meter("myapp")
    
    // Create counter
    requestCounter, _ := meter.Int64Counter(
        "http_requests_total",
        metric.WithDescription("Total number of HTTP requests"),
    )
    
    // Create histogram
    requestDuration, _ := meter.Float64Histogram(
        "http_request_duration_seconds",
        metric.WithDescription("HTTP request duration in seconds"),
    )
    
    // Use in handler
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        defer func() {
            duration := time.Since(start).Seconds()
            
            labels := []attribute.KeyValue{
                attribute.String("method", r.Method),
                attribute.String("path", r.URL.Path),
            }
            
            requestCounter.Add(r.Context(), 1, metric.WithAttributes(labels...))
            requestDuration.Record(r.Context(), duration, metric.WithAttributes(labels...))
        }()
        
        // Handle request
    })
}
```

### Prometheus Metrics

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
    
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(httpRequestDuration)
    prometheus.MustRegister(activeConnections)
}

func prometheusMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        wrapped := &statusRecorder{ResponseWriter: w, status: 200}
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start).Seconds()
        
        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            fmt.Sprintf("%d", wrapped.status),
        ).Inc()
        
        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

func setupMetricsEndpoint() {
    http.Handle("/metrics", promhttp.Handler())
}
```

### Structured Logging for Cloud

```go
import (
    "log/slog"
    "go.opentelemetry.io/otel/trace"
)

type CloudLogger struct {
    logger *slog.Logger
}

func NewCloudLogger() *CloudLogger {
    opts := &slog.HandlerOptions{
        Level: slog.LevelInfo,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // Rename fields for cloud providers
            switch a.Key {
            case slog.TimeKey:
                return slog.Attr{Key: "timestamp", Value: a.Value}
            case slog.LevelKey:
                return slog.Attr{Key: "severity", Value: a.Value}
            case slog.MessageKey:
                return slog.Attr{Key: "message", Value: a.Value}
            }
            return a
        },
    }
    
    handler := slog.NewJSONHandler(os.Stdout, opts)
    return &CloudLogger{
        logger: slog.New(handler),
    }
}

func (l *CloudLogger) LogWithTrace(ctx context.Context, level slog.Level, msg string, args ...any) {
    span := trace.SpanFromContext(ctx)
    
    if span.SpanContext().IsValid() {
        args = append(args,
            "trace_id", span.SpanContext().TraceID().String(),
            "span_id", span.SpanContext().SpanID().String(),
        )
    }
    
    l.logger.Log(ctx, level, msg, args...)
}
```

## Service Mesh Integration

### Istio Support

```go
// Health checks for Istio
func setupHealthChecks() {
    http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })
    
    http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
        // Check dependencies
        if !isDatabaseReady() || !isCacheReady() {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    })
}

// Handle Envoy headers
func handleEnvoyHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract Envoy headers
        requestID := r.Header.Get("X-Request-Id")
        b3TraceID := r.Header.Get("X-B3-Traceid")
        b3SpanID := r.Header.Get("X-B3-Spanid")
        
        ctx := context.WithValue(r.Context(), "request_id", requestID)
        ctx = context.WithValue(ctx, "trace_id", b3TraceID)
        ctx = context.WithValue(ctx, "span_id", b3SpanID)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Serverless Go

### AWS Lambda

```go
package main

import (
    "context"
    "encoding/json"
    
    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
)

type Request struct {
    UserID string `json:"userId"`
    Action string `json:"action"`
}

type Response struct {
    StatusCode int         `json:"statusCode"`
    Body       string      `json:"body"`
    Headers    map[string]string `json:"headers"`
}

var dynamoClient *dynamodb.Client

func init() {
    cfg, err := config.LoadDefaultConfig(context.Background())
    if err != nil {
        panic(err)
    }
    dynamoClient = dynamodb.NewFromConfig(cfg)
}

func handler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    var req Request
    if err := json.Unmarshal([]byte(request.Body), &req); err != nil {
        return events.APIGatewayProxyResponse{
            StatusCode: 400,
            Body:       `{"error":"Invalid request"}`,
        }, nil
    }
    
    // Process request
    result, err := processRequest(ctx, req)
    if err != nil {
        return events.APIGatewayProxyResponse{
            StatusCode: 500,
            Body:       `{"error":"Internal server error"}`,
        }, nil
    }
    
    body, _ := json.Marshal(result)
    
    return events.APIGatewayProxyResponse{
        StatusCode: 200,
        Headers: map[string]string{
            "Content-Type": "application/json",
        },
        Body: string(body),
    }, nil
}

func main() {
    lambda.Start(handler)
}
```

### Google Cloud Functions

```go
package function

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    
    "cloud.google.com/go/firestore"
)

var firestoreClient *firestore.Client

func init() {
    ctx := context.Background()
    var err error
    firestoreClient, err = firestore.NewClient(ctx, "project-id")
    if err != nil {
        panic(err)
    }
}

func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Parse request
    var req Request
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }
    
    // Process with Firestore
    doc := firestoreClient.Collection("users").Doc(req.UserID)
    _, err := doc.Set(ctx, map[string]interface{}{
        "action":    req.Action,
        "timestamp": time.Now(),
    })
    if err != nil {
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "status": "success",
    })
}
```

## Configuration Management

### Environment-Based Config

```go
package config

import (
    "github.com/kelseyhightower/envconfig"
    "github.com/spf13/viper"
)

type Config struct {
    // Server
    Port         string `envconfig:"PORT" default:"8080"`
    Environment  string `envconfig:"ENVIRONMENT" default:"development"`
    
    // Database
    DatabaseURL  string `envconfig:"DATABASE_URL" required:"true"`
    MaxConns     int    `envconfig:"DB_MAX_CONNS" default:"25"`
    
    // Redis
    RedisURL     string `envconfig:"REDIS_URL" default:"localhost:6379"`
    
    // Cloud
    AWSRegion    string `envconfig:"AWS_REGION" default:"us-east-1"`
    GCPProject   string `envconfig:"GCP_PROJECT"`
    
    // Observability
    OTELEndpoint string `envconfig:"OTEL_ENDPOINT"`
    LogLevel     string `envconfig:"LOG_LEVEL" default:"info"`
    
    // Features
    Features     map[string]bool
}

func Load() (*Config, error) {
    var cfg Config
    
    // Load from environment
    if err := envconfig.Process("", &cfg); err != nil {
        return nil, err
    }
    
    // Load from config file if exists
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AddConfigPath("/etc/myapp/")
    
    if err := viper.ReadInConfig(); err == nil {
        // Merge with file config
        if err := viper.Unmarshal(&cfg); err != nil {
            return nil, err
        }
    }
    
    // Load from Kubernetes ConfigMap
    if _, err := os.Stat("/config/app.yaml"); err == nil {
        viper.SetConfigFile("/config/app.yaml")
        if err := viper.MergeInConfig(); err != nil {
            return nil, err
        }
    }
    
    return &cfg, nil
}
```

### Secret Management

```go
import (
    "github.com/aws/aws-sdk-go-v2/service/secretsmanager"
    "google.golang.org/api/secretmanager/v1"
)

type SecretManager interface {
    GetSecret(ctx context.Context, name string) (string, error)
}

// AWS Secrets Manager
type AWSSecretManager struct {
    client *secretsmanager.Client
}

func (m *AWSSecretManager) GetSecret(ctx context.Context, name string) (string, error) {
    result, err := m.client.GetSecretValue(ctx, &secretsmanager.GetSecretValueInput{
        SecretId: &name,
    })
    if err != nil {
        return "", err
    }
    return *result.SecretString, nil
}

// Google Secret Manager
type GCPSecretManager struct {
    client *secretmanager.Service
}

func (m *GCPSecretManager) GetSecret(ctx context.Context, name string) (string, error) {
    resp, err := m.client.Projects.Secrets.Versions.Access(name).Do()
    if err != nil {
        return "", err
    }
    
    data, err := base64.StdEncoding.DecodeString(resp.Payload.Data)
    if err != nil {
        return "", err
    }
    
    return string(data), nil
}

// Kubernetes Secrets
func getKubernetesSecret(name string) (string, error) {
    path := filepath.Join("/var/run/secrets", name)
    data, err := os.ReadFile(path)
    if err != nil {
        return "", err
    }
    return string(data), nil
}
```

## Circuit Breaker and Resilience

```go
import (
    "github.com/sony/gobreaker"
)

func setupCircuitBreaker() *gobreaker.CircuitBreaker {
    settings := gobreaker.Settings{
        Name:        "API",
        MaxRequests: 3,
        Interval:    time.Minute,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            slog.Info("circuit breaker state change",
                "name", name,
                "from", from,
                "to", to,
            )
        },
    }
    
    return gobreaker.NewCircuitBreaker(settings)
}

func callWithCircuitBreaker(cb *gobreaker.CircuitBreaker, fn func() (interface{}, error)) (interface{}, error) {
    result, err := cb.Execute(func() (interface{}, error) {
        return fn()
    })
    
    if err != nil {
        if err == gobreaker.ErrOpenState {
            return nil, fmt.Errorf("service unavailable: circuit open")
        }
        if err == gobreaker.ErrTooManyRequests {
            return nil, fmt.Errorf("service busy: too many requests")
        }
        return nil, err
    }
    
    return result, nil
}
```

## Best Practices

### 1. Health Checks
```go
// Comprehensive health check
func (s *Server) healthCheck() HealthStatus {
    return HealthStatus{
        Status: "healthy",
        Checks: map[string]bool{
            "database": s.db.Ping() == nil,
            "cache":    s.cache.Ping() == nil,
            "storage":  s.storage.Available(),
        },
    }
}
```

### 2. Graceful Degradation
```go
// Fallback when services are unavailable
if err := primaryService.Call(); err != nil {
    slog.Warn("primary service failed, using fallback")
    return fallbackService.Call()
}
```

### 3. Resource Limits
```go
// Set appropriate resource limits
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 4. Observability First
```go
// Instrument everything
ctx, span := tracer.Start(ctx, "operation")
defer span.End()

counter.Inc()
histogram.Observe(duration)
```

## Exercises

1. **Build a Kubernetes Operator**: Create an operator that manages a custom resource.

2. **Implement Service Mesh**: Add Istio support with proper observability.

3. **Serverless Function**: Deploy a Go function to AWS Lambda, Google Cloud Functions, and Azure Functions.

4. **Cloud-Native API**: Build a fully observable API with metrics, tracing, and logging.

5. **Multi-Cloud Application**: Create an application that works across AWS, GCP, and Azure.

## Summary

Go and cloud-native are inseparable. From container optimization to Kubernetes operators, from observability to serverless, Go provides the tools and patterns for building robust cloud applications. The key is understanding not just the technology, but the cloud-native principles: observability, resilience, scalability, and automation.

Key takeaways:
- Optimize containers with multi-stage builds and distroless images
- Kubernetes operators extend cluster functionality
- Observability is non-negotiable in cloud environments
- Service mesh integration provides advanced networking features
- Serverless Go offers rapid scaling and reduced operational overhead

Next, we'll explore security considerations for modern Go applications.

---

*Continue to [Chapter 18: Security in Modern Go](chapter18.md)*