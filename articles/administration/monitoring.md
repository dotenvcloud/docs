---
title: Monitoring
slug: monitoring
order: 3
tags: [administration, monitoring, metrics, alerting, observability]
---

# Monitoring

This guide covers monitoring strategies for DotEnv, including system metrics, application monitoring, alerting configurations, and observability practices. Learn how to ensure your DotEnv instance runs reliably and efficiently.

## Monitoring Architecture

### Observability Stack

```
┌─────────────────────────────────────────┐
│         Data Sources                    │
│  • Application Metrics                 │
│  • System Metrics                      │
│  • Logs                               │
│  • Traces                             │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Collection Layer                │
│  • Prometheus (Metrics)                │
│  • Elasticsearch (Logs)                │
│  • Jaeger (Traces)                     │
│  • StatsD (Application)                │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Storage & Processing            │
│  • Time Series Database                │
│  • Log Aggregation                     │
│  • Trace Analysis                      │
│  • Anomaly Detection                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│       Visualization & Alerting          │
│  • Grafana Dashboards                  │
│  • Alert Manager                       │
│  • Status Pages                        │
│  • Reports                             │
└─────────────────────────────────────────┘
```

## Metrics Collection

### Application Metrics

```javascript
// Application metrics configuration
class MetricsCollector {
    constructor() {
        this.client = new StatsD({
            host: process.env.STATSD_HOST,
            port: 8125,
            prefix: "dotenv.",
        });

        this.metrics = {
            // Request metrics
            http_requests_total: new Counter({
                name: "http_requests_total",
                help: "Total HTTP requests",
                labelNames: ["method", "route", "status"],
            }),

            http_request_duration: new Histogram({
                name: "http_request_duration_seconds",
                help: "HTTP request latency",
                labelNames: ["method", "route"],
                buckets: [0.1, 0.5, 1, 2, 5],
            }),

            // Business metrics
            secrets_accessed: new Counter({
                name: "secrets_accessed_total",
                help: "Total secrets accessed",
                labelNames: ["project", "environment", "method"],
            }),

            api_calls: new Counter({
                name: "api_calls_total",
                help: "Total API calls",
                labelNames: ["endpoint", "client_type", "status"],
            }),

            // System metrics
            active_users: new Gauge({
                name: "active_users",
                help: "Number of active users",
            }),

            database_connections: new Gauge({
                name: "database_connections",
                help: "Active database connections",
                labelNames: ["pool"],
            }),
        };
    }

    recordRequest(req, res, duration) {
        const labels = {
            method: req.method,
            route: req.route?.path || "unknown",
            status: res.statusCode,
        };

        this.metrics.http_requests_total.inc(labels);
        this.metrics.http_request_duration.observe(
            { method: labels.method, route: labels.route },
            duration / 1000,
        );

        // StatsD metrics
        this.client.increment(`requests.${labels.method}.${labels.status}`);
        this.client.timing("request.duration", duration);
    }

    recordSecretAccess(project, environment, method) {
        this.metrics.secrets_accessed.inc({
            project,
            environment,
            method,
        });

        this.client.increment(`secrets.${method}.${project}.${environment}`);
    }
}

// Middleware for automatic metric collection
const metricsMiddleware = (req, res, next) => {
    const start = Date.now();

    res.on("finish", () => {
        const duration = Date.now() - start;
        metricsCollector.recordRequest(req, res, duration);
    });

    next();
};
```

### System Metrics

```javascript
// System metrics monitoring
class SystemMetrics {
    constructor() {
        this.interval = 30000; // 30 seconds
        this.metrics = {};
    }

    async collectMetrics() {
        // CPU metrics
        const cpuUsage = process.cpuUsage();
        this.metrics.cpu_usage_user = cpuUsage.user / 1000000;
        this.metrics.cpu_usage_system = cpuUsage.system / 1000000;

        // Memory metrics
        const memUsage = process.memoryUsage();
        this.metrics.memory_heap_used = memUsage.heapUsed;
        this.metrics.memory_heap_total = memUsage.heapTotal;
        this.metrics.memory_rss = memUsage.rss;
        this.metrics.memory_external = memUsage.external;

        // Event loop metrics
        this.metrics.event_loop_lag = await this.measureEventLoopLag();

        // Database metrics
        const dbStats = await this.getDatabaseStats();
        this.metrics.db_connections_active = dbStats.active;
        this.metrics.db_connections_idle = dbStats.idle;
        this.metrics.db_connections_waiting = dbStats.waiting;

        // Redis metrics
        const redisInfo = await redis.info();
        this.metrics.redis_memory_used = redisInfo.used_memory;
        this.metrics.redis_connected_clients = redisInfo.connected_clients;
        this.metrics.redis_ops_per_sec = redisInfo.instantaneous_ops_per_sec;

        return this.metrics;
    }

    measureEventLoopLag() {
        return new Promise((resolve) => {
            const start = process.hrtime.bigint();
            setImmediate(() => {
                const lag = Number(process.hrtime.bigint() - start) / 1000000;
                resolve(lag);
            });
        });
    }

    startCollection() {
        setInterval(async () => {
            const metrics = await this.collectMetrics();

            // Export to Prometheus
            Object.entries(metrics).forEach(([name, value]) => {
                if (typeof value === "number") {
                    prometheus.gauge(name, value);
                }
            });
        }, this.interval);
    }
}
```

## Log Management

### Structured Logging

```javascript
// Structured logging configuration
const winston = require("winston");
const { ElasticsearchTransport } = require("winston-elasticsearch");

const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json(),
    ),

    defaultMeta: {
        service: "dotenv-api",
        environment: process.env.NODE_ENV,
        version: process.env.APP_VERSION,
    },

    transports: [
        // Console transport for development
        new winston.transports.Console({
            format: winston.format.combine(
                winston.format.colorize(),
                winston.format.simple(),
            ),
        }),

        // Elasticsearch transport for production
        new ElasticsearchTransport({
            level: "info",
            clientOpts: {
                node: process.env.ELASTICSEARCH_URL,
                auth: {
                    username: process.env.ELASTICSEARCH_USER,
                    password: process.env.ELASTICSEARCH_PASS,
                },
            },
            index: "dotenv-logs",
            dataStream: true,
        }),

        // File transport for backup
        new winston.transports.File({
            filename: "logs/error.log",
            level: "error",
            maxsize: 10485760, // 10MB
            maxFiles: 5,
        }),
    ],
});

// Request logging
const requestLogger = (req, res, next) => {
    const start = Date.now();

    res.on("finish", () => {
        logger.info("HTTP Request", {
            method: req.method,
            url: req.url,
            status: res.statusCode,
            duration: Date.now() - start,
            ip: req.ip,
            user_agent: req.get("user-agent"),
            user_id: req.user?.id,
            request_id: req.id,
        });
    });

    next();
};

// Error logging
const errorLogger = (err, req, res, next) => {
    logger.error("Application Error", {
        error: {
            message: err.message,
            stack: err.stack,
            code: err.code,
        },
        request: {
            method: req.method,
            url: req.url,
            body: req.body,
            user_id: req.user?.id,
        },
    });

    next(err);
};
```

### Log Aggregation

```javascript
// Log aggregation pipeline
class LogAggregator {
    constructor() {
        this.pipeline = {
            inputs: {
                application: "/var/log/dotenv/*.log",
                nginx: "/var/log/nginx/*.log",
                system: "/var/log/syslog",
            },

            filters: [
                {
                    type: "grok",
                    pattern: "%{COMBINEDAPACHELOG}",
                },
                {
                    type: "date",
                    match: ["timestamp", "ISO8601"],
                },
                {
                    type: "geoip",
                    source: "client_ip",
                },
                {
                    type: "mutate",
                    add_field: {
                        service: "dotenv",
                        environment: process.env.NODE_ENV,
                    },
                },
            ],

            outputs: {
                elasticsearch: {
                    hosts: [process.env.ELASTICSEARCH_URL],
                    index: "dotenv-logs-%{+YYYY.MM.dd}",
                },

                s3: {
                    bucket: "dotenv-logs-archive",
                    prefix: "logs/%{+YYYY/MM/dd}/",
                    compression: "gzip",
                },
            },
        };
    }

    async searchLogs(query) {
        const results = await elasticsearch.search({
            index: "dotenv-logs-*",
            body: {
                query: {
                    bool: {
                        must: [
                            { match: { message: query.text } },
                            {
                                range: {
                                    "@timestamp": {
                                        gte: query.from,
                                        lte: query.to,
                                    },
                                },
                            },
                        ],
                        filter: query.filters || [],
                    },
                },
                aggs: {
                    status_codes: {
                        terms: { field: "status" },
                    },
                    response_times: {
                        percentiles: { field: "duration" },
                    },
                },
                size: query.size || 100,
                sort: [{ "@timestamp": "desc" }],
            },
        });

        return results;
    }
}
```

## Distributed Tracing

### Trace Implementation

```javascript
// OpenTelemetry tracing setup
const { NodeTracerProvider } = require("@opentelemetry/sdk-trace-node");
const { JaegerExporter } = require("@opentelemetry/exporter-jaeger");
const { BatchSpanProcessor } = require("@opentelemetry/sdk-trace-base");
const { registerInstrumentations } = require("@opentelemetry/instrumentation");

// Initialize tracer
const provider = new NodeTracerProvider({
    resource: {
        attributes: {
            "service.name": "dotenv-api",
            "service.version": process.env.APP_VERSION,
            "deployment.environment": process.env.NODE_ENV,
        },
    },
});

// Configure exporter
const jaegerExporter = new JaegerExporter({
    endpoint: process.env.JAEGER_ENDPOINT,
    serviceName: "dotenv-api",
});

provider.addSpanProcessor(new BatchSpanProcessor(jaegerExporter));
provider.register();

// Auto-instrumentation
registerInstrumentations({
    instrumentations: [
        new HttpInstrumentation({
            requestHook: (span, request) => {
                span.setAttributes({
                    "http.request.body": JSON.stringify(request.body),
                    "user.id": request.user?.id,
                });
            },
        }),
        new ExpressInstrumentation(),
        new RedisInstrumentation(),
        new PostgresInstrumentation(),
    ],
});

// Manual instrumentation example
const tracer = provider.getTracer("dotenv-api");

async function encryptSecret(projectId, secret) {
    const span = tracer.startSpan("encrypt_secret", {
        attributes: {
            "project.id": projectId,
            "secret.name": secret.name,
        },
    });

    try {
        // Trace key retrieval
        const keySpan = tracer.startSpan("retrieve_encryption_key", {
            parent: span,
        });
        const key = await getEncryptionKey(projectId);
        keySpan.end();

        // Trace encryption
        const encryptSpan = tracer.startSpan("perform_encryption", {
            parent: span,
        });
        const encrypted = await encrypt(secret.value, key);
        encryptSpan.setAttributes({
            "encryption.algorithm": "AES-256-GCM",
            "encryption.key_version": key.version,
        });
        encryptSpan.end();

        span.setStatus({ code: SpanStatusCode.OK });
        return encrypted;
    } catch (error) {
        span.setStatus({
            code: SpanStatusCode.ERROR,
            message: error.message,
        });
        throw error;
    } finally {
        span.end();
    }
}
```

## Dashboard Configuration

### Grafana Dashboards

```json
{
    "dashboard": {
        "title": "DotEnv System Overview",
        "panels": [
            {
                "title": "Request Rate",
                "targets": [
                    {
                        "expr": "rate(http_requests_total[5m])",
                        "legendFormat": "{{method}} {{route}}"
                    }
                ],
                "type": "graph"
            },
            {
                "title": "Response Time (p95)",
                "targets": [
                    {
                        "expr": "histogram_quantile(0.95, rate(http_request_duration_bucket[5m]))",
                        "legendFormat": "95th percentile"
                    }
                ],
                "type": "graph"
            },
            {
                "title": "Error Rate",
                "targets": [
                    {
                        "expr": "rate(http_requests_total{status=~'5..'}[5m])",
                        "legendFormat": "5xx errors"
                    }
                ],
                "type": "graph",
                "alert": {
                    "condition": "avg() > 0.01",
                    "message": "Error rate above 1%"
                }
            },
            {
                "title": "Active Users",
                "targets": [
                    {
                        "expr": "active_users",
                        "legendFormat": "Users"
                    }
                ],
                "type": "stat"
            },
            {
                "title": "Database Connections",
                "targets": [
                    {
                        "expr": "database_connections",
                        "legendFormat": "{{pool}}"
                    }
                ],
                "type": "graph"
            },
            {
                "title": "Memory Usage",
                "targets": [
                    {
                        "expr": "memory_heap_used / memory_heap_total * 100",
                        "legendFormat": "Heap Usage %"
                    }
                ],
                "type": "gauge"
            }
        ]
    }
}
```

### Custom Dashboards

```javascript
// Dashboard configuration manager
class DashboardManager {
    async createSecurityDashboard() {
        const dashboard = {
            title: "Security Monitoring",
            refresh: "30s",
            panels: [
                {
                    title: "Failed Login Attempts",
                    query: "sum(rate(auth_failures_total[5m])) by (reason)",
                    type: "graph",
                    alert: {
                        condition: "last() > 10",
                        for: "5m",
                        severity: "warning",
                    },
                },
                {
                    title: "API Key Usage",
                    query: "sum(rate(api_key_usage[1h])) by (key_id)",
                    type: "heatmap",
                },
                {
                    title: "Encryption Operations",
                    query: "rate(encryption_operations_total[5m])",
                    type: "graph",
                    splitBy: "operation",
                },
                {
                    title: "Suspicious Activities",
                    query: "security_anomaly_score",
                    type: "timeseries",
                    threshold: {
                        value: 0.8,
                        color: "red",
                    },
                },
            ],
        };

        return await this.saveDashboard(dashboard);
    }

    async createBusinessDashboard() {
        const dashboard = {
            title: "Business Metrics",
            refresh: "5m",
            panels: [
                {
                    title: "API Usage by Plan",
                    query: "sum(api_calls_total) by (plan)",
                    type: "piechart",
                },
                {
                    title: "Secrets Created",
                    query: "increase(secrets_created_total[1d])",
                    type: "stat",
                },
                {
                    title: "Active Projects",
                    query: "count(count by (project)(secrets_accessed_total))",
                    type: "stat",
                },
                {
                    title: "User Growth",
                    query: "increase(users_total[30d])",
                    type: "graph",
                },
            ],
        };

        return await this.saveDashboard(dashboard);
    }
}
```

## Alerting

### Alert Rules

```yaml
# alerting-rules.yaml
groups:
    - name: application
      interval: 30s
      rules:
          - alert: HighErrorRate
            expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
            for: 5m
            labels:
                severity: critical
                team: backend
            annotations:
                summary: "High error rate detected"
                description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.route }}"

          - alert: SlowResponseTime
            expr: histogram_quantile(0.95, http_request_duration_bucket) > 2
            for: 10m
            labels:
                severity: warning
                team: backend
            annotations:
                summary: "Slow response times"
                description: "95th percentile response time is {{ $value }}s"

    - name: infrastructure
      interval: 60s
      rules:
          - alert: HighMemoryUsage
            expr: (memory_heap_used / memory_heap_total) > 0.9
            for: 5m
            labels:
                severity: warning
                team: ops
            annotations:
                summary: "High memory usage"
                description: "Memory usage is {{ $value | humanizePercentage }}"

          - alert: DatabaseConnectionPoolExhausted
            expr: database_connections_waiting > 10
            for: 2m
            labels:
                severity: critical
                team: ops
            annotations:
                summary: "Database connection pool exhausted"
                description: "{{ $value }} connections waiting"

    - name: security
      interval: 30s
      rules:
          - alert: BruteForceAttempt
            expr: sum(rate(auth_failures_total[5m])) by (ip) > 10
            for: 2m
            labels:
                severity: high
                team: security
            annotations:
                summary: "Possible brute force attempt"
                description: "{{ $value }} failed attempts from {{ $labels.ip }}"

          - alert: AnomalousSecretAccess
            expr: rate(secrets_accessed_total[5m]) > 100
            for: 5m
            labels:
                severity: warning
                team: security
            annotations:
                summary: "Anomalous secret access pattern"
                description: "{{ $value }} secrets/sec accessed"
```

### Alert Manager Configuration

```javascript
// Alert manager setup
class AlertManager {
    constructor() {
        this.channels = {
            email: new EmailChannel(),
            slack: new SlackChannel(),
            pagerduty: new PagerDutyChannel(),
            webhook: new WebhookChannel(),
        };

        this.routes = [
            {
                match: { severity: "critical" },
                channels: ["pagerduty", "slack", "email"],
                repeat_interval: "5m",
            },
            {
                match: { severity: "high" },
                channels: ["slack", "email"],
                repeat_interval: "30m",
            },
            {
                match: { severity: "warning" },
                channels: ["slack"],
                repeat_interval: "2h",
            },
        ];
    }

    async sendAlert(alert) {
        const route = this.findRoute(alert);

        for (const channelName of route.channels) {
            const channel = this.channels[channelName];

            try {
                await channel.send({
                    title: alert.annotations.summary,
                    description: alert.annotations.description,
                    severity: alert.labels.severity,
                    source: alert.generatorURL,
                    fingerprint: alert.fingerprint,
                    startsAt: alert.startsAt,
                    labels: alert.labels,
                });
            } catch (error) {
                logger.error("Alert delivery failed", {
                    channel: channelName,
                    alert: alert.fingerprint,
                    error: error.message,
                });
            }
        }

        // Record alert
        await this.recordAlert(alert);
    }

    async recordAlert(alert) {
        await db.alerts.create({
            fingerprint: alert.fingerprint,
            name: alert.labels.alertname,
            severity: alert.labels.severity,
            status: alert.status,
            starts_at: alert.startsAt,
            ends_at: alert.endsAt,
            labels: alert.labels,
            annotations: alert.annotations,
        });
    }
}
```

## Health Checks

### Application Health

```javascript
// Health check implementation
class HealthChecker {
    constructor() {
        this.checks = {
            database: this.checkDatabase,
            redis: this.checkRedis,
            storage: this.checkStorage,
            encryption: this.checkEncryption,
            api: this.checkAPI,
        };
    }

    async performHealthCheck() {
        const results = {
            status: "healthy",
            timestamp: new Date(),
            checks: {},
        };

        // Run all checks in parallel
        const checkPromises = Object.entries(this.checks).map(
            async ([name, check]) => {
                const start = Date.now();

                try {
                    const result = await check.call(this);
                    results.checks[name] = {
                        status: "healthy",
                        duration: Date.now() - start,
                        details: result,
                    };
                } catch (error) {
                    results.checks[name] = {
                        status: "unhealthy",
                        duration: Date.now() - start,
                        error: error.message,
                    };
                    results.status = "unhealthy";
                }
            },
        );

        await Promise.all(checkPromises);

        return results;
    }

    async checkDatabase() {
        const result = await db.raw("SELECT 1");

        const stats = await db.raw(`
      SELECT 
        count(*) as connections,
        count(*) FILTER (WHERE state = 'active') as active,
        count(*) FILTER (WHERE state = 'idle') as idle
      FROM pg_stat_activity
      WHERE datname = current_database()
    `);

        return {
            connected: true,
            connections: stats.rows[0],
        };
    }

    async checkEncryption() {
        // Test encryption/decryption
        const testData = "health-check-test";
        const encrypted = await encrypt(testData);
        const decrypted = await decrypt(encrypted);

        if (decrypted !== testData) {
            throw new Error("Encryption check failed");
        }

        return {
            functional: true,
            algorithm: "AES-256-GCM",
        };
    }
}

// Health check endpoint
app.get("/health", async (req, res) => {
    const health = await healthChecker.performHealthCheck();

    res.status(health.status === "healthy" ? 200 : 503).json(health);
});
```

## Performance Monitoring

### APM Integration

```javascript
// Application Performance Monitoring
const apm = require("elastic-apm-node").start({
    serviceName: "dotenv-api",
    secretToken: process.env.APM_TOKEN,
    serverUrl: process.env.APM_SERVER,
    environment: process.env.NODE_ENV,

    // Capture body for debugging
    captureBody: "errors",

    // Custom context
    globalLabels: {
        region: process.env.AWS_REGION,
        deployment: process.env.DEPLOYMENT_ID,
    },

    // Performance tuning
    transactionSampleRate: 0.1,
    maxQueueSize: 1000,
});

// Custom transaction tracking
async function trackTransaction(name, type, fn) {
    const transaction = apm.startTransaction(name, type);

    try {
        const result = await fn();
        transaction.result = "success";
        return result;
    } catch (error) {
        apm.captureError(error);
        transaction.result = "error";
        throw error;
    } finally {
        transaction.end();
    }
}

// Performance metrics collection
class PerformanceMonitor {
    constructor() {
        this.metrics = new Map();
    }

    startTimer(name) {
        const timer = {
            name,
            start: process.hrtime.bigint(),
        };

        return timer;
    }

    endTimer(timer) {
        const duration =
            Number(process.hrtime.bigint() - timer.start) / 1000000;

        if (!this.metrics.has(timer.name)) {
            this.metrics.set(timer.name, {
                count: 0,
                total: 0,
                min: Infinity,
                max: 0,
                p50: 0,
                p95: 0,
                p99: 0,
                samples: [],
            });
        }

        const metric = this.metrics.get(timer.name);
        metric.count++;
        metric.total += duration;
        metric.min = Math.min(metric.min, duration);
        metric.max = Math.max(metric.max, duration);
        metric.samples.push(duration);

        // Keep only last 1000 samples
        if (metric.samples.length > 1000) {
            metric.samples.shift();
        }

        // Calculate percentiles
        const sorted = [...metric.samples].sort((a, b) => a - b);
        metric.p50 = sorted[Math.floor(sorted.length * 0.5)];
        metric.p95 = sorted[Math.floor(sorted.length * 0.95)];
        metric.p99 = sorted[Math.floor(sorted.length * 0.99)];

        return duration;
    }

    getMetrics() {
        const results = {};

        for (const [name, metric] of this.metrics) {
            results[name] = {
                count: metric.count,
                average: metric.total / metric.count,
                min: metric.min,
                max: metric.max,
                p50: metric.p50,
                p95: metric.p95,
                p99: metric.p99,
            };
        }

        return results;
    }
}
```

## Best Practices

### 1. Comprehensive Coverage

```javascript
// ✅ Good: Monitor all critical paths
const monitoring = {
    infrastructure: ["cpu", "memory", "disk", "network"],
    application: ["requests", "errors", "latency", "throughput"],
    business: ["users", "api_calls", "secrets_accessed"],
    security: ["auth_failures", "anomalies", "audit_events"],
};

// ❌ Bad: Only monitoring basic metrics
const monitoring = {
    metrics: ["cpu", "memory"],
};
```

### 2. Actionable Alerts

```javascript
// ✅ Good: Specific, actionable alerts
const alert = {
    name: "DatabaseConnectionPoolExhausted",
    condition: "connections_waiting > 10",
    playbook: "https://wiki/database-connection-pool-exhausted",
    escalation: ["oncall-dba", "backend-team"],
};

// ❌ Bad: Vague alerts
const alert = {
    name: "SomethingWrong",
    condition: "error_count > 0",
};
```

### 3. Correlation and Context

```javascript
// ✅ Good: Correlated metrics with context
logger.info("Request processed", {
    request_id: req.id,
    user_id: user.id,
    trace_id: span.traceId,
    duration: duration,
    business_context: {
        project: project.id,
        action: "secret_access",
    },
});

// ❌ Bad: Isolated metrics
console.log("Request took", duration, "ms");
```

## Resources

- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana Dashboard Guide](https://grafana.com/docs/grafana/latest/dashboards/)
- [Elastic APM Guide](https://www.elastic.co/guide/en/apm/guide/current/index.html)

## Next Steps

- [Configure alerting rules](/documentation/v1/administration/alerting)
- [Set up log analysis](/documentation/v1/administration/log-analysis)
- [Plan capacity and scaling](/documentation/v1/administration/scaling)
- [Review security monitoring](/documentation/v1/security-compliance/audit-logging)
