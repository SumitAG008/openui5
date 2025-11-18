# Azure API Management - Comprehensive Documentation

This guide provides comprehensive documentation for Azure API Management (APIM), covering API complexity levels, implementation patterns, testing strategies, and enterprise integration scenarios.

## Table of Contents

1. [Overview](#overview)
2. [API Complexity Levels](#api-complexity-levels)
   - [Low Complexity APIs](#low-complexity-apis)
   - [Medium Complexity APIs](#medium-complexity-apis)
   - [High Complexity APIs](#high-complexity-apis)
3. [API Creation and Implementation](#api-creation-and-implementation)
4. [Testing Strategies](#testing-strategies)
5. [API Policies](#api-policies)
6. [Vendor Access Configuration](#vendor-access-configuration)
7. [Data Lake Integration](#data-lake-integration)

---

## Overview

Azure API Management is a hybrid, multicloud management platform for APIs across all environments. It serves as a gateway to manage, secure, and analyze APIs while providing a developer portal for API consumers.

### Key Components

- **API Gateway**: Routes calls, enforces policies, collects telemetry
- **Azure Portal**: Administrative interface for API configuration
- **Developer Portal**: Auto-generated portal for API documentation
- **Management Plane**: REST API for programmatic access

---

## API Complexity Levels

### Low Complexity APIs

Low complexity APIs are straightforward, single-operation endpoints with minimal transformation requirements. They typically involve direct passthrough or simple response formatting.

#### Characteristics

- Single backend service
- Minimal or no data transformation
- Simple authentication (API key or basic)
- No complex orchestration
- Stateless operations

#### Example 1: Simple GET Endpoint

**Use Case**: Retrieve product information by ID

```xml
<!-- API Policy Configuration -->
<policies>
    <inbound>
        <base />
        <set-backend-service base-url="https://products-api.contoso.com" />
        <set-header name="X-Request-ID" exists-action="override">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <set-header name="X-Response-Time" exists-action="override">
            <value>@(context.Elapsed.TotalMilliseconds.ToString())</value>
        </set-header>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

**OpenAPI Specification**:

```yaml
openapi: 3.0.1
info:
  title: Product API
  version: '1.0'
paths:
  /products/{id}:
    get:
      summary: Get product by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Product details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'
components:
  schemas:
    Product:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        price:
          type: number
```

#### Example 2: Health Check Endpoint

**Use Case**: Simple service health verification

```xml
<policies>
    <inbound>
        <return-response>
            <set-status code="200" reason="OK" />
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>{"status": "healthy", "timestamp": "@(DateTime.UtcNow.ToString("o"))"}</set-body>
        </return-response>
    </inbound>
</policies>
```

#### Example 3: Static Configuration Endpoint

**Use Case**: Return application configuration

```xml
<policies>
    <inbound>
        <base />
        <cache-lookup vary-by-developer="false"
                      vary-by-developer-groups="false"
                      downstream-caching-type="none">
            <vary-by-header>Accept</vary-by-header>
        </cache-lookup>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <cache-store duration="3600" />
    </outbound>
</policies>
```

---

### Medium Complexity APIs

Medium complexity APIs involve multiple operations, data transformations, conditional logic, or integration with multiple backend services.

#### Characteristics

- Multiple backend services
- Data transformation and mapping
- Conditional routing
- Request/response validation
- Caching strategies
- Error handling and retry logic

#### Example 1: Data Aggregation API

**Use Case**: Aggregate customer data from multiple services

```xml
<policies>
    <inbound>
        <base />
        <set-variable name="customerId" value="@(context.Request.MatchedParameters["id"])" />

        <!-- Validate request -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
            <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud">
                    <value>api://customer-api</value>
                </claim>
            </required-claims>
        </validate-jwt>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />

        <!-- Call Profile Service -->
        <send-request mode="new" response-variable-name="profileResponse" timeout="20">
            <set-url>@($"https://profile-service.contoso.com/api/profiles/{context.Variables["customerId"]}")</set-url>
            <set-method>GET</set-method>
            <set-header name="Authorization" exists-action="override">
                <value>@(context.Request.Headers.GetValueOrDefault("Authorization", ""))</value>
            </set-header>
        </send-request>

        <!-- Call Orders Service -->
        <send-request mode="new" response-variable-name="ordersResponse" timeout="20">
            <set-url>@($"https://orders-service.contoso.com/api/customers/{context.Variables["customerId"]}/orders")</set-url>
            <set-method>GET</set-method>
            <set-header name="Authorization" exists-action="override">
                <value>@(context.Request.Headers.GetValueOrDefault("Authorization", ""))</value>
            </set-header>
        </send-request>

        <!-- Aggregate responses -->
        <set-body>@{
            var profile = ((IResponse)context.Variables["profileResponse"]).Body.As<JObject>();
            var orders = ((IResponse)context.Variables["ordersResponse"]).Body.As<JArray>();

            return new JObject(
                new JProperty("customer", profile),
                new JProperty("recentOrders", orders.Take(5)),
                new JProperty("totalOrders", orders.Count)
            ).ToString();
        }</set-body>
    </outbound>
    <on-error>
        <base />
        <set-body>@{
            return new JObject(
                new JProperty("error", context.LastError.Message),
                new JProperty("correlationId", context.RequestId)
            ).ToString();
        }</set-body>
    </on-error>
</policies>
```

#### Example 2: Conditional Routing API

**Use Case**: Route requests based on content or headers

```xml
<policies>
    <inbound>
        <base />

        <!-- Extract routing information -->
        <set-variable name="region" value="@(context.Request.Headers.GetValueOrDefault("X-Region", "default"))" />
        <set-variable name="apiVersion" value="@(context.Request.Headers.GetValueOrDefault("Api-Version", "v1"))" />

        <!-- Route based on region -->
        <choose>
            <when condition="@(context.Variables.GetValueOrDefault<string>("region") == "eu")">
                <set-backend-service base-url="https://api-eu.contoso.com" />
            </when>
            <when condition="@(context.Variables.GetValueOrDefault<string>("region") == "asia")">
                <set-backend-service base-url="https://api-asia.contoso.com" />
            </when>
            <otherwise>
                <set-backend-service base-url="https://api-us.contoso.com" />
            </otherwise>
        </choose>

        <!-- Version-based path rewriting -->
        <choose>
            <when condition="@(context.Variables.GetValueOrDefault<string>("apiVersion") == "v2")">
                <rewrite-uri template="/v2{context.Request.MatchedPath}" />
            </when>
        </choose>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
</policies>
```

#### Example 3: Request Transformation API

**Use Case**: Transform legacy SOAP to REST

```xml
<policies>
    <inbound>
        <base />

        <!-- Transform JSON request to SOAP -->
        <set-body>@{
            var json = context.Request.Body.As<JObject>();
            return $@"<?xml version=""1.0"" encoding=""utf-8""?>
            <soap:Envelope xmlns:soap=""http://schemas.xmlsoap.org/soap/envelope/"">
                <soap:Body>
                    <GetCustomer xmlns=""http://contoso.com/services"">
                        <CustomerId>{json["customerId"]}</CustomerId>
                        <IncludeDetails>{json["includeDetails"] ?? "false"}</IncludeDetails>
                    </GetCustomer>
                </soap:Body>
            </soap:Envelope>";
        }</set-body>

        <set-header name="Content-Type" exists-action="override">
            <value>text/xml</value>
        </set-header>
        <set-header name="SOAPAction" exists-action="override">
            <value>"http://contoso.com/services/GetCustomer"</value>
        </set-header>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />

        <!-- Transform SOAP response to JSON -->
        <set-body>@{
            var soap = context.Response.Body.As<XDocument>();
            var ns = XNamespace.Get("http://contoso.com/services");
            var result = soap.Descendants(ns + "GetCustomerResult").FirstOrDefault();

            return new JObject(
                new JProperty("customerId", result?.Element(ns + "Id")?.Value),
                new JProperty("name", result?.Element(ns + "Name")?.Value),
                new JProperty("email", result?.Element(ns + "Email")?.Value)
            ).ToString();
        }</set-body>

        <set-header name="Content-Type" exists-action="override">
            <value>application/json</value>
        </set-header>
    </outbound>
</policies>
```

---

### High Complexity APIs

High complexity APIs involve sophisticated orchestration, workflow management, advanced security patterns, and complex business logic implementation.

#### Characteristics

- Complex multi-step workflows
- Advanced security (mTLS, encryption, tokenization)
- Event-driven patterns
- Saga pattern implementations
- Complex caching strategies
- Circuit breaker patterns
- Advanced monitoring and diagnostics

#### Example 1: Multi-Step Order Processing API

**Use Case**: Complete order processing with inventory check, payment, and fulfillment

```xml
<policies>
    <inbound>
        <base />

        <!-- Validate and parse order -->
        <set-variable name="order" value="@(context.Request.Body.As<JObject>(preserveContent: true))" />
        <set-variable name="correlationId" value="@(Guid.NewGuid().ToString())" />

        <!-- JWT Validation with custom claims -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" output-token-variable-name="jwt">
            <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
            <required-claims>
                <claim name="roles" match="any">
                    <value>Order.Write</value>
                    <value>Order.Admin</value>
                </claim>
            </required-claims>
        </validate-jwt>

        <!-- Extract user context -->
        <set-variable name="userId" value="@(((Jwt)context.Variables["jwt"]).Claims.GetValueOrDefault("oid", "unknown"))" />

        <!-- Rate limiting per user -->
        <rate-limit-by-key calls="10" renewal-period="60"
                          counter-key="@(context.Variables.GetValueOrDefault<string>("userId"))" />
    </inbound>

    <backend>
        <!-- Step 1: Inventory Check -->
        <send-request mode="new" response-variable-name="inventoryResponse" timeout="30">
            <set-url>https://inventory-service.contoso.com/api/check</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-header name="X-Correlation-ID" exists-action="override">
                <value>@((string)context.Variables["correlationId"])</value>
            </set-header>
            <set-body>@{
                var order = (JObject)context.Variables["order"];
                return new JObject(
                    new JProperty("items", order["items"]),
                    new JProperty("warehouseRegion", order["shippingAddress"]?["region"] ?? "default")
                ).ToString();
            }</set-body>
        </send-request>

        <!-- Check inventory result -->
        <choose>
            <when condition="@(((IResponse)context.Variables["inventoryResponse"]).StatusCode != 200)">
                <return-response>
                    <set-status code="409" reason="Inventory Unavailable" />
                    <set-body>@{
                        return new JObject(
                            new JProperty("error", "Inventory check failed"),
                            new JProperty("correlationId", context.Variables["correlationId"])
                        ).ToString();
                    }</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Step 2: Payment Processing -->
        <send-request mode="new" response-variable-name="paymentResponse" timeout="60">
            <set-url>https://payment-service.contoso.com/api/process</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-header name="X-Correlation-ID" exists-action="override">
                <value>@((string)context.Variables["correlationId"])</value>
            </set-header>
            <set-header name="X-Idempotency-Key" exists-action="override">
                <value>@($"{context.Variables["userId"]}-{context.Variables["correlationId"]}")</value>
            </set-header>
            <set-body>@{
                var order = (JObject)context.Variables["order"];
                return new JObject(
                    new JProperty("amount", order["totalAmount"]),
                    new JProperty("currency", order["currency"] ?? "USD"),
                    new JProperty("paymentMethod", order["paymentMethod"]),
                    new JProperty("customerId", context.Variables["userId"])
                ).ToString();
            }</set-body>
        </send-request>

        <!-- Check payment result -->
        <choose>
            <when condition="@(((IResponse)context.Variables["paymentResponse"]).StatusCode != 200)">
                <!-- Compensate: Release inventory reservation -->
                <send-request mode="new" response-variable-name="releaseResponse" timeout="30">
                    <set-url>https://inventory-service.contoso.com/api/release</set-url>
                    <set-method>POST</set-method>
                    <set-header name="X-Correlation-ID" exists-action="override">
                        <value>@((string)context.Variables["correlationId"])</value>
                    </set-header>
                    <set-body>@{
                        var inventoryResult = ((IResponse)context.Variables["inventoryResponse"]).Body.As<JObject>();
                        return new JObject(
                            new JProperty("reservationId", inventoryResult["reservationId"])
                        ).ToString();
                    }</set-body>
                </send-request>

                <return-response>
                    <set-status code="402" reason="Payment Failed" />
                    <set-body>@{
                        return new JObject(
                            new JProperty("error", "Payment processing failed"),
                            new JProperty("correlationId", context.Variables["correlationId"])
                        ).ToString();
                    }</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Step 3: Create Order Record -->
        <send-request mode="new" response-variable-name="orderResponse" timeout="30">
            <set-url>https://order-service.contoso.com/api/orders</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-header name="X-Correlation-ID" exists-action="override">
                <value>@((string)context.Variables["correlationId"])</value>
            </set-header>
            <set-body>@{
                var order = (JObject)context.Variables["order"];
                var inventory = ((IResponse)context.Variables["inventoryResponse"]).Body.As<JObject>();
                var payment = ((IResponse)context.Variables["paymentResponse"]).Body.As<JObject>();

                return new JObject(
                    new JProperty("customerId", context.Variables["userId"]),
                    new JProperty("items", order["items"]),
                    new JProperty("shippingAddress", order["shippingAddress"]),
                    new JProperty("reservationId", inventory["reservationId"]),
                    new JProperty("paymentId", payment["transactionId"]),
                    new JProperty("correlationId", context.Variables["correlationId"])
                ).ToString();
            }</set-body>
        </send-request>

        <!-- Step 4: Trigger Fulfillment (async) -->
        <send-one-way-request mode="new">
            <set-url>https://fulfillment-service.contoso.com/api/fulfill</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-header name="X-Correlation-ID" exists-action="override">
                <value>@((string)context.Variables["correlationId"])</value>
            </set-header>
            <set-body>@{
                var orderResult = ((IResponse)context.Variables["orderResponse"]).Body.As<JObject>();
                return new JObject(
                    new JProperty("orderId", orderResult["orderId"]),
                    new JProperty("priority", "normal")
                ).ToString();
            }</set-body>
        </send-one-way-request>
    </backend>

    <outbound>
        <base />

        <!-- Construct final response -->
        <set-body>@{
            var orderResult = ((IResponse)context.Variables["orderResponse"]).Body.As<JObject>();
            var paymentResult = ((IResponse)context.Variables["paymentResponse"]).Body.As<JObject>();

            return new JObject(
                new JProperty("orderId", orderResult["orderId"]),
                new JProperty("status", "confirmed"),
                new JProperty("paymentConfirmation", paymentResult["confirmationNumber"]),
                new JProperty("estimatedDelivery", orderResult["estimatedDelivery"]),
                new JProperty("correlationId", context.Variables["correlationId"])
            ).ToString();
        }</set-body>

        <!-- Log to Event Hub for analytics -->
        <log-to-eventhub logger-id="order-analytics">@{
            var order = (JObject)context.Variables["order"];
            return new JObject(
                new JProperty("eventType", "OrderCreated"),
                new JProperty("orderId", ((IResponse)context.Variables["orderResponse"]).Body.As<JObject>()["orderId"]),
                new JProperty("customerId", context.Variables["userId"]),
                new JProperty("amount", order["totalAmount"]),
                new JProperty("timestamp", DateTime.UtcNow)
            ).ToString();
        }</log-to-eventhub>
    </outbound>

    <on-error>
        <base />

        <!-- Log error details -->
        <log-to-eventhub logger-id="error-analytics">@{
            return new JObject(
                new JProperty("eventType", "OrderError"),
                new JProperty("error", context.LastError.Message),
                new JProperty("source", context.LastError.Source),
                new JProperty("correlationId", context.Variables.GetValueOrDefault<string>("correlationId", "unknown")),
                new JProperty("timestamp", DateTime.UtcNow)
            ).ToString();
        }</log-to-eventhub>

        <return-response>
            <set-status code="500" reason="Internal Server Error" />
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>@{
                return new JObject(
                    new JProperty("error", "An error occurred processing your order"),
                    new JProperty("correlationId", context.Variables.GetValueOrDefault<string>("correlationId", "unknown")),
                    new JProperty("support", "Please contact support with the correlation ID")
                ).ToString();
            }</set-body>
        </return-response>
    </on-error>
</policies>
```

#### Example 2: Circuit Breaker Pattern

**Use Case**: Implement resilient API calls with circuit breaker

```xml
<policies>
    <inbound>
        <base />

        <!-- Check circuit breaker state -->
        <cache-lookup-value key="circuit-breaker-backend-service" variable-name="circuitState" />

        <choose>
            <when condition="@(context.Variables.GetValueOrDefault<string>("circuitState") == "open")">
                <!-- Check if enough time has passed to try again -->
                <cache-lookup-value key="circuit-breaker-backend-service-timestamp" variable-name="openTimestamp" />
                <choose>
                    <when condition="@((DateTime.UtcNow - DateTime.Parse((string)context.Variables["openTimestamp"])).TotalSeconds < 60)">
                        <return-response>
                            <set-status code="503" reason="Service Unavailable" />
                            <set-header name="Retry-After" exists-action="override">
                                <value>60</value>
                            </set-header>
                            <set-body>{"error": "Service temporarily unavailable", "retryAfter": 60}</set-body>
                        </return-response>
                    </when>
                    <otherwise>
                        <!-- Move to half-open state -->
                        <cache-store-value key="circuit-breaker-backend-service" value="half-open" duration="300" />
                    </otherwise>
                </choose>
            </when>
        </choose>
    </inbound>

    <backend>
        <retry condition="@(context.Response.StatusCode >= 500)" count="3" interval="1" first-fast-retry="true">
            <forward-request buffer-request-body="true" timeout="30" />
        </retry>
    </backend>

    <outbound>
        <base />

        <!-- Track success for circuit breaker -->
        <choose>
            <when condition="@(context.Response.StatusCode < 500)">
                <!-- Reset failure count on success -->
                <cache-store-value key="circuit-breaker-backend-service-failures" value="0" duration="300" />
                <cache-store-value key="circuit-breaker-backend-service" value="closed" duration="300" />
            </when>
        </choose>
    </outbound>

    <on-error>
        <base />

        <!-- Increment failure counter -->
        <cache-lookup-value key="circuit-breaker-backend-service-failures"
                           default-value="0"
                           variable-name="failures" />

        <set-variable name="newFailures" value="@(int.Parse((string)context.Variables["failures"]) + 1)" />

        <cache-store-value key="circuit-breaker-backend-service-failures"
                          value="@(context.Variables["newFailures"].ToString())"
                          duration="300" />

        <!-- Open circuit if threshold reached -->
        <choose>
            <when condition="@((int)context.Variables["newFailures"] >= 5)">
                <cache-store-value key="circuit-breaker-backend-service" value="open" duration="300" />
                <cache-store-value key="circuit-breaker-backend-service-timestamp"
                                  value="@(DateTime.UtcNow.ToString("o"))"
                                  duration="300" />
            </when>
        </choose>

        <return-response>
            <set-status code="502" reason="Bad Gateway" />
            <set-body>@{
                return new JObject(
                    new JProperty("error", "Backend service error"),
                    new JProperty("requestId", context.RequestId)
                ).ToString();
            }</set-body>
        </return-response>
    </on-error>
</policies>
```

#### Example 3: GraphQL Federation Gateway

**Use Case**: Federate multiple GraphQL services

```xml
<policies>
    <inbound>
        <base />

        <!-- Parse GraphQL query -->
        <set-variable name="graphqlQuery" value="@(context.Request.Body.As<JObject>(preserveContent: true))" />
        <set-variable name="operationType" value="@{
            var query = ((JObject)context.Variables["graphqlQuery"])["query"].ToString();
            if (query.TrimStart().StartsWith("mutation")) return "mutation";
            if (query.TrimStart().StartsWith("subscription")) return "subscription";
            return "query";
        }" />

        <!-- Route based on operation and fields -->
        <set-variable name="queryFields" value="@{
            var query = ((JObject)context.Variables["graphqlQuery"])["query"].ToString();
            var fields = new List<string>();

            if (query.Contains("user") || query.Contains("users")) fields.Add("users");
            if (query.Contains("product") || query.Contains("products")) fields.Add("products");
            if (query.Contains("order") || query.Contains("orders")) fields.Add("orders");

            return string.Join(",", fields);
        }" />
    </inbound>

    <backend>
        <!-- Parallel calls to federated services -->
        <choose>
            <when condition="@(context.Variables.GetValueOrDefault<string>("queryFields").Contains("users"))">
                <send-request mode="new" response-variable-name="usersResponse" timeout="30">
                    <set-url>https://users-graphql.contoso.com/graphql</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>@(((JObject)context.Variables["graphqlQuery"]).ToString())</set-body>
                </send-request>
            </when>
        </choose>

        <choose>
            <when condition="@(context.Variables.GetValueOrDefault<string>("queryFields").Contains("products"))">
                <send-request mode="new" response-variable-name="productsResponse" timeout="30">
                    <set-url>https://products-graphql.contoso.com/graphql</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>@(((JObject)context.Variables["graphqlQuery"]).ToString())</set-body>
                </send-request>
            </when>
        </choose>

        <choose>
            <when condition="@(context.Variables.GetValueOrDefault<string>("queryFields").Contains("orders"))">
                <send-request mode="new" response-variable-name="ordersResponse" timeout="30">
                    <set-url>https://orders-graphql.contoso.com/graphql</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>@(((JObject)context.Variables["graphqlQuery"]).ToString())</set-body>
                </send-request>
            </when>
        </choose>
    </backend>

    <outbound>
        <base />

        <!-- Merge GraphQL responses -->
        <set-body>@{
            var result = new JObject();
            var data = new JObject();
            var errors = new JArray();

            void MergeResponse(string varName) {
                if (context.Variables.ContainsKey(varName)) {
                    var response = ((IResponse)context.Variables[varName]).Body.As<JObject>();
                    if (response["data"] != null) {
                        foreach (var prop in ((JObject)response["data"]).Properties()) {
                            data[prop.Name] = prop.Value;
                        }
                    }
                    if (response["errors"] != null) {
                        foreach (var error in (JArray)response["errors"]) {
                            errors.Add(error);
                        }
                    }
                }
            }

            MergeResponse("usersResponse");
            MergeResponse("productsResponse");
            MergeResponse("ordersResponse");

            result["data"] = data;
            if (errors.Count > 0) {
                result["errors"] = errors;
            }

            return result.ToString();
        }</set-body>
    </outbound>
</policies>
```

---

## API Creation and Implementation

### Step 1: Planning and Design

Before creating an API, complete the following planning activities:

#### Define API Requirements

```markdown
## API Requirements Checklist

- [ ] Business requirements documented
- [ ] Target consumers identified
- [ ] Security requirements defined
- [ ] Performance SLAs established
- [ ] Data contracts specified
- [ ] Versioning strategy determined
- [ ] Error handling approach defined
```

#### Design API Contract

```yaml
# Example OpenAPI 3.0 Specification
openapi: 3.0.1
info:
  title: Customer API
  description: API for managing customer data
  version: '1.0'
  contact:
    name: API Team
    email: api-team@contoso.com

servers:
  - url: https://api.contoso.com/v1
    description: Production
  - url: https://api-staging.contoso.com/v1
    description: Staging

security:
  - bearerAuth: []

paths:
  /customers:
    get:
      summary: List customers
      operationId: listCustomers
      tags:
        - Customers
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: pageSize
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: List of customers
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomerList'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

  schemas:
    Customer:
      type: object
      required:
        - id
        - email
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        name:
          type: string
        createdAt:
          type: string
          format: date-time

    CustomerList:
      type: object
      properties:
        items:
          type: array
          items:
            $ref: '#/components/schemas/Customer'
        totalCount:
          type: integer
        page:
          type: integer
        pageSize:
          type: integer

    Error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
```

### Step 2: Create API in Azure Portal

#### Using Azure CLI

```bash
# Create API from OpenAPI specification
az apim api import \
    --resource-group myResourceGroup \
    --service-name myApimService \
    --path customer \
    --api-id customer-api \
    --specification-format OpenApi \
    --specification-path ./openapi.yaml \
    --display-name "Customer API" \
    --service-url https://backend.contoso.com

# Set API policies
az apim api policy update \
    --resource-group myResourceGroup \
    --service-name myApimService \
    --api-id customer-api \
    --xml-policy '<policies><inbound><base /><cors><allowed-origins><origin>*</origin></allowed-origins></cors></inbound><backend><base /></backend><outbound><base /></outbound><on-error><base /></on-error></policies>'
```

#### Using ARM Template

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "apimServiceName": {
            "type": "string"
        },
        "apiName": {
            "type": "string",
            "defaultValue": "customer-api"
        }
    },
    "resources": [
        {
            "type": "Microsoft.ApiManagement/service/apis",
            "apiVersion": "2021-08-01",
            "name": "[concat(parameters('apimServiceName'), '/', parameters('apiName'))]",
            "properties": {
                "displayName": "Customer API",
                "description": "API for managing customer data",
                "subscriptionRequired": true,
                "path": "customer",
                "protocols": ["https"],
                "isCurrent": true,
                "apiVersion": "v1",
                "apiVersionSetId": "[resourceId('Microsoft.ApiManagement/service/apiVersionSets', parameters('apimServiceName'), 'customer-api-version-set')]"
            }
        },
        {
            "type": "Microsoft.ApiManagement/service/apis/operations",
            "apiVersion": "2021-08-01",
            "name": "[concat(parameters('apimServiceName'), '/', parameters('apiName'), '/list-customers')]",
            "dependsOn": [
                "[resourceId('Microsoft.ApiManagement/service/apis', parameters('apimServiceName'), parameters('apiName'))]"
            ],
            "properties": {
                "displayName": "List Customers",
                "method": "GET",
                "urlTemplate": "/customers",
                "responses": [
                    {
                        "statusCode": 200,
                        "description": "Success"
                    }
                ]
            }
        }
    ]
}
```

### Step 3: Configure Backend Services

```xml
<!-- Named value for backend URL -->
<set-backend-service base-url="{{backend-service-url}}" />

<!-- Load balancing across multiple backends -->
<policies>
    <inbound>
        <base />
        <set-variable name="backendIndex" value="@(new Random().Next(0, 3))" />
        <choose>
            <when condition="@((int)context.Variables["backendIndex"] == 0)">
                <set-backend-service base-url="https://backend1.contoso.com" />
            </when>
            <when condition="@((int)context.Variables["backendIndex"] == 1)">
                <set-backend-service base-url="https://backend2.contoso.com" />
            </when>
            <otherwise>
                <set-backend-service base-url="https://backend3.contoso.com" />
            </otherwise>
        </choose>
    </inbound>
</policies>
```

### Step 4: Implement Versioning

```xml
<!-- Version via URL path -->
<policies>
    <inbound>
        <base />
        <set-backend-service base-url="@{
            var version = context.Request.MatchedParameters.GetValueOrDefault("version", "v1");
            return $"https://api-{version}.contoso.com";
        }" />
    </inbound>
</policies>

<!-- Version via header -->
<policies>
    <inbound>
        <base />
        <choose>
            <when condition="@(context.Request.Headers.GetValueOrDefault("Api-Version", "1") == "2")">
                <set-backend-service base-url="https://api-v2.contoso.com" />
            </when>
            <otherwise>
                <set-backend-service base-url="https://api-v1.contoso.com" />
            </otherwise>
        </choose>
    </inbound>
</policies>
```

---

## Testing Strategies

### Unit Testing Policies

#### Test Framework Setup

```csharp
using Microsoft.Azure.ApiManagement.PolicyToolkit.Testing;
using Xunit;

public class PolicyTests
{
    private readonly PolicyTestContext _context;

    public PolicyTests()
    {
        _context = new PolicyTestContext();
    }

    [Fact]
    public async Task InboundPolicy_ShouldAddRequestId()
    {
        // Arrange
        var policy = @"
            <inbound>
                <set-header name='X-Request-ID' exists-action='override'>
                    <value>@(Guid.NewGuid().ToString())</value>
                </set-header>
            </inbound>";

        _context.SetPolicy(policy);

        // Act
        var result = await _context.ExecuteInboundAsync();

        // Assert
        Assert.True(result.Request.Headers.ContainsKey("X-Request-ID"));
        Assert.True(Guid.TryParse(result.Request.Headers["X-Request-ID"], out _));
    }

    [Fact]
    public async Task OutboundPolicy_ShouldTransformResponse()
    {
        // Arrange
        var policy = @"
            <outbound>
                <set-body>@{
                    var original = context.Response.Body.As<JObject>();
                    original['transformed'] = true;
                    return original.ToString();
                }</set-body>
            </outbound>";

        _context.SetPolicy(policy);
        _context.Response.Body = "{\"data\": \"test\"}";

        // Act
        var result = await _context.ExecuteOutboundAsync();

        // Assert
        var body = JObject.Parse(result.Response.Body);
        Assert.True((bool)body["transformed"]);
    }
}
```

### Integration Testing

#### Postman Collection

```json
{
    "info": {
        "name": "Customer API Tests",
        "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
    },
    "item": [
        {
            "name": "Authentication Tests",
            "item": [
                {
                    "name": "Valid Token - Should Return 200",
                    "request": {
                        "method": "GET",
                        "header": [
                            {
                                "key": "Authorization",
                                "value": "Bearer {{valid_token}}"
                            }
                        ],
                        "url": "{{base_url}}/customers"
                    },
                    "event": [
                        {
                            "listen": "test",
                            "script": {
                                "exec": [
                                    "pm.test('Status code is 200', function () {",
                                    "    pm.response.to.have.status(200);",
                                    "});",
                                    "",
                                    "pm.test('Response has customers array', function () {",
                                    "    var jsonData = pm.response.json();",
                                    "    pm.expect(jsonData.items).to.be.an('array');",
                                    "});"
                                ]
                            }
                        }
                    ]
                },
                {
                    "name": "Invalid Token - Should Return 401",
                    "request": {
                        "method": "GET",
                        "header": [
                            {
                                "key": "Authorization",
                                "value": "Bearer invalid_token"
                            }
                        ],
                        "url": "{{base_url}}/customers"
                    },
                    "event": [
                        {
                            "listen": "test",
                            "script": {
                                "exec": [
                                    "pm.test('Status code is 401', function () {",
                                    "    pm.response.to.have.status(401);",
                                    "});"
                                ]
                            }
                        }
                    ]
                }
            ]
        },
        {
            "name": "Rate Limiting Tests",
            "item": [
                {
                    "name": "Exceed Rate Limit - Should Return 429",
                    "request": {
                        "method": "GET",
                        "header": [
                            {
                                "key": "Authorization",
                                "value": "Bearer {{valid_token}}"
                            }
                        ],
                        "url": "{{base_url}}/customers"
                    },
                    "event": [
                        {
                            "listen": "prerequest",
                            "script": {
                                "exec": [
                                    "// Send multiple requests to trigger rate limit",
                                    "const requests = [];",
                                    "for (let i = 0; i < 100; i++) {",
                                    "    requests.push(",
                                    "        pm.sendRequest({",
                                    "            url: pm.variables.get('base_url') + '/customers',",
                                    "            method: 'GET',",
                                    "            header: {",
                                    "                'Authorization': 'Bearer ' + pm.variables.get('valid_token')",
                                    "            }",
                                    "        })",
                                    "    );",
                                    "}",
                                    "Promise.all(requests);"
                                ]
                            }
                        },
                        {
                            "listen": "test",
                            "script": {
                                "exec": [
                                    "pm.test('Rate limit header present', function () {",
                                    "    pm.response.to.have.header('X-RateLimit-Remaining');",
                                    "});"
                                ]
                            }
                        }
                    ]
                }
            ]
        }
    ],
    "variable": [
        {
            "key": "base_url",
            "value": "https://api.contoso.com/v1"
        }
    ]
}
```

### Load Testing

#### Azure Load Testing Script

```yaml
# load-test-config.yaml
version: v0.1
testId: customer-api-load-test
displayName: Customer API Load Test
testPlan: load-test.jmx
engineInstances: 5
configurationFiles:
  - test-data.csv
failureCriteria:
  - avg(response_time_ms) > 500
  - percentage(error) > 5
  - p99(response_time_ms) > 2000
```

#### JMeter Test Plan (Simplified)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Customer API Load Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_URL" elementType="Argument">
            <stringProp name="Argument.name">BASE_URL</stringProp>
            <stringProp name="Argument.value">${__P(base_url,https://api.contoso.com)}</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="API Users">
        <intProp name="ThreadGroup.num_threads">100</intProp>
        <intProp name="ThreadGroup.ramp_time">60</intProp>
        <longProp name="ThreadGroup.duration">300</longProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="List Customers">
          <stringProp name="HTTPSampler.domain">${BASE_URL}</stringProp>
          <stringProp name="HTTPSampler.path">/v1/customers</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### Contract Testing

#### Pact Provider Verification

```csharp
using PactNet;
using PactNet.Verifier;
using Xunit;

public class CustomerApiProviderTests
{
    [Fact]
    public void EnsureCustomerApiHonorsConsumerPact()
    {
        var config = new PactVerifierConfig
        {
            ProviderVersion = "1.0.0",
            PublishVerificationResults = true
        };

        using var pactVerifier = new PactVerifier(config);

        pactVerifier
            .ServiceProvider("CustomerAPI", "https://api.contoso.com/v1")
            .WithPactBrokerSource(new Uri("https://pact-broker.contoso.com"))
            .WithProviderStateUrl(new Uri("https://api.contoso.com/v1/pact-states"))
            .Verify();
    }
}
```

---

## API Policies

### Security Policies

#### JWT Validation with Role-Based Access

```xml
<policies>
    <inbound>
        <base />

        <!-- Validate JWT -->
        <validate-jwt header-name="Authorization"
                      failed-validation-httpcode="401"
                      failed-validation-error-message="Unauthorized"
                      output-token-variable-name="jwt">
            <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration" />
            <audiences>
                <audience>api://customer-api</audience>
            </audiences>
            <issuers>
                <issuer>https://sts.windows.net/{tenant}/</issuer>
            </issuers>
            <required-claims>
                <claim name="roles" match="any">
                    <value>Customer.Read</value>
                    <value>Customer.ReadWrite</value>
                </claim>
            </required-claims>
        </validate-jwt>

        <!-- Extract and validate specific claims -->
        <set-variable name="userRoles" value="@{
            var jwt = (Jwt)context.Variables["jwt"];
            return jwt.Claims.GetValueOrDefault("roles", "").ToString();
        }" />

        <!-- Check write permission for POST/PUT/DELETE -->
        <choose>
            <when condition="@(new[] { "POST", "PUT", "DELETE" }.Contains(context.Request.Method))">
                <choose>
                    <when condition="@(!context.Variables.GetValueOrDefault<string>("userRoles").Contains("Customer.ReadWrite"))">
                        <return-response>
                            <set-status code="403" reason="Forbidden" />
                            <set-body>{"error": "Write permission required"}</set-body>
                        </return-response>
                    </when>
                </choose>
            </when>
        </choose>
    </inbound>
</policies>
```

#### API Key Validation

```xml
<policies>
    <inbound>
        <base />

        <!-- Check for API key in header or query -->
        <set-variable name="apiKey" value="@{
            var key = context.Request.Headers.GetValueOrDefault("X-API-Key", "");
            if (string.IsNullOrEmpty(key)) {
                key = context.Request.Url.Query.GetValueOrDefault("api_key", "");
            }
            return key;
        }" />

        <!-- Validate API key exists -->
        <choose>
            <when condition="@(string.IsNullOrEmpty((string)context.Variables["apiKey"]))">
                <return-response>
                    <set-status code="401" reason="Unauthorized" />
                    <set-body>{"error": "API key required"}</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Validate API key against cache/database -->
        <cache-lookup-value key="@($"api-key-{context.Variables["apiKey"]}")"
                           variable-name="keyData" />

        <choose>
            <when condition="@(!context.Variables.ContainsKey("keyData"))">
                <!-- Look up in backend -->
                <send-request mode="new" response-variable-name="keyLookup" timeout="10">
                    <set-url>@($"https://key-service.contoso.com/validate?key={context.Variables["apiKey"]}")</set-url>
                    <set-method>GET</set-method>
                </send-request>

                <choose>
                    <when condition="@(((IResponse)context.Variables["keyLookup"]).StatusCode != 200)">
                        <return-response>
                            <set-status code="401" reason="Unauthorized" />
                            <set-body>{"error": "Invalid API key"}</set-body>
                        </return-response>
                    </when>
                    <otherwise>
                        <!-- Cache valid key -->
                        <cache-store-value key="@($"api-key-{context.Variables["apiKey"]}")"
                                          value="@(((IResponse)context.Variables["keyLookup"]).Body.As<string>())"
                                          duration="3600" />
                    </otherwise>
                </choose>
            </when>
        </choose>
    </inbound>
</policies>
```

#### IP Filtering

```xml
<policies>
    <inbound>
        <base />

        <!-- Whitelist specific IPs -->
        <ip-filter action="allow">
            <address-range from="10.0.0.0" to="10.255.255.255" />
            <address>203.0.113.50</address>
        </ip-filter>

        <!-- Or blacklist specific IPs -->
        <ip-filter action="forbid">
            <address-range from="192.168.1.0" to="192.168.1.255" />
        </ip-filter>

        <!-- Dynamic IP filtering based on named value -->
        <choose>
            <when condition="@(!((string)context.Variables.GetValueOrDefault("allowedIps", "")).Contains(context.Request.IpAddress))">
                <return-response>
                    <set-status code="403" reason="Forbidden" />
                    <set-body>{"error": "IP not allowed"}</set-body>
                </return-response>
            </when>
        </choose>
    </inbound>
</policies>
```

#### mTLS (Mutual TLS) Configuration

```xml
<policies>
    <inbound>
        <base />

        <!-- Validate client certificate -->
        <choose>
            <when condition="@(context.Request.Certificate == null)">
                <return-response>
                    <set-status code="403" reason="Forbidden" />
                    <set-body>{"error": "Client certificate required"}</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Validate certificate thumbprint -->
        <choose>
            <when condition="@(!((string)context.Variables.GetValueOrDefault("allowedThumbprints", "")).Contains(context.Request.Certificate.Thumbprint))">
                <return-response>
                    <set-status code="403" reason="Forbidden" />
                    <set-body>{"error": "Invalid client certificate"}</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Validate certificate expiration -->
        <choose>
            <when condition="@(context.Request.Certificate.NotAfter < DateTime.UtcNow)">
                <return-response>
                    <set-status code="403" reason="Forbidden" />
                    <set-body>{"error": "Client certificate expired"}</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Validate issuer -->
        <choose>
            <when condition="@(context.Request.Certificate.Issuer != "CN=Contoso CA, O=Contoso, C=US")">
                <return-response>
                    <set-status code="403" reason="Forbidden" />
                    <set-body>{"error": "Invalid certificate issuer"}</set-body>
                </return-response>
            </when>
        </choose>
    </inbound>
</policies>
```

### Rate Limiting Policies

#### Basic Rate Limiting

```xml
<policies>
    <inbound>
        <base />

        <!-- Rate limit by subscription -->
        <rate-limit calls="100" renewal-period="60" />

        <!-- Quota by subscription -->
        <quota calls="10000" renewal-period="604800" /> <!-- Weekly -->
    </inbound>
</policies>
```

#### Advanced Rate Limiting by Key

```xml
<policies>
    <inbound>
        <base />

        <!-- Extract user ID from JWT -->
        <set-variable name="userId" value="@{
            var auth = context.Request.Headers.GetValueOrDefault("Authorization", "");
            if (auth.StartsWith("Bearer ")) {
                var token = auth.Substring(7);
                var handler = new System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler();
                var jwt = handler.ReadJwtToken(token);
                return jwt.Claims.FirstOrDefault(c => c.Type == "oid")?.Value ?? "anonymous";
            }
            return "anonymous";
        }" />

        <!-- Rate limit by user -->
        <rate-limit-by-key calls="100"
                          renewal-period="60"
                          counter-key="@((string)context.Variables["userId"])"
                          increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 300)" />

        <!-- Rate limit by user and operation -->
        <rate-limit-by-key calls="10"
                          renewal-period="60"
                          counter-key="@($"{context.Variables["userId"]}-{context.Operation.Id}")" />

        <!-- Different limits for different tiers -->
        <choose>
            <when condition="@(context.Product.Name == "Premium")">
                <rate-limit-by-key calls="1000" renewal-period="60" counter-key="@(context.Subscription.Id)" />
            </when>
            <when condition="@(context.Product.Name == "Standard")">
                <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Id)" />
            </when>
            <otherwise>
                <rate-limit-by-key calls="10" renewal-period="60" counter-key="@(context.Subscription.Id)" />
            </otherwise>
        </choose>
    </inbound>

    <outbound>
        <base />

        <!-- Add rate limit headers -->
        <set-header name="X-RateLimit-Limit" exists-action="override">
            <value>100</value>
        </set-header>
        <set-header name="X-RateLimit-Remaining" exists-action="override">
            <value>@(context.Variables.GetValueOrDefault<string>("remainingCalls", "unknown"))</value>
        </set-header>
    </outbound>
</policies>
```

#### Quota Management

```xml
<policies>
    <inbound>
        <base />

        <!-- Quota by key with multiple periods -->
        <quota-by-key calls="1000"
                      renewal-period="86400"
                      counter-key="@(context.Subscription.Id)"
                      increment-condition="@(context.Response.StatusCode < 400)" />

        <!-- Bandwidth quota -->
        <quota-by-key bandwidth="104857600"
                      renewal-period="604800"
                      counter-key="@(context.Subscription.Id)" /> <!-- 100MB weekly -->
    </inbound>
</policies>
```

### Caching Policies

```xml
<policies>
    <inbound>
        <base />

        <!-- Cache lookup -->
        <cache-lookup vary-by-developer="false"
                      vary-by-developer-groups="false"
                      caching-type="internal"
                      downstream-caching-type="public"
                      must-revalidate="false">
            <vary-by-header>Accept</vary-by-header>
            <vary-by-header>Accept-Encoding</vary-by-header>
            <vary-by-query-parameter>page</vary-by-query-parameter>
            <vary-by-query-parameter>pageSize</vary-by-query-parameter>
        </cache-lookup>
    </inbound>

    <outbound>
        <base />

        <!-- Store in cache -->
        <cache-store duration="3600" />

        <!-- Or conditional caching -->
        <choose>
            <when condition="@(context.Response.StatusCode == 200)">
                <cache-store duration="3600" />
            </when>
        </choose>
    </outbound>
</policies>
```

### CORS Policy

```xml
<policies>
    <inbound>
        <base />

        <!-- CORS configuration -->
        <cors allow-credentials="true">
            <allowed-origins>
                <origin>https://app.contoso.com</origin>
                <origin>https://admin.contoso.com</origin>
            </allowed-origins>
            <allowed-methods preflight-result-max-age="300">
                <method>GET</method>
                <method>POST</method>
                <method>PUT</method>
                <method>DELETE</method>
                <method>OPTIONS</method>
            </allowed-methods>
            <allowed-headers>
                <header>Content-Type</header>
                <header>Authorization</header>
                <header>X-Request-ID</header>
            </allowed-headers>
            <expose-headers>
                <header>X-RateLimit-Limit</header>
                <header>X-RateLimit-Remaining</header>
            </expose-headers>
        </cors>
    </inbound>
</policies>
```

---

## Vendor Access Configuration

### Product and Subscription Setup

#### Creating Products for Vendors

```bash
# Create a product for external vendors
az apim product create \
    --resource-group myResourceGroup \
    --service-name myApimService \
    --product-id vendor-standard \
    --display-name "Vendor Standard" \
    --description "Standard API access for vendors" \
    --subscription-required true \
    --approval-required true \
    --subscriptions-limit 5 \
    --state published

# Associate APIs with product
az apim product api add \
    --resource-group myResourceGroup \
    --service-name myApimService \
    --product-id vendor-standard \
    --api-id customer-api
```

#### Vendor-Specific Policies

```xml
<policies>
    <inbound>
        <base />

        <!-- Identify vendor from subscription -->
        <set-variable name="vendorId" value="@(context.Subscription.Name)" />

        <!-- Vendor-specific rate limits -->
        <choose>
            <when condition="@(context.Product.Name == "Vendor Premium")">
                <rate-limit-by-key calls="10000" renewal-period="60" counter-key="@(context.Subscription.Id)" />
                <quota-by-key calls="1000000" renewal-period="2592000" counter-key="@(context.Subscription.Id)" />
            </when>
            <when condition="@(context.Product.Name == "Vendor Standard")">
                <rate-limit-by-key calls="1000" renewal-period="60" counter-key="@(context.Subscription.Id)" />
                <quota-by-key calls="100000" renewal-period="2592000" counter-key="@(context.Subscription.Id)" />
            </when>
            <otherwise>
                <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Id)" />
                <quota-by-key calls="10000" renewal-period="2592000" counter-key="@(context.Subscription.Id)" />
            </otherwise>
        </choose>

        <!-- Data filtering based on vendor -->
        <set-header name="X-Vendor-ID" exists-action="override">
            <value>@((string)context.Variables["vendorId"])</value>
        </set-header>
    </inbound>

    <outbound>
        <base />

        <!-- Filter response data based on vendor access -->
        <choose>
            <when condition="@(context.Product.Name != "Vendor Premium")">
                <set-body>@{
                    var response = context.Response.Body.As<JObject>();

                    // Remove sensitive fields for non-premium vendors
                    if (response["items"] != null) {
                        foreach (JObject item in response["items"]) {
                            item.Remove("internalNotes");
                            item.Remove("costPrice");
                            item.Remove("supplierInfo");
                        }
                    }

                    return response.ToString();
                }</set-body>
            </when>
        </choose>
    </outbound>
</policies>
```

### OAuth 2.0 Client Credentials for Vendors

```xml
<policies>
    <inbound>
        <base />

        <!-- Validate vendor OAuth token -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
            <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
            <audiences>
                <audience>api://vendor-api</audience>
            </audiences>
            <required-claims>
                <claim name="azp" match="any">
                    <!-- Allowed vendor application IDs -->
                    <value>{{vendor-app-id-1}}</value>
                    <value>{{vendor-app-id-2}}</value>
                    <value>{{vendor-app-id-3}}</value>
                </claim>
            </required-claims>
        </validate-jwt>

        <!-- Map vendor app ID to vendor profile -->
        <set-variable name="vendorProfile" value="@{
            var appId = context.Request.Headers.GetValueOrDefault("Authorization", "");
            // Extract app ID from token and map to vendor profile
            var vendorMap = new Dictionary<string, string> {
                { "app-id-1", "vendor-acme" },
                { "app-id-2", "vendor-globex" },
                { "app-id-3", "vendor-initech" }
            };
            return vendorMap.GetValueOrDefault(appId, "unknown");
        }" />
    </inbound>
</policies>
```

### Vendor Onboarding Workflow

```xml
<!-- Self-service vendor registration API -->
<policies>
    <inbound>
        <base />

        <!-- Validate registration request -->
        <set-variable name="registration" value="@(context.Request.Body.As<JObject>(preserveContent: true))" />

        <!-- Check required fields -->
        <choose>
            <when condition="@{
                var reg = (JObject)context.Variables["registration"];
                return string.IsNullOrEmpty(reg["companyName"]?.ToString()) ||
                       string.IsNullOrEmpty(reg["email"]?.ToString()) ||
                       string.IsNullOrEmpty(reg["contactName"]?.ToString());
            }">
                <return-response>
                    <set-status code="400" reason="Bad Request" />
                    <set-body>{"error": "Missing required fields: companyName, email, contactName"}</set-body>
                </return-response>
            </when>
        </choose>
    </inbound>

    <backend>
        <!-- Create vendor in backend system -->
        <send-request mode="new" response-variable-name="vendorResponse" timeout="30">
            <set-url>https://vendor-service.contoso.com/api/vendors</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>@(((JObject)context.Variables["registration"]).ToString())</set-body>
        </send-request>

        <!-- Create subscription in APIM -->
        <send-request mode="new" response-variable-name="subscriptionResponse" timeout="30">
            <set-url>@($"https://management.azure.com/subscriptions/{{subscription-id}}/resourceGroups/{{resource-group}}/providers/Microsoft.ApiManagement/service/{{service-name}}/subscriptions/{Guid.NewGuid()}?api-version=2021-08-01")</set-url>
            <set-method>PUT</set-method>
            <set-header name="Authorization" exists-action="override">
                <value>Bearer {{management-token}}</value>
            </set-header>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>@{
                var reg = (JObject)context.Variables["registration"];
                var vendorResult = ((IResponse)context.Variables["vendorResponse"]).Body.As<JObject>();

                return new JObject(
                    new JProperty("properties", new JObject(
                        new JProperty("scope", "/products/vendor-standard"),
                        new JProperty("displayName", reg["companyName"].ToString()),
                        new JProperty("ownerId", $"/users/{vendorResult["userId"]}")
                    ))
                ).ToString();
            }</set-body>
        </send-request>
    </backend>

    <outbound>
        <base />

        <set-body>@{
            var vendorResult = ((IResponse)context.Variables["vendorResponse"]).Body.As<JObject>();
            var subscriptionResult = ((IResponse)context.Variables["subscriptionResponse"]).Body.As<JObject>();

            return new JObject(
                new JProperty("vendorId", vendorResult["vendorId"]),
                new JProperty("subscriptionKey", subscriptionResult["properties"]?["primaryKey"]),
                new JProperty("status", "pending_approval"),
                new JProperty("message", "Your registration is pending approval. You will receive an email once approved.")
            ).ToString();
        }</set-body>
    </outbound>
</policies>
```

### Vendor Usage Analytics

```xml
<policies>
    <inbound>
        <base />
    </inbound>

    <outbound>
        <base />

        <!-- Log vendor usage to Event Hub -->
        <log-to-eventhub logger-id="vendor-analytics">@{
            return new JObject(
                new JProperty("eventType", "VendorApiCall"),
                new JProperty("vendorId", context.Subscription.Name),
                new JProperty("productName", context.Product.Name),
                new JProperty("operationId", context.Operation.Id),
                new JProperty("responseCode", context.Response.StatusCode),
                new JProperty("responseTime", context.Elapsed.TotalMilliseconds),
                new JProperty("requestSize", context.Request.Body?.As<string>()?.Length ?? 0),
                new JProperty("responseSize", context.Response.Body?.As<string>()?.Length ?? 0),
                new JProperty("timestamp", DateTime.UtcNow),
                new JProperty("clientIp", context.Request.IpAddress)
            ).ToString();
        }</log-to-eventhub>
    </outbound>
</policies>
```

---

## Data Lake Integration

### Overview

Integrating Azure API Management with Data Lake enables:
- API traffic analytics and insights
- Compliance and audit logging
- Machine learning on API usage patterns
- Long-term data retention

### Architecture Patterns

#### Pattern 1: Direct Event Hub to Data Lake

```
API Management → Event Hub → Stream Analytics → Data Lake Storage
```

#### Pattern 2: Via Azure Functions

```
API Management → Event Hub → Azure Function → Data Lake Storage
```

### Event Hub Logger Configuration

#### Create Event Hub Logger

```bash
# Create Event Hub namespace
az eventhubs namespace create \
    --name apim-analytics \
    --resource-group myResourceGroup \
    --location eastus \
    --sku Standard

# Create Event Hub
az eventhubs eventhub create \
    --name api-logs \
    --namespace-name apim-analytics \
    --resource-group myResourceGroup \
    --partition-count 4 \
    --message-retention 7

# Get connection string
CONNECTION_STRING=$(az eventhubs namespace authorization-rule keys list \
    --namespace-name apim-analytics \
    --resource-group myResourceGroup \
    --name RootManageSharedAccessKey \
    --query primaryConnectionString -o tsv)

# Create APIM logger
az apim logger create \
    --resource-group myResourceGroup \
    --service-name myApimService \
    --logger-id datalake-logger \
    --logger-type azureEventHub \
    --description "Logger for Data Lake analytics" \
    --credentials "connectionString=$CONNECTION_STRING" "name=api-logs"
```

### Logging Policies for Data Lake

#### Comprehensive Request/Response Logging

```xml
<policies>
    <inbound>
        <base />

        <!-- Capture request details -->
        <set-variable name="requestBody" value="@(context.Request.Body?.As<string>(preserveContent: true) ?? "")" />
        <set-variable name="requestTimestamp" value="@(DateTime.UtcNow)" />
    </inbound>

    <outbound>
        <base />

        <!-- Log to Event Hub for Data Lake ingestion -->
        <log-to-eventhub logger-id="datalake-logger">@{
            var requestBody = context.Variables.GetValueOrDefault<string>("requestBody", "");
            var responseBody = context.Response.Body?.As<string>(preserveContent: true) ?? "";

            // Mask sensitive data
            Func<string, string> maskSensitive = (input) => {
                if (string.IsNullOrEmpty(input)) return input;

                // Mask credit card numbers
                input = System.Text.RegularExpressions.Regex.Replace(
                    input,
                    @"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b",
                    "****-****-****-****");

                // Mask SSN
                input = System.Text.RegularExpressions.Regex.Replace(
                    input,
                    @"\b\d{3}[-\s]?\d{2}[-\s]?\d{4}\b",
                    "***-**-****");

                return input;
            };

            return new JObject(
                // Request information
                new JProperty("request", new JObject(
                    new JProperty("timestamp", context.Variables["requestTimestamp"]),
                    new JProperty("method", context.Request.Method),
                    new JProperty("url", context.Request.Url.ToString()),
                    new JProperty("headers", new JObject(
                        context.Request.Headers
                            .Where(h => !new[] { "Authorization", "Ocp-Apim-Subscription-Key" }.Contains(h.Key))
                            .Select(h => new JProperty(h.Key, string.Join(", ", h.Value)))
                    )),
                    new JProperty("body", maskSensitive(requestBody)),
                    new JProperty("clientIp", context.Request.IpAddress)
                )),

                // Response information
                new JProperty("response", new JObject(
                    new JProperty("timestamp", DateTime.UtcNow),
                    new JProperty("statusCode", context.Response.StatusCode),
                    new JProperty("statusReason", context.Response.StatusReason),
                    new JProperty("headers", new JObject(
                        context.Response.Headers.Select(h => new JProperty(h.Key, string.Join(", ", h.Value)))
                    )),
                    new JProperty("body", maskSensitive(responseBody.Length > 10000 ? responseBody.Substring(0, 10000) + "...[truncated]" : responseBody))
                )),

                // Context information
                new JProperty("context", new JObject(
                    new JProperty("apiId", context.Api.Id),
                    new JProperty("apiName", context.Api.Name),
                    new JProperty("operationId", context.Operation.Id),
                    new JProperty("operationName", context.Operation.Name),
                    new JProperty("productId", context.Product?.Id),
                    new JProperty("productName", context.Product?.Name),
                    new JProperty("subscriptionId", context.Subscription?.Id),
                    new JProperty("subscriptionName", context.Subscription?.Name),
                    new JProperty("userId", context.User?.Id)
                )),

                // Performance metrics
                new JProperty("metrics", new JObject(
                    new JProperty("totalTime", context.Elapsed.TotalMilliseconds),
                    new JProperty("requestSize", requestBody.Length),
                    new JProperty("responseSize", responseBody.Length)
                )),

                // Correlation
                new JProperty("correlationId", context.RequestId),
                new JProperty("traceId", context.Request.Headers.GetValueOrDefault("traceparent", context.RequestId))
            ).ToString();
        }</log-to-eventhub>
    </outbound>

    <on-error>
        <base />

        <!-- Log errors -->
        <log-to-eventhub logger-id="datalake-logger">@{
            return new JObject(
                new JProperty("eventType", "Error"),
                new JProperty("error", new JObject(
                    new JProperty("source", context.LastError.Source),
                    new JProperty("reason", context.LastError.Reason),
                    new JProperty("message", context.LastError.Message),
                    new JProperty("scope", context.LastError.Scope),
                    new JProperty("section", context.LastError.Section)
                )),
                new JProperty("context", new JObject(
                    new JProperty("apiId", context.Api.Id),
                    new JProperty("operationId", context.Operation.Id),
                    new JProperty("subscriptionId", context.Subscription?.Id)
                )),
                new JProperty("correlationId", context.RequestId),
                new JProperty("timestamp", DateTime.UtcNow)
            ).ToString();
        }</log-to-eventhub>
    </on-error>
</policies>
```

### Stream Analytics Job for Data Lake

```sql
-- Stream Analytics query to process API logs
WITH ApiLogs AS (
    SELECT
        GetRecordPropertyValue(GetArrayElement(EventProcessedUtcTime, 0), 'timestamp') AS EventTime,
        correlationId,
        context.apiId,
        context.apiName,
        context.operationId,
        context.productName,
        context.subscriptionName,
        request.method AS HttpMethod,
        request.clientIp AS ClientIP,
        response.statusCode AS StatusCode,
        metrics.totalTime AS ResponseTimeMs,
        metrics.requestSize AS RequestSizeBytes,
        metrics.responseSize AS ResponseSizeBytes
    FROM
        [api-logs-input]
)

-- Output to Data Lake for raw storage
SELECT
    *
INTO
    [datalake-raw-output]
FROM
    ApiLogs

-- Aggregated metrics per minute
SELECT
    System.Timestamp() AS WindowEnd,
    apiName,
    operationId,
    productName,
    COUNT(*) AS RequestCount,
    AVG(ResponseTimeMs) AS AvgResponseTime,
    MAX(ResponseTimeMs) AS MaxResponseTime,
    MIN(ResponseTimeMs) AS MinResponseTime,
    SUM(RequestSizeBytes) AS TotalRequestBytes,
    SUM(ResponseSizeBytes) AS TotalResponseBytes,
    SUM(CASE WHEN StatusCode >= 200 AND StatusCode < 300 THEN 1 ELSE 0 END) AS SuccessCount,
    SUM(CASE WHEN StatusCode >= 400 AND StatusCode < 500 THEN 1 ELSE 0 END) AS ClientErrorCount,
    SUM(CASE WHEN StatusCode >= 500 THEN 1 ELSE 0 END) AS ServerErrorCount
INTO
    [datalake-aggregated-output]
FROM
    ApiLogs
GROUP BY
    apiName,
    operationId,
    productName,
    TumblingWindow(minute, 1)

-- Error alerts
SELECT
    correlationId,
    apiName,
    operationId,
    StatusCode,
    ClientIP,
    EventTime
INTO
    [error-alerts-output]
FROM
    ApiLogs
WHERE
    StatusCode >= 500
```

### Data Lake Storage Structure

```
/api-analytics/
├── raw/
│   ├── year=2024/
│   │   ├── month=01/
│   │   │   ├── day=15/
│   │   │   │   ├── hour=00/
│   │   │   │   │   └── api-logs-00.parquet
│   │   │   │   ├── hour=01/
│   │   │   │   │   └── api-logs-01.parquet
│   │   │   │   └── ...
├── aggregated/
│   ├── year=2024/
│   │   ├── month=01/
│   │   │   └── metrics-2024-01.parquet
├── errors/
│   ├── year=2024/
│   │   ├── month=01/
│   │   │   └── errors-2024-01.parquet
└── schemas/
    ├── raw-schema.json
    ├── aggregated-schema.json
    └── errors-schema.json
```

### Synapse Analytics Integration

```sql
-- Create external table for API logs
CREATE EXTERNAL TABLE api_logs (
    correlationId VARCHAR(36),
    eventTime DATETIME2,
    apiId VARCHAR(100),
    apiName VARCHAR(200),
    operationId VARCHAR(100),
    productName VARCHAR(200),
    subscriptionName VARCHAR(200),
    httpMethod VARCHAR(10),
    clientIP VARCHAR(45),
    statusCode INT,
    responseTimeMs FLOAT,
    requestSizeBytes BIGINT,
    responseSizeBytes BIGINT
)
WITH (
    LOCATION = '/api-analytics/raw/',
    DATA_SOURCE = DataLakeStorage,
    FILE_FORMAT = ParquetFormat
);

-- Query for API performance analysis
SELECT
    apiName,
    operationId,
    COUNT(*) AS TotalRequests,
    AVG(responseTimeMs) AS AvgResponseTime,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY responseTimeMs) AS P95ResponseTime,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY responseTimeMs) AS P99ResponseTime,
    SUM(CASE WHEN statusCode >= 500 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS ErrorRate
FROM api_logs
WHERE eventTime >= DATEADD(day, -7, GETDATE())
GROUP BY apiName, operationId
ORDER BY TotalRequests DESC;

-- Vendor usage analysis
SELECT
    subscriptionName AS Vendor,
    apiName,
    COUNT(*) AS ApiCalls,
    SUM(requestSizeBytes + responseSizeBytes) / 1024 / 1024 AS TotalDataMB,
    AVG(responseTimeMs) AS AvgResponseTime
FROM api_logs
WHERE eventTime >= DATEADD(month, -1, GETDATE())
GROUP BY subscriptionName, apiName
ORDER BY ApiCalls DESC;
```

### Power BI Dashboard Queries

```dax
// DAX measure for error rate
Error Rate =
DIVIDE(
    CALCULATE(COUNT(api_logs[correlationId]), api_logs[statusCode] >= 500),
    COUNT(api_logs[correlationId]),
    0
) * 100

// DAX measure for P95 response time
P95 Response Time =
PERCENTILE.INC(api_logs[responseTimeMs], 0.95)

// DAX measure for requests per second
Requests Per Second =
DIVIDE(
    COUNT(api_logs[correlationId]),
    DATEDIFF(MIN(api_logs[eventTime]), MAX(api_logs[eventTime]), SECOND),
    0
)
```

### Diagnostic Settings Configuration

```bash
# Enable diagnostic settings for comprehensive logging
az monitor diagnostic-settings create \
    --name apim-diagnostics \
    --resource $(az apim show -g myResourceGroup -n myApimService --query id -o tsv) \
    --storage-account $(az storage account show -g myResourceGroup -n mystorageaccount --query id -o tsv) \
    --event-hub-name api-diagnostics \
    --event-hub-rule $(az eventhubs namespace authorization-rule show \
        --namespace-name apim-analytics \
        --resource-group myResourceGroup \
        --name RootManageSharedAccessKey \
        --query id -o tsv) \
    --logs '[
        {
            "category": "GatewayLogs",
            "enabled": true,
            "retentionPolicy": {
                "enabled": true,
                "days": 90
            }
        },
        {
            "category": "WebSocketConnectionLogs",
            "enabled": true,
            "retentionPolicy": {
                "enabled": true,
                "days": 30
            }
        }
    ]' \
    --metrics '[
        {
            "category": "AllMetrics",
            "enabled": true,
            "retentionPolicy": {
                "enabled": true,
                "days": 90
            }
        }
    ]'
```

---

## Best Practices Summary

### Security
- Always validate JWTs with proper audience and issuer checks
- Implement rate limiting and quotas at multiple levels
- Use mTLS for sensitive APIs
- Mask sensitive data in logs
- Regularly rotate API keys and certificates

### Performance
- Implement caching strategies appropriate to data volatility
- Use async operations for non-critical backend calls
- Set appropriate timeouts for backend services
- Use circuit breakers for unreliable backends

### Operations
- Log all API calls with correlation IDs
- Implement comprehensive error handling
- Use named values for environment-specific configuration
- Version your APIs from the start
- Document all APIs using OpenAPI specifications

### Vendor Management
- Create separate products for different vendor tiers
- Implement approval workflows for vendor onboarding
- Monitor vendor usage and enforce quotas
- Provide self-service documentation via developer portal

### Data Lake Integration
- Structure data for efficient querying
- Implement data retention policies
- Mask or encrypt sensitive data before storage
- Create aggregated views for common analytics queries

---

## Additional Resources

- [Azure API Management Documentation](https://docs.microsoft.com/azure/api-management/)
- [API Management Policies Reference](https://docs.microsoft.com/azure/api-management/api-management-policies)
- [Event Hub Integration](https://docs.microsoft.com/azure/api-management/api-management-howto-log-event-hubs)
- [Azure Data Lake Storage](https://docs.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)
- [Azure Synapse Analytics](https://docs.microsoft.com/azure/synapse-analytics/)
