---
title: Staging Environments
slug: staging-environments
order: 7
tags: [use-cases, staging, testing, deployment]
---

# Staging Environments

Learn how to set up and manage staging environments using DotEnv. This guide covers best practices for pre-production testing, data management, and maintaining parity with production while ensuring safe testing practices.

## Overview

Staging environment challenges:

- Production parity
- Data privacy and anonymization
- Performance testing
- Integration testing
- Access control
- Cost optimization

## Staging Architecture

### 1. Environment Hierarchy

```yaml
# .dotenv/staging-config.yaml
environments:
    staging:
        parent: production
        description: "Main staging environment"
        restrictions:
            - no_real_customer_data
            - limited_external_api_calls
        features:
            - performance_monitoring
            - debug_logging
            - test_payments

    staging-hotfix:
        parent: staging
        description: "Hotfix testing environment"
        ttl: 48h # Auto-cleanup after 48 hours

    staging-release:
        parent: staging
        description: "Release candidate testing"
        approval_required: true

    staging-load:
        parent: staging
        description: "Load testing environment"
        scaling:
            min_instances: 5
            max_instances: 20
```

### 2. Infrastructure Configuration

```terraform
# terraform/staging/main.tf
variable "environment" {
  default = "staging"
}

variable "dotenv_project" {
  default = "my-app"
}

# Staging-specific resources
resource "aws_instance" "staging_app" {
  instance_type = "t3.large"  # Smaller than production

  tags = {
    Environment = var.environment
    Project     = var.dotenv_project
    AutoShutdown = "true"  # Cost optimization
  }

  user_data = templatefile("${path.module}/user-data.sh", {
    dotenv_project = var.dotenv_project
    environment    = var.environment
  })
}

# Staging database (with anonymized data)
resource "aws_db_instance" "staging_db" {
  instance_class        = "db.t3.medium"
  allocated_storage     = 100
  backup_retention_period = 3  # Shorter than production

  # Restore from production snapshot with anonymization
  snapshot_identifier = data.aws_db_snapshot.latest_prod_anonymized.id

  tags = {
    Environment = var.environment
    DataType    = "anonymized"
  }
}
```

## Data Management

### 1. Data Anonymization Pipeline

```javascript
// scripts/anonymize-staging-data.js
const { Client } = require("pg");
const faker = require("faker");
const crypto = require("crypto");

class StagingDataAnonymizer {
    constructor(databaseUrl) {
        this.client = new Client({
            connectionString: databaseUrl,
        });
        this.salt = process.env.ANONYMIZATION_SALT;
    }

    async anonymize() {
        await this.client.connect();

        try {
            await this.client.query("BEGIN");

            // Anonymize user data
            await this.anonymizeUsers();

            // Anonymize sensitive business data
            await this.anonymizeOrders();
            await this.anonymizePayments();

            // Clear audit logs
            await this.clearAuditLogs();

            await this.client.query("COMMIT");
            console.log("✅ Data anonymization complete");
        } catch (error) {
            await this.client.query("ROLLBACK");
            console.error("❌ Anonymization failed:", error);
            throw error;
        } finally {
            await this.client.end();
        }
    }

    async anonymizeUsers() {
        console.log("Anonymizing user data...");

        const users = await this.client.query(
            "SELECT id, email, name FROM users",
        );

        for (const user of users.rows) {
            const hashedId = this.hashId(user.id);

            await this.client.query(
                `
        UPDATE users 
        SET 
          email = $1,
          name = $2,
          phone = $3,
          address = $4,
          date_of_birth = $5
        WHERE id = $6
      `,
                [
                    `user+${hashedId}@staging.example.com`,
                    faker.name.findName(),
                    faker.phone.phoneNumber(),
                    faker.address.streetAddress(),
                    faker.date.past(50, new Date(2000, 0, 1)),
                    user.id,
                ],
            );
        }

        console.log(`Anonymized ${users.rows.length} users`);
    }

    async anonymizeOrders() {
        console.log("Anonymizing order data...");

        // Keep order structure but anonymize details
        await this.client.query(`
      UPDATE orders 
      SET 
        shipping_address = 'Staging Address ' || id,
        billing_address = 'Staging Billing ' || id,
        customer_notes = 'Anonymized',
        internal_notes = 'Staging Data'
      WHERE 1=1
    `);

        // Anonymize order items
        await this.client.query(`
      UPDATE order_items
      SET 
        custom_text = CASE 
          WHEN custom_text IS NOT NULL 
          THEN 'Custom Text ' || id 
          ELSE NULL 
        END
    `);
    }

    async anonymizePayments() {
        console.log("Anonymizing payment data...");

        // Replace all payment tokens with test tokens
        await this.client.query(`
      UPDATE payments
      SET
        stripe_customer_id = 'cus_test_' || id,
        stripe_payment_method = 'pm_test_' || id,
        last_four = '4242',
        card_brand = 'Visa',
        card_exp_month = 12,
        card_exp_year = EXTRACT(YEAR FROM CURRENT_DATE) + 2
      WHERE payment_method = 'card'
    `);
    }

    async clearAuditLogs() {
        console.log("Clearing sensitive audit logs...");

        // Keep structure but clear sensitive data
        await this.client.query(`
      UPDATE audit_logs
      SET
        user_ip = '10.0.0.' || (id % 254 + 1),
        user_agent = 'Mozilla/5.0 (Staging)',
        request_body = '{"anonymized": true}',
        response_body = '{"status": "success"}'
      WHERE created_at < CURRENT_DATE - INTERVAL '7 days'
    `);

        // Delete very old logs
        await this.client.query(`
      DELETE FROM audit_logs
      WHERE created_at < CURRENT_DATE - INTERVAL '30 days'
    `);
    }

    hashId(id) {
        return crypto
            .createHash("sha256")
            .update(`${id}${this.salt}`)
            .digest("hex")
            .substring(0, 8);
    }
}

// Run anonymization
if (require.main === module) {
    const anonymizer = new StagingDataAnonymizer(
        process.env.STAGING_DATABASE_URL,
    );

    anonymizer.anonymize().catch(console.error);
}

module.exports = StagingDataAnonymizer;
```

### 2. Test Data Generation

```javascript
// scripts/generate-test-data.js
const { Client } = require("pg");
const faker = require("faker");

class TestDataGenerator {
    constructor(databaseUrl) {
        this.client = new Client({
            connectionString: databaseUrl,
        });
    }

    async generate(options = {}) {
        const { users = 1000, orders = 5000, products = 100 } = options;

        await this.client.connect();

        try {
            console.log("Generating test data for staging...");

            // Generate in batches for performance
            await this.generateUsers(users);
            await this.generateProducts(products);
            await this.generateOrders(orders);

            // Generate related data
            await this.generateReviews();
            await this.generateAnalytics();

            console.log("✅ Test data generation complete");
        } finally {
            await this.client.end();
        }
    }

    async generateUsers(count) {
        console.log(`Generating ${count} test users...`);

        const users = [];
        for (let i = 0; i < count; i++) {
            users.push({
                email: `test.user.${i}@staging.example.com`,
                name: faker.name.findName(),
                role: faker.random.arrayElement([
                    "customer",
                    "customer",
                    "admin",
                ]),
                status: faker.random.arrayElement([
                    "active",
                    "active",
                    "inactive",
                ]),
                created_at: faker.date.past(2),
            });
        }

        // Bulk insert
        const values = users
            .map(
                (u) =>
                    `('${u.email}', '${u.name}', '${u.role}', '${u.status}', '${u.created_at.toISOString()}')`,
            )
            .join(",");

        await this.client.query(`
      INSERT INTO users (email, name, role, status, created_at)
      VALUES ${values}
      ON CONFLICT (email) DO NOTHING
    `);
    }

    async generateProducts(count) {
        console.log(`Generating ${count} test products...`);

        const categories = [
            "Electronics",
            "Clothing",
            "Books",
            "Home",
            "Sports",
        ];

        for (let i = 0; i < count; i++) {
            await this.client.query(
                `
        INSERT INTO products (
          name, description, price, category, 
          sku, stock_quantity, status
        ) VALUES ($1, $2, $3, $4, $5, $6, $7)
      `,
                [
                    faker.commerce.productName(),
                    faker.lorem.paragraph(),
                    faker.commerce.price(10, 1000),
                    faker.random.arrayElement(categories),
                    `SKU-${Date.now()}-${i}`,
                    faker.datatype.number({ min: 0, max: 1000 }),
                    faker.random.arrayElement(["active", "active", "draft"]),
                ],
            );
        }
    }

    async generateOrders(count) {
        console.log(`Generating ${count} test orders...`);

        // Get user and product IDs
        const users = await this.client.query(
            "SELECT id FROM users ORDER BY RANDOM() LIMIT 1000",
        );
        const products = await this.client.query(
            "SELECT id, price FROM products WHERE status = 'active'",
        );

        for (let i = 0; i < count; i++) {
            const user = faker.random.arrayElement(users.rows);
            const orderDate = faker.date.past(1);
            const status = this.getOrderStatus(orderDate);

            // Create order
            const order = await this.client.query(
                `
        INSERT INTO orders (
          user_id, status, total_amount, 
          created_at, updated_at
        ) VALUES ($1, $2, $3, $4, $5)
        RETURNING id
      `,
                [
                    user.id,
                    status,
                    0, // Will update after adding items
                    orderDate,
                    orderDate,
                ],
            );

            // Add order items
            const itemCount = faker.datatype.number({ min: 1, max: 5 });
            let totalAmount = 0;

            for (let j = 0; j < itemCount; j++) {
                const product = faker.random.arrayElement(products.rows);
                const quantity = faker.datatype.number({ min: 1, max: 3 });
                const itemTotal = product.price * quantity;
                totalAmount += itemTotal;

                await this.client.query(
                    `
          INSERT INTO order_items (
            order_id, product_id, quantity, 
            unit_price, total_price
          ) VALUES ($1, $2, $3, $4, $5)
        `,
                    [
                        order.rows[0].id,
                        product.id,
                        quantity,
                        product.price,
                        itemTotal,
                    ],
                );
            }

            // Update order total
            await this.client.query(
                "UPDATE orders SET total_amount = $1 WHERE id = $2",
                [totalAmount, order.rows[0].id],
            );
        }
    }

    getOrderStatus(orderDate) {
        const daysSince =
            (Date.now() - orderDate.getTime()) / (1000 * 60 * 60 * 24);

        if (daysSince < 1) return "pending";
        if (daysSince < 3) return "processing";
        if (daysSince < 7)
            return faker.random.arrayElement(["shipped", "delivered"]);
        return "delivered";
    }

    async generateReviews() {
        console.log("Generating product reviews...");

        const orders = await this.client.query(`
      SELECT DISTINCT oi.product_id, o.user_id
      FROM order_items oi
      JOIN orders o ON oi.order_id = o.id
      WHERE o.status = 'delivered'
      ORDER BY RANDOM()
      LIMIT 500
    `);

        for (const { product_id, user_id } of orders.rows) {
            if (Math.random() > 0.7) continue; // 70% review rate

            await this.client.query(
                `
        INSERT INTO reviews (
          product_id, user_id, rating, 
          title, comment, verified_purchase
        ) VALUES ($1, $2, $3, $4, $5, true)
        ON CONFLICT DO NOTHING
      `,
                [
                    product_id,
                    user_id,
                    faker.datatype.number({ min: 3, max: 5 }),
                    faker.lorem.sentence(),
                    faker.lorem.paragraph(),
                ],
            );
        }
    }

    async generateAnalytics() {
        console.log("Generating analytics data...");

        // Page views
        const pages = ["/home", "/products", "/about", "/contact"];
        const startDate = new Date();
        startDate.setDate(startDate.getDate() - 30);

        for (let d = 0; d < 30; d++) {
            const date = new Date(startDate);
            date.setDate(date.getDate() + d);

            const dailyViews = faker.datatype.number({ min: 1000, max: 5000 });

            for (let v = 0; v < dailyViews; v++) {
                await this.client.query(
                    `
          INSERT INTO analytics_events (
            event_type, page_url, user_id,
            session_id, timestamp
          ) VALUES ($1, $2, $3, $4, $5)
        `,
                    [
                        "page_view",
                        faker.random.arrayElement(pages),
                        Math.random() > 0.3
                            ? faker.datatype.number({ min: 1, max: 1000 })
                            : null,
                        faker.datatype.uuid(),
                        new Date(date.getTime() + Math.random() * 86400000),
                    ],
                );
            }
        }
    }
}

// CLI execution
if (require.main === module) {
    const generator = new TestDataGenerator(process.env.STAGING_DATABASE_URL);

    generator
        .generate({
            users: 1000,
            orders: 5000,
            products: 100,
        })
        .catch(console.error);
}

module.exports = TestDataGenerator;
```

## Access Control

### 1. Staging Access Management

```javascript
// services/staging-access.js
const jwt = require("jsonwebtoken");
const speakeasy = require("speakeasy");

class StagingAccessControl {
    constructor() {
        this.accessPolicies = {
            developer: {
                environments: ["staging", "staging-hotfix"],
                permissions: ["read", "write", "deploy"],
                requiresMFA: false,
                ipWhitelist: null,
            },
            qa: {
                environments: ["staging", "staging-release"],
                permissions: ["read", "test"],
                requiresMFA: false,
                ipWhitelist: null,
            },
            external: {
                environments: ["staging"],
                permissions: ["read"],
                requiresMFA: true,
                ipWhitelist: ["client-ip-range"],
                timeRestriction: {
                    start: "09:00",
                    end: "18:00",
                    timezone: "America/New_York",
                },
            },
            automated: {
                environments: ["staging", "staging-load"],
                permissions: ["read", "test"],
                requiresMFA: false,
                tokenExpiry: 3600, // 1 hour
            },
        };
    }

    async generateAccessToken(user, environment) {
        const policy = this.accessPolicies[user.role];

        if (!policy) {
            throw new Error(`Unknown role: ${user.role}`);
        }

        if (!policy.environments.includes(environment)) {
            throw new Error(
                `Access denied to ${environment} for role ${user.role}`,
            );
        }

        // Check MFA if required
        if (policy.requiresMFA) {
            await this.verifyMFA(user);
        }

        // Check IP whitelist
        if (policy.ipWhitelist) {
            this.verifyIP(user.ip, policy.ipWhitelist);
        }

        // Check time restrictions
        if (policy.timeRestriction) {
            this.verifyTimeAccess(policy.timeRestriction);
        }

        // Generate token
        const token = jwt.sign(
            {
                userId: user.id,
                email: user.email,
                role: user.role,
                environment,
                permissions: policy.permissions,
                issuedAt: Date.now(),
            },
            process.env.STAGING_JWT_SECRET,
            {
                expiresIn: policy.tokenExpiry || "8h",
            },
        );

        // Log access
        await this.logAccess({
            user: user.email,
            environment,
            action: "token_generated",
            ip: user.ip,
        });

        return token;
    }

    async verifyMFA(user) {
        const token = user.mfaToken;
        if (!token) {
            throw new Error("MFA token required");
        }

        const verified = speakeasy.totp.verify({
            secret: user.mfaSecret,
            encoding: "base32",
            token,
            window: 2,
        });

        if (!verified) {
            throw new Error("Invalid MFA token");
        }
    }

    verifyIP(userIP, whitelist) {
        const ipRange = require("ip-range-check");

        if (!whitelist.some((range) => ipRange(userIP, range))) {
            throw new Error(`IP ${userIP} not whitelisted`);
        }
    }

    verifyTimeAccess(restriction) {
        const moment = require("moment-timezone");
        const now = moment().tz(restriction.timezone);
        const start = moment.tz(
            restriction.start,
            "HH:mm",
            restriction.timezone,
        );
        const end = moment.tz(restriction.end, "HH:mm", restriction.timezone);

        if (!now.isBetween(start, end)) {
            throw new Error(
                `Access only allowed between ${restriction.start} and ${restriction.end} ${restriction.timezone}`,
            );
        }
    }

    async logAccess(entry) {
        // Implementation depends on your logging system
        console.log("Staging Access:", entry);
    }
}

module.exports = StagingAccessControl;
```

### 2. Temporary Access Grants

```javascript
// scripts/grant-staging-access.js
const { DotEnvClient } = require("@dotenv/sdk");
const QRCode = require("qrcode");

class StagingAccessGrant {
    constructor() {
        this.client = new DotEnvClient({
            apiKey: process.env.DOTENV_API_KEY,
        });
    }

    async grantTemporaryAccess(options) {
        const {
            email,
            duration = "24h",
            environment = "staging",
            reason,
            restrictions = {},
        } = options;

        console.log(`Granting staging access to ${email}...`);

        // Create temporary credentials
        const credentials = {
            username: `temp_${email.split("@")[0]}_${Date.now()}`,
            password: this.generateSecurePassword(),
            apiKey: this.generateAPIKey(),
            validUntil: this.calculateExpiry(duration),
            environment,
            restrictions,
        };

        // Store in DotEnv
        await this.client.createSecret({
            project: "staging-access",
            environment: "temp",
            key: credentials.username,
            value: JSON.stringify(credentials),
            metadata: {
                email,
                reason,
                grantedBy: process.env.USER,
                grantedAt: new Date().toISOString(),
            },
        });

        // Generate access instructions
        const accessInfo = await this.generateAccessInfo(credentials);

        // Send to user
        await this.sendAccessEmail(email, accessInfo);

        // Schedule cleanup
        this.scheduleCleanup(credentials.username, duration);

        console.log("✅ Access granted successfully");

        return credentials;
    }

    generateSecurePassword() {
        const crypto = require("crypto");
        return crypto.randomBytes(16).toString("base64url");
    }

    generateAPIKey() {
        const crypto = require("crypto");
        return `stg_${crypto.randomBytes(24).toString("hex")}`;
    }

    calculateExpiry(duration) {
        const ms = require("ms");
        return new Date(Date.now() + ms(duration));
    }

    async generateAccessInfo(credentials) {
        const accessUrl = new URL(process.env.STAGING_URL);
        accessUrl.username = credentials.username;
        accessUrl.password = credentials.password;

        const info = {
            url: process.env.STAGING_URL,
            credentials: {
                username: credentials.username,
                password: credentials.password,
                apiKey: credentials.apiKey,
            },
            validUntil: credentials.validUntil,
            quickAccess: accessUrl.toString(),
        };

        // Generate QR code for mobile access
        const qrCode = await QRCode.toDataURL(
            JSON.stringify({
                url: process.env.STAGING_URL,
                token: credentials.apiKey,
            }),
        );

        info.qrCode = qrCode;

        return info;
    }

    async sendAccessEmail(email, accessInfo) {
        // Email implementation
        console.log(`Sending access details to ${email}`);

        const emailContent = `
      Staging Environment Access Granted
      
      URL: ${accessInfo.url}
      Username: ${accessInfo.credentials.username}
      Password: ${accessInfo.credentials.password}
      API Key: ${accessInfo.credentials.apiKey}
      
      Valid Until: ${accessInfo.validUntil}
      
      Quick Access: ${accessInfo.quickAccess}
      
      Security Notes:
      - This access is temporary and will expire automatically
      - Do not share these credentials
      - All actions are logged for security
    `;

        // Send email (implementation depends on your email service)
    }

    scheduleCleanup(username, duration) {
        const ms = require("ms");

        setTimeout(async () => {
            try {
                await this.client.deleteSecret({
                    project: "staging-access",
                    environment: "temp",
                    key: username,
                });

                console.log(`Cleaned up access for ${username}`);
            } catch (error) {
                console.error(`Failed to cleanup ${username}:`, error);
            }
        }, ms(duration));
    }
}

// CLI usage
if (require.main === module) {
    const grant = new StagingAccessGrant();

    const [email, duration, reason] = process.argv.slice(2);

    if (!email || !reason) {
        console.log(
            "Usage: node grant-staging-access.js <email> [duration] <reason>",
        );
        process.exit(1);
    }

    grant
        .grantTemporaryAccess({
            email,
            duration: duration || "24h",
            reason,
        })
        .catch(console.error);
}

module.exports = StagingAccessGrant;
```

## Testing Workflows

### 1. Automated Testing Pipeline

```javascript
// test/staging/e2e-suite.js
const { chromium } = require("playwright");
const { expect } = require("@playwright/test");

class StagingE2ETests {
    constructor(stagingUrl) {
        this.baseUrl = stagingUrl;
        this.testResults = [];
    }

    async runFullSuite() {
        const browser = await chromium.launch();
        const context = await browser.newContext();

        try {
            // Run test suites
            await this.testAuthentication(context);
            await this.testCriticalPaths(context);
            await this.testPaymentFlow(context);
            await this.testAPIEndpoints(context);
            await this.testPerformance(context);

            // Generate report
            await this.generateReport();
        } finally {
            await browser.close();
        }
    }

    async testAuthentication(context) {
        console.log("Testing authentication flows...");

        const page = await context.newPage();

        // Test login
        await page.goto(`${this.baseUrl}/login`);
        await page.fill("#email", "test.user@staging.example.com");
        await page.fill("#password", "TestPassword123!");
        await page.click('button[type="submit"]');

        await expect(page).toHaveURL(`${this.baseUrl}/dashboard`);

        // Test session persistence
        await page.reload();
        await expect(page).toHaveURL(`${this.baseUrl}/dashboard`);

        // Test logout
        await page.click("#logout-button");
        await expect(page).toHaveURL(`${this.baseUrl}/login`);

        this.recordTest("Authentication", "passed");
    }

    async testCriticalPaths(context) {
        console.log("Testing critical user paths...");

        const page = await context.newPage();

        // Test product browsing
        await page.goto(`${this.baseUrl}/products`);
        await expect(page.locator(".product-card")).toHaveCount(20);

        // Test product details
        await page.click(".product-card:first-child");
        await expect(page.locator("h1.product-title")).toBeVisible();

        // Test add to cart
        await page.click("#add-to-cart");
        await expect(page.locator(".cart-count")).toHaveText("1");

        // Test checkout flow
        await page.goto(`${this.baseUrl}/checkout`);
        await expect(page.locator(".order-summary")).toBeVisible();

        this.recordTest("Critical Paths", "passed");
    }

    async testPaymentFlow(context) {
        console.log("Testing payment integration...");

        const page = await context.newPage();

        // Use Stripe test mode
        await page.goto(`${this.baseUrl}/checkout`);

        // Fill test card details
        await page
            .frameLocator("#stripe-iframe")
            .locator("#cardNumber")
            .fill("4242424242424242");
        await page
            .frameLocator("#stripe-iframe")
            .locator("#cardExpiry")
            .fill("12/25");
        await page
            .frameLocator("#stripe-iframe")
            .locator("#cardCvc")
            .fill("123");

        await page.click("#submit-payment");

        // Verify success
        await expect(page).toHaveURL(/\/order-confirmation/);
        await expect(page.locator(".order-success")).toBeVisible();

        this.recordTest("Payment Flow", "passed");
    }

    async testAPIEndpoints(context) {
        console.log("Testing API endpoints...");

        const request = context.request;

        // Test public endpoints
        const products = await request.get(`${this.baseUrl}/api/products`);
        expect(products.status()).toBe(200);

        // Test authenticated endpoints
        const loginResponse = await request.post(
            `${this.baseUrl}/api/auth/login`,
            {
                data: {
                    email: "test.user@staging.example.com",
                    password: "TestPassword123!",
                },
            },
        );

        const { token } = await loginResponse.json();

        const profile = await request.get(`${this.baseUrl}/api/user/profile`, {
            headers: {
                Authorization: `Bearer ${token}`,
            },
        });

        expect(profile.status()).toBe(200);

        this.recordTest("API Endpoints", "passed");
    }

    async testPerformance(context) {
        console.log("Testing performance metrics...");

        const page = await context.newPage();

        // Enable performance tracking
        await page.coverage.startJSCoverage();
        await page.coverage.startCSSCoverage();

        const metrics = [];

        // Test key pages
        const pages = ["/", "/products", "/about", "/contact"];

        for (const path of pages) {
            const startTime = Date.now();
            await page.goto(`${this.baseUrl}${path}`);
            const loadTime = Date.now() - startTime;

            const performanceTiming = await page.evaluate(() =>
                JSON.stringify(window.performance.timing),
            );

            metrics.push({
                path,
                loadTime,
                timing: JSON.parse(performanceTiming),
            });
        }

        // Check performance thresholds
        const slowPages = metrics.filter((m) => m.loadTime > 3000);

        if (slowPages.length > 0) {
            this.recordTest("Performance", "warning", {
                message: `${slowPages.length} pages exceeded 3s load time`,
                details: slowPages,
            });
        } else {
            this.recordTest("Performance", "passed");
        }
    }

    recordTest(name, status, details = null) {
        this.testResults.push({
            name,
            status,
            timestamp: new Date().toISOString(),
            details,
        });
    }

    async generateReport() {
        const report = {
            environment: "staging",
            runDate: new Date().toISOString(),
            baseUrl: this.baseUrl,
            results: this.testResults,
            summary: {
                total: this.testResults.length,
                passed: this.testResults.filter((r) => r.status === "passed")
                    .length,
                failed: this.testResults.filter((r) => r.status === "failed")
                    .length,
                warnings: this.testResults.filter((r) => r.status === "warning")
                    .length,
            },
        };

        console.log("\n📊 Test Results:");
        console.log(JSON.stringify(report, null, 2));

        // Save report
        require("fs").writeFileSync(
            `staging-test-report-${Date.now()}.json`,
            JSON.stringify(report, null, 2),
        );
    }
}

// Run tests
if (require.main === module) {
    const tests = new StagingE2ETests(process.env.STAGING_URL);
    tests.runFullSuite().catch(console.error);
}

module.exports = StagingE2ETests;
```

### 2. Load Testing

```javascript
// test/staging/load-test.js
const autocannon = require("autocannon");

class StagingLoadTest {
    constructor(stagingUrl) {
        this.baseUrl = stagingUrl;
    }

    async runLoadTest(scenario = "default") {
        const scenarios = {
            default: {
                connections: 100,
                duration: 300, // 5 minutes
                pipelining: 10,
            },
            stress: {
                connections: 1000,
                duration: 600, // 10 minutes
                pipelining: 20,
            },
            spike: {
                connections: 500,
                duration: 60, // 1 minute
                pipelining: 50,
            },
        };

        const config = scenarios[scenario];

        console.log(`Running ${scenario} load test...`);

        const instance = autocannon({
            url: this.baseUrl,
            ...config,
            headers: {
                "X-Staging-Test": "true",
            },
            requests: [
                {
                    method: "GET",
                    path: "/",
                },
                {
                    method: "GET",
                    path: "/products",
                },
                {
                    method: "GET",
                    path: "/api/products",
                },
                {
                    method: "POST",
                    path: "/api/auth/login",
                    headers: {
                        "Content-Type": "application/json",
                    },
                    body: JSON.stringify({
                        email: "load.test@staging.example.com",
                        password: "LoadTest123!",
                    }),
                },
            ],
        });

        autocannon.track(instance, {
            renderProgressBar: true,
            renderResultsTable: true,
        });

        instance.on("done", (results) => {
            this.analyzeResults(results);
        });
    }

    analyzeResults(results) {
        const thresholds = {
            latency: {
                p99: 1000, // 1 second
                p95: 500, // 500ms
                p50: 200, // 200ms
            },
            throughput: {
                min: 1000, // 1000 req/sec
            },
            errors: {
                maxRate: 0.01, // 1% error rate
            },
        };

        const issues = [];

        // Check latency
        if (results.latency.p99 > thresholds.latency.p99) {
            issues.push(
                `P99 latency ${results.latency.p99}ms exceeds threshold`,
            );
        }

        // Check throughput
        if (results.throughput.mean < thresholds.throughput.min) {
            issues.push(
                `Throughput ${results.throughput.mean} req/sec below minimum`,
            );
        }

        // Check errors
        const errorRate = results.errors / results.requests.total;
        if (errorRate > thresholds.errors.maxRate) {
            issues.push(
                `Error rate ${(errorRate * 100).toFixed(2)}% exceeds threshold`,
            );
        }

        if (issues.length > 0) {
            console.log("\n⚠️  Performance Issues:");
            issues.forEach((issue) => console.log(`  - ${issue}`));
        } else {
            console.log("\n✅ All performance thresholds met");
        }

        // Save detailed results
        this.saveResults(results);
    }

    saveResults(results) {
        const report = {
            timestamp: new Date().toISOString(),
            environment: "staging",
            results: {
                duration: results.duration,
                requests: results.requests,
                latency: results.latency,
                throughput: results.throughput,
                errors: results.errors,
                timeouts: results.timeouts,
            },
        };

        require("fs").writeFileSync(
            `load-test-results-${Date.now()}.json`,
            JSON.stringify(report, null, 2),
        );
    }
}

// CLI execution
if (require.main === module) {
    const scenario = process.argv[2] || "default";
    const test = new StagingLoadTest(process.env.STAGING_URL);
    test.runLoadTest(scenario);
}

module.exports = StagingLoadTest;
```

## Monitoring

### 1. Staging Health Checks

```javascript
// monitoring/staging-health.js
const axios = require("axios");

class StagingHealthMonitor {
    constructor() {
        this.checks = [
            {
                name: "API Health",
                url: "/health",
                expectedStatus: 200,
                timeout: 5000,
            },
            {
                name: "Database Connection",
                url: "/health/db",
                expectedStatus: 200,
                timeout: 10000,
            },
            {
                name: "Redis Connection",
                url: "/health/redis",
                expectedStatus: 200,
                timeout: 5000,
            },
            {
                name: "External APIs",
                url: "/health/external",
                expectedStatus: 200,
                timeout: 15000,
            },
            {
                name: "Static Assets",
                url: "/static/app.js",
                expectedStatus: 200,
                timeout: 5000,
            },
        ];
    }

    async runHealthChecks() {
        const results = await Promise.all(
            this.checks.map((check) => this.performCheck(check)),
        );

        const healthy = results.every((r) => r.healthy);

        return {
            healthy,
            timestamp: new Date().toISOString(),
            checks: results,
        };
    }

    async performCheck(check) {
        const startTime = Date.now();

        try {
            const response = await axios.get(
                `${process.env.STAGING_URL}${check.url}`,
                {
                    timeout: check.timeout,
                    validateStatus: () => true,
                },
            );

            const duration = Date.now() - startTime;
            const healthy = response.status === check.expectedStatus;

            return {
                name: check.name,
                healthy,
                status: response.status,
                expectedStatus: check.expectedStatus,
                duration,
                details: response.data,
            };
        } catch (error) {
            return {
                name: check.name,
                healthy: false,
                error: error.message,
                duration: Date.now() - startTime,
            };
        }
    }

    async continuousMonitoring(interval = 60000) {
        console.log("Starting continuous staging health monitoring...");

        setInterval(async () => {
            const results = await this.runHealthChecks();

            if (!results.healthy) {
                await this.alertOnFailure(results);
            }

            // Log metrics
            this.logMetrics(results);
        }, interval);
    }

    async alertOnFailure(results) {
        const failedChecks = results.checks.filter((c) => !c.healthy);

        console.error("⚠️  Staging health check failures:");
        failedChecks.forEach((check) => {
            console.error(
                `  - ${check.name}: ${check.error || `Status ${check.status}`}`,
            );
        });

        // Send alerts (implementation depends on your alerting system)
        // await this.sendSlackAlert(failedChecks);
        // await this.createPagerDutyIncident(failedChecks);
    }

    logMetrics(results) {
        // Send to monitoring system (Prometheus, DataDog, etc.)
        results.checks.forEach((check) => {
            console.log(
                `staging_health{check="${check.name}",status="${check.healthy ? "up" : "down"}"} ${check.healthy ? 1 : 0}`,
            );
            console.log(
                `staging_response_time{check="${check.name}"} ${check.duration}`,
            );
        });
    }
}

// Start monitoring
if (require.main === module) {
    const monitor = new StagingHealthMonitor();
    monitor.continuousMonitoring();
}

module.exports = StagingHealthMonitor;
```

## Best Practices

### 1. Staging Configuration

```yaml
# config/staging-best-practices.yaml
staging_configuration:
    infrastructure:
        - Use similar but smaller resources than production
        - Enable detailed logging and monitoring
        - Use separate databases with anonymized data
        - Implement auto-shutdown for cost savings

    security:
        - Restrict access with IP whitelisting
        - Use temporary credentials
        - Enable audit logging
        - Regular data anonymization

    testing:
        - Automated test suites on deployment
        - Load testing before production releases
        - Security scanning
        - Performance benchmarking

    data_management:
        - No real customer data
        - Regular data refresh from production
        - Test data generation scripts
        - Automated cleanup of old data
```

### 2. Release Process

```bash
#!/bin/bash
# scripts/staging-release.sh

version=$1

if [ -z "$version" ]; then
  echo "Usage: ./staging-release.sh <version>"
  exit 1
fi

echo "🚀 Deploying version $version to staging..."

# Pre-deployment checks
echo "Running pre-deployment checks..."

# Verify tests pass
npm test || exit 1

# Build application
npm run build || exit 1

# Deploy to staging
echo "Deploying to staging environment..."
dotenv run -- npm run deploy:staging

# Run post-deployment tests
echo "Running post-deployment tests..."
npm run test:staging

# Update staging documentation
cat > staging-release-notes.md << EOF
# Staging Release: $version
Date: $(date)
Deployed by: $USER

## Changes
$(git log --oneline -n 10)

## Test Results
- E2E Tests: Passed
- Load Tests: Pending
- Security Scan: Pending

## Access
URL: $STAGING_URL
API: $STAGING_API_URL
EOF

echo "✅ Staging deployment complete!"
```

## Troubleshooting

### Common Issues

1. **Data Inconsistencies**

    ```sql
    -- Check for production data leakage
    SELECT COUNT(*) FROM users
    WHERE email NOT LIKE '%@staging.example.com';
    ```

2. **Performance Degradation**

    ```bash
    # Check resource usage
    kubectl top pods -n staging
    kubectl top nodes
    ```

3. **Access Issues**
    ```javascript
    // Verify access tokens
    const jwt = require("jsonwebtoken");
    const token = process.env.STAGING_ACCESS_TOKEN;
    const decoded = jwt.decode(token);
    console.log("Token expires:", new Date(decoded.exp * 1000));
    ```

## Resources

- [Staging Environment Best Practices](https://12factor.net/dev-prod-parity)
- [Data Anonymization Techniques](https://www.privacytools.io/)
- [Load Testing with Autocannon](https://github.com/mcollina/autocannon)
- [DotEnv Environment Management](/documentation/v1/core-concepts/environments-concept)

## Next Steps

- [Set up staging environment](#staging-architecture)
- [Configure data anonymization](#data-management)
- [Implement access control](#access-control)
- [Create testing workflows](#testing-workflows)
