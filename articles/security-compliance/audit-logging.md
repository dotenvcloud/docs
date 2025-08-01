---
title: Audit Logging
slug: audit-logging
order: 4
tags: [security, audit, logging, compliance, monitoring]
---

# Audit Logging

DotEnv provides comprehensive audit logging to track all security-relevant events, ensure compliance, and enable forensic analysis. This guide covers our audit logging architecture, event types, and analysis tools.

## Audit Logging Architecture

### Event Flow

```
┌─────────────────────────────────────────┐
│           Event Sources                 │
│  • API Requests                        │
│  • User Actions                        │
│  • System Changes                      │
│  • Security Events                     │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Event Processor                 │
│  • Enrichment                          │
│  • Classification                      │
│  • Filtering                           │
│  • Validation                          │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Storage Layer                   │
│  • Immutable Storage                   │
│  • Encryption                          │
│  • Replication                         │
│  • Retention                           │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│        Analysis Tools                   │
│  • Real-time Monitoring                │
│  • Compliance Reports                  │
│  • Forensic Analysis                   │
│  • Alerting                           │
└─────────────────────────────────────────┘
```

## Event Types

### 1. Authentication Events

```javascript
// Authentication event schema
const authenticationEvents = {
    "auth.login": {
        description: "User login attempt",
        severity: "info",
        data: {
            user_id: "string",
            email: "string",
            method: "password|sso|api_key",
            ip_address: "string",
            user_agent: "string",
            success: "boolean",
            mfa_used: "boolean",
            failure_reason: "string?",
        },
    },

    "auth.logout": {
        description: "User logout",
        severity: "info",
        data: {
            user_id: "string",
            session_duration: "number",
            voluntary: "boolean",
        },
    },

    "auth.mfa_enabled": {
        description: "MFA enabled for account",
        severity: "info",
        data: {
            user_id: "string",
            mfa_type: "totp|sms|webauthn",
            backup_codes_generated: "number",
        },
    },

    "auth.password_changed": {
        description: "Password changed",
        severity: "medium",
        data: {
            user_id: "string",
            changed_by: "self|admin|system",
            required: "boolean",
        },
    },

    "auth.failed_attempts": {
        description: "Multiple failed login attempts",
        severity: "high",
        data: {
            identifier: "string", // email or username
            attempt_count: "number",
            ip_addresses: "string[]",
            locked: "boolean",
        },
    },
};
```

### 2. Authorization Events

```javascript
// Authorization event schema
const authorizationEvents = {
    "authz.permission_granted": {
        description: "Permission granted to user",
        severity: "medium",
        data: {
            user_id: "string",
            permission: "string",
            resource_type: "string",
            resource_id: "string",
            granted_by: "string",
            reason: "string",
        },
    },

    "authz.permission_revoked": {
        description: "Permission revoked from user",
        severity: "medium",
        data: {
            user_id: "string",
            permission: "string",
            resource_type: "string",
            resource_id: "string",
            revoked_by: "string",
            reason: "string",
        },
    },

    "authz.role_assigned": {
        description: "Role assigned to user",
        severity: "medium",
        data: {
            user_id: "string",
            role: "string",
            scope: "organization|project|team",
            scope_id: "string",
            assigned_by: "string",
        },
    },

    "authz.access_denied": {
        description: "Access attempt denied",
        severity: "medium",
        data: {
            user_id: "string",
            resource_type: "string",
            resource_id: "string",
            action: "string",
            reason: "string",
        },
    },
};
```

### 3. Secret Management Events

```javascript
// Secret management event schema
const secretEvents = {
    "secret.created": {
        description: "New secret created",
        severity: "medium",
        data: {
            secret_id: "string",
            project_id: "string",
            environment: "string",
            created_by: "string",
            encrypted: "boolean",
            key_type: "server|client",
        },
    },

    "secret.read": {
        description: "Secret value accessed",
        severity: "info",
        data: {
            secret_id: "string",
            project_id: "string",
            environment: "string",
            accessed_by: "string",
            method: "api|cli|web",
            decrypted: "boolean",
        },
    },

    "secret.updated": {
        description: "Secret value modified",
        severity: "high",
        data: {
            secret_id: "string",
            project_id: "string",
            environment: "string",
            updated_by: "string",
            changes: {
                value_changed: "boolean",
                metadata_changed: "boolean",
            },
        },
    },

    "secret.deleted": {
        description: "Secret deleted",
        severity: "high",
        data: {
            secret_id: "string",
            project_id: "string",
            environment: "string",
            deleted_by: "string",
            soft_delete: "boolean",
        },
    },

    "secret.bulk_operation": {
        description: "Bulk secret operation",
        severity: "high",
        data: {
            operation: "import|export|rotate",
            project_id: "string",
            environment: "string",
            secret_count: "number",
            performed_by: "string",
        },
    },
};
```

### 4. Key Management Events

```javascript
// Key management event schema
const keyEvents = {
    "key.generated": {
        description: "Encryption key generated",
        severity: "high",
        data: {
            key_id: "string",
            project_id: "string",
            algorithm: "string",
            key_type: "server|client",
            generated_by: "string",
        },
    },

    "key.rotated": {
        description: "Encryption key rotated",
        severity: "high",
        data: {
            old_key_id: "string",
            new_key_id: "string",
            project_id: "string",
            secrets_affected: "number",
            rotated_by: "string",
            reason: "scheduled|manual|compromise",
        },
    },

    "key.accessed": {
        description: "Encryption key retrieved",
        severity: "medium",
        data: {
            key_id: "string",
            project_id: "string",
            accessed_by: "string",
            purpose: "encryption|decryption|validation",
        },
    },

    "key.exported": {
        description: "Encryption key exported",
        severity: "critical",
        data: {
            key_id: "string",
            project_id: "string",
            exported_by: "string",
            destination: "string",
            approval_id: "string?",
        },
    },
};
```

### 5. System Events

```javascript
// System event schema
const systemEvents = {
    "system.configuration_changed": {
        description: "System configuration modified",
        severity: "high",
        data: {
            component: "string",
            setting: "string",
            old_value: "any",
            new_value: "any",
            changed_by: "string",
        },
    },

    "system.integration_added": {
        description: "New integration configured",
        severity: "medium",
        data: {
            integration_type: "string",
            integration_id: "string",
            configured_by: "string",
            permissions: "string[]",
        },
    },

    "system.backup_created": {
        description: "System backup created",
        severity: "info",
        data: {
            backup_id: "string",
            backup_type: "full|incremental",
            size_bytes: "number",
            duration_ms: "number",
        },
    },

    "system.anomaly_detected": {
        description: "Anomalous behavior detected",
        severity: "critical",
        data: {
            anomaly_type: "string",
            confidence: "number",
            details: "object",
            auto_response: "string?",
        },
    },
};
```

## Event Processing

### 1. Event Collection

```javascript
// Event collector implementation
class AuditEventCollector {
    constructor() {
        this.queue = new EventQueue();
        this.enrichers = [];
        this.filters = [];
    }

    async collectEvent(eventType, data, context) {
        // Create base event
        const event = {
            id: uuidv4(),
            timestamp: new Date().toISOString(),
            type: eventType,
            data: data,
            context: {
                request_id: context.requestId,
                session_id: context.sessionId,
                ip_address: context.ipAddress,
                user_agent: context.userAgent,
            },
        };

        // Enrich event
        for (const enricher of this.enrichers) {
            await enricher.enrich(event);
        }

        // Apply filters
        for (const filter of this.filters) {
            if (!filter.shouldLog(event)) {
                return null;
            }
        }

        // Queue for processing
        await this.queue.push(event);

        return event;
    }

    addEnricher(enricher) {
        this.enrichers.push(enricher);
    }

    addFilter(filter) {
        this.filters.push(filter);
    }
}

// Event enrichers
class GeoIPEnricher {
    async enrich(event) {
        if (event.context.ip_address) {
            const geoData = await this.lookupGeoIP(event.context.ip_address);
            event.context.geo = {
                country: geoData.country,
                region: geoData.region,
                city: geoData.city,
                latitude: geoData.latitude,
                longitude: geoData.longitude,
            };
        }
    }
}

class UserContextEnricher {
    async enrich(event) {
        if (event.data.user_id) {
            const user = await this.getUser(event.data.user_id);
            event.context.user = {
                email: user.email,
                roles: user.roles,
                organization_id: user.organization_id,
                created_at: user.created_at,
            };
        }
    }
}

class ThreatIntelligenceEnricher {
    async enrich(event) {
        if (event.context.ip_address) {
            const threatData = await this.checkThreatIntel(
                event.context.ip_address,
            );
            if (threatData.isThreat) {
                event.context.threat = {
                    score: threatData.score,
                    categories: threatData.categories,
                    last_seen: threatData.lastSeen,
                };
                event.severity = "critical";
            }
        }
    }
}
```

### 2. Event Storage

```javascript
// Immutable event storage
class AuditEventStorage {
    constructor() {
        this.primaryStore = new ImmutableEventStore();
        this.archiveStore = new ArchiveEventStore();
        this.realtimeStore = new RealtimeEventStore();
    }

    async storeEvent(event) {
        // Generate integrity hash
        event.integrity = this.generateIntegrityHash(event);

        // Store in primary immutable store
        await this.primaryStore.append(event);

        // Store in realtime store for immediate queries
        await this.realtimeStore.index(event);

        // Archive if needed
        if (this.shouldArchive(event)) {
            await this.archiveStore.store(event);
        }
    }

    generateIntegrityHash(event) {
        const content = JSON.stringify({
            id: event.id,
            timestamp: event.timestamp,
            type: event.type,
            data: event.data,
        });

        return crypto.createHash("sha256").update(content).digest("hex");
    }

    async verifyIntegrity(event) {
        const calculatedHash = this.generateIntegrityHash(event);
        return calculatedHash === event.integrity;
    }
}

// Immutable event store
class ImmutableEventStore {
    async append(event) {
        // Write to append-only log
        const entry = {
            ...event,
            stored_at: new Date().toISOString(),
            storage_version: "1.0",
        };

        // Encrypt sensitive data
        if (this.containsSensitiveData(event)) {
            entry.data = await this.encryptData(entry.data);
            entry.encrypted = true;
        }

        // Write to multiple locations for redundancy
        await Promise.all([
            this.writeToLocal(entry),
            this.writeToRemote(entry),
            this.writeToBackup(entry),
        ]);
    }

    containsSensitiveData(event) {
        const sensitiveEvents = [
            "secret.read",
            "secret.created",
            "key.accessed",
            "auth.password_changed",
        ];

        return sensitiveEvents.includes(event.type);
    }
}
```

### 3. Event Retention

```javascript
// Retention policy implementation
class RetentionPolicy {
    constructor() {
        this.policies = {
            "auth.*": {
                retention_days: 365,
                archive_after_days: 90,
            },
            "secret.*": {
                retention_days: 2555, // 7 years
                archive_after_days: 365,
            },
            "key.*": {
                retention_days: 2555, // 7 years
                archive_after_days: 365,
            },
            "system.*": {
                retention_days: 730, // 2 years
                archive_after_days: 180,
            },
            "api.*": {
                retention_days: 90,
                archive_after_days: 30,
            },
        };
    }

    getRetentionPeriod(eventType) {
        for (const [pattern, policy] of Object.entries(this.policies)) {
            if (this.matchesPattern(eventType, pattern)) {
                return policy;
            }
        }

        // Default policy
        return {
            retention_days: 365,
            archive_after_days: 90,
        };
    }

    async enforceRetention() {
        const cutoffDates = this.calculateCutoffDates();

        for (const [pattern, dates] of Object.entries(cutoffDates)) {
            // Archive old events
            await this.archiveEvents(pattern, dates.archive_before);

            // Delete expired events
            await this.deleteEvents(pattern, dates.delete_before);
        }
    }

    calculateCutoffDates() {
        const dates = {};

        for (const [pattern, policy] of Object.entries(this.policies)) {
            dates[pattern] = {
                archive_before: new Date(
                    Date.now() -
                        policy.archive_after_days * 24 * 60 * 60 * 1000,
                ),
                delete_before: new Date(
                    Date.now() - policy.retention_days * 24 * 60 * 60 * 1000,
                ),
            };
        }

        return dates;
    }
}
```

## Analysis and Reporting

### 1. Real-Time Monitoring

```javascript
// Real-time event monitoring
class AuditMonitor {
    constructor() {
        this.rules = [];
        this.alerts = [];
        this.metrics = new MetricsCollector();
    }

    addRule(rule) {
        this.rules.push(rule);
    }

    async processEvent(event) {
        // Update metrics
        this.metrics.increment(`events.${event.type}`);

        // Check rules
        for (const rule of this.rules) {
            if (rule.matches(event)) {
                await this.handleRuleMatch(rule, event);
            }
        }
    }

    async handleRuleMatch(rule, event) {
        // Generate alert
        const alert = {
            id: uuidv4(),
            rule_id: rule.id,
            event_id: event.id,
            severity: rule.severity,
            title: rule.title,
            description: rule.describe(event),
            timestamp: new Date().toISOString(),
        };

        // Store alert
        await this.storeAlert(alert);

        // Notify
        await this.notify(alert);

        // Auto-response
        if (rule.autoResponse) {
            await rule.autoResponse(event);
        }
    }
}

// Example monitoring rules
const monitoringRules = [
    {
        id: "failed_login_attempts",
        title: "Multiple Failed Login Attempts",
        matches: (event) => {
            return (
                event.type === "auth.failed_attempts" &&
                event.data.attempt_count > 5
            );
        },
        severity: "high",
        describe: (event) => {
            return `${event.data.attempt_count} failed login attempts for ${event.data.identifier}`;
        },
        autoResponse: async (event) => {
            if (event.data.attempt_count > 10) {
                await lockAccount(event.data.identifier);
            }
        },
    },

    {
        id: "privilege_escalation",
        title: "Privilege Escalation Detected",
        matches: (event) => {
            return (
                event.type === "authz.permission_granted" &&
                ["admin", "owner"].includes(event.data.permission)
            );
        },
        severity: "critical",
        describe: (event) => {
            return `User ${event.data.user_id} granted ${event.data.permission} by ${event.data.granted_by}`;
        },
    },

    {
        id: "mass_secret_access",
        title: "Mass Secret Access",
        matches: (event) => {
            return (
                event.type === "secret.bulk_operation" &&
                event.data.operation === "export"
            );
        },
        severity: "critical",
        describe: (event) => {
            return `${event.data.secret_count} secrets exported by ${event.data.performed_by}`;
        },
    },
];
```

### 2. Compliance Reporting

```javascript
// Compliance report generator
class ComplianceReporter {
    async generateSOC2Report(startDate, endDate) {
        const report = {
            period: { start: startDate, end: endDate },
            controls: {},
        };

        // CC6.1: Logical and Physical Access Controls
        report.controls.CC61 = await this.analyzeAccessControls(
            startDate,
            endDate,
        );

        // CC6.7: Restriction of Access
        report.controls.CC67 = await this.analyzeAccessRestrictions(
            startDate,
            endDate,
        );

        // CC7.2: System Monitoring
        report.controls.CC72 = await this.analyzeSystemMonitoring(
            startDate,
            endDate,
        );

        return report;
    }

    async analyzeAccessControls(startDate, endDate) {
        const events = await this.queryEvents({
            types: ["auth.*", "authz.*"],
            date_range: { start: startDate, end: endDate },
        });

        return {
            total_login_attempts: events.filter((e) => e.type === "auth.login")
                .length,
            successful_logins: events.filter(
                (e) => e.type === "auth.login" && e.data.success,
            ).length,
            failed_logins: events.filter(
                (e) => e.type === "auth.login" && !e.data.success,
            ).length,
            mfa_usage_rate: this.calculateMFAUsage(events),
            permission_changes: events.filter((e) =>
                e.type.startsWith("authz."),
            ).length,
            evidence: events.slice(0, 100), // Sample events
        };
    }

    async generateGDPRReport(userId) {
        // Data subject access request
        const events = await this.queryEvents({
            user_id: userId,
            include_all: true,
        });

        return {
            user_id: userId,
            data_collected: this.categorizeUserData(events),
            processing_activities: this.summarizeProcessing(events),
            data_sharing: this.identifyDataSharing(events),
            retention_periods: this.getRetentionPeriods(events),
            export_format: "json",
            generated_at: new Date().toISOString(),
        };
    }
}
```

### 3. Forensic Analysis

```javascript
// Forensic analysis tools
class ForensicAnalyzer {
    async investigateIncident(incidentId, timeWindow) {
        const investigation = {
            incident_id: incidentId,
            timeline: [],
            actors: new Set(),
            affected_resources: new Set(),
            related_events: [],
        };

        // Get initial event
        const incident = await this.getEvent(incidentId);

        // Build timeline
        const timeline = await this.buildTimeline(
            incident,
            timeWindow.before,
            timeWindow.after,
        );

        // Identify actors
        for (const event of timeline) {
            if (event.data.user_id) {
                investigation.actors.add(event.data.user_id);
            }
            if (event.data.performed_by) {
                investigation.actors.add(event.data.performed_by);
            }
        }

        // Identify affected resources
        for (const event of timeline) {
            if (event.data.resource_id) {
                investigation.affected_resources.add({
                    type: event.data.resource_type,
                    id: event.data.resource_id,
                });
            }
        }

        // Correlation analysis
        investigation.correlations = await this.correlateEvents(timeline);

        // Generate report
        return this.generateForensicReport(investigation);
    }

    async buildTimeline(incident, beforeMinutes, afterMinutes) {
        const startTime = new Date(
            new Date(incident.timestamp).getTime() - beforeMinutes * 60 * 1000,
        );
        const endTime = new Date(
            new Date(incident.timestamp).getTime() + afterMinutes * 60 * 1000,
        );

        // Get all events in time window
        const events = await this.queryEvents({
            date_range: { start: startTime, end: endTime },
        });

        // Filter related events
        const related = events.filter((event) =>
            this.isRelated(event, incident),
        );

        // Sort by timestamp
        return related.sort(
            (a, b) => new Date(a.timestamp) - new Date(b.timestamp),
        );
    }

    isRelated(event, incident) {
        // Same user
        if (event.data.user_id === incident.data.user_id) return true;

        // Same resource
        if (event.data.resource_id === incident.data.resource_id) return true;

        // Same IP address
        if (event.context.ip_address === incident.context.ip_address)
            return true;

        // Same session
        if (event.context.session_id === incident.context.session_id)
            return true;

        return false;
    }
}
```

## Query Interface

### 1. Query Language

```javascript
// Audit query DSL
class AuditQueryBuilder {
    constructor() {
        this.query = {};
    }

    type(eventType) {
        this.query.type = eventType;
        return this;
    }

    types(eventTypes) {
        this.query.types = eventTypes;
        return this;
    }

    user(userId) {
        this.query.user_id = userId;
        return this;
    }

    dateRange(start, end) {
        this.query.date_range = { start, end };
        return this;
    }

    severity(minSeverity) {
        this.query.min_severity = minSeverity;
        return this;
    }

    resource(resourceType, resourceId) {
        this.query.resource = {
            type: resourceType,
            id: resourceId,
        };
        return this;
    }

    ipAddress(ip) {
        this.query.ip_address = ip;
        return this;
    }

    limit(count) {
        this.query.limit = count;
        return this;
    }

    build() {
        return this.query;
    }
}

// Usage examples
const query = new AuditQueryBuilder()
    .types(["secret.*", "key.*"])
    .user("user-123")
    .dateRange("2024-01-01", "2024-01-31")
    .severity("medium")
    .limit(1000)
    .build();
```

### 2. Search API

```javascript
// Audit search implementation
class AuditSearchService {
    async search(query) {
        // Validate query
        this.validateQuery(query);

        // Build search parameters
        const searchParams = this.buildSearchParams(query);

        // Execute search
        const results = await this.executeSearch(searchParams);

        // Post-process results
        return this.processResults(results, query);
    }

    async aggregateEvents(query, groupBy) {
        const results = await this.search(query);

        const aggregated = {};
        for (const event of results) {
            const key = this.getGroupKey(event, groupBy);
            if (!aggregated[key]) {
                aggregated[key] = {
                    count: 0,
                    events: [],
                };
            }
            aggregated[key].count++;
            aggregated[key].events.push(event);
        }

        return aggregated;
    }

    async exportEvents(query, format) {
        const results = await this.search(query);

        switch (format) {
            case "json":
                return JSON.stringify(results, null, 2);

            case "csv":
                return this.convertToCSV(results);

            case "siem":
                return this.convertToSIEM(results);

            default:
                throw new Error(`Unsupported format: ${format}`);
        }
    }
}
```

## Integration

### 1. SIEM Integration

```javascript
// SIEM connector
class SIEMConnector {
    constructor(config) {
        this.endpoint = config.endpoint;
        this.apiKey = config.apiKey;
        this.batchSize = config.batchSize || 100;
    }

    async streamEvents() {
        const stream = new EventStream();

        stream.on("event", async (event) => {
            // Convert to SIEM format
            const siemEvent = this.convertToSIEMFormat(event);

            // Buffer events
            this.buffer.push(siemEvent);

            // Send when buffer is full
            if (this.buffer.length >= this.batchSize) {
                await this.flush();
            }
        });

        // Periodic flush
        setInterval(() => this.flush(), 5000);
    }

    convertToSIEMFormat(event) {
        return {
            timestamp: event.timestamp,
            severity: this.mapSeverity(event.severity),
            category: this.mapCategory(event.type),
            source: "dotenv",
            event_id: event.id,
            user: event.data.user_id,
            action: event.type,
            result: event.data.success !== false ? "success" : "failure",
            details: event.data,
        };
    }

    async flush() {
        if (this.buffer.length === 0) return;

        const events = this.buffer.splice(0, this.batchSize);

        await fetch(this.endpoint, {
            method: "POST",
            headers: {
                Authorization: `Bearer ${this.apiKey}`,
                "Content-Type": "application/json",
            },
            body: JSON.stringify({ events }),
        });
    }
}
```

### 2. Webhook Notifications

```javascript
// Webhook notifier
class WebhookNotifier {
    constructor(config) {
        this.webhooks = config.webhooks || [];
        this.filters = config.filters || [];
    }

    async notifyEvent(event) {
        // Check filters
        if (!this.shouldNotify(event)) return;

        // Send to all configured webhooks
        await Promise.all(
            this.webhooks.map((webhook) => this.sendWebhook(webhook, event)),
        );
    }

    shouldNotify(event) {
        if (this.filters.length === 0) return true;

        return this.filters.some((filter) => {
            if (filter.type && !event.type.match(filter.type)) return false;
            if (filter.severity && event.severity < filter.severity)
                return false;
            if (filter.user_id && event.data.user_id !== filter.user_id)
                return false;

            return true;
        });
    }

    async sendWebhook(webhook, event) {
        const payload = {
            event: event,
            webhook_id: webhook.id,
            timestamp: new Date().toISOString(),
        };

        // Sign payload
        const signature = this.signPayload(payload, webhook.secret);

        try {
            await fetch(webhook.url, {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    "X-DotEnv-Signature": signature,
                },
                body: JSON.stringify(payload),
            });
        } catch (error) {
            console.error(`Webhook delivery failed: ${error}`);
        }
    }

    signPayload(payload, secret) {
        return crypto
            .createHmac("sha256", secret)
            .update(JSON.stringify(payload))
            .digest("hex");
    }
}
```

## Best Practices

### 1. Comprehensive Logging

```javascript
// ✅ Good: Log all security-relevant events
await auditLog("secret.accessed", {
    secret_id: secret.id,
    user_id: user.id,
    purpose: "deployment",
    environment: "production",
});

// ❌ Bad: Missing important context
await auditLog("secret_accessed", { id: secret.id });
```

### 2. Structured Data

```javascript
// ✅ Good: Structured, searchable data
const event = {
    type: "auth.login",
    data: {
        user_id: user.id,
        email: user.email,
        method: "password",
        mfa_used: true,
        ip_address: request.ip,
    },
};

// ❌ Bad: Unstructured string logging
const event = `User ${user.email} logged in from ${request.ip}`;
```

### 3. Immutability

```javascript
// ✅ Good: Immutable audit trail
class ImmutableAuditLog {
    async append(event) {
        // Write-once, read-many
        await this.storage.appendOnly(event);

        // Generate integrity hash
        event.hash = this.generateHash(event);

        // No updates or deletes allowed
    }
}

// ❌ Bad: Mutable audit logs
class MutableAuditLog {
    async update(eventId, changes) {
        // Never modify audit logs!
    }
}
```

## Resources

- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [OWASP Logging Guide](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [SOC 2 Compliance Guide](https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/soc2)
- [DotEnv Audit API](/documentation/v1/api/audit)

## Next Steps

- [Review compliance requirements](/documentation/v1/security-compliance/compliance)
- [Implement security best practices](/documentation/v1/security-compliance/security-best-practices)
- [Set up incident response](/documentation/v1/security-compliance/incident-response)
- [Configure SIEM integration](/documentation/v1/administration/integrations)
