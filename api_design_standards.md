# API Design Standards

**Technology-agnostic principles for building great APIs**

These standards apply to any API style: REST, GraphQL, gRPC, or custom protocols. Focus on principles that make APIs easy to use, debug, and maintain.

---

## 1. Consistent Response Formats

### Principle

**Standardize success and error responses across all endpoints.**

Every API response should follow the same structural pattern, making client code predictable and debugging easier.

### Why This Matters

- **Predictable client code** - Same parsing logic works for all endpoints
- **Easier debugging** - Know exactly where to find error details
- **Better documentation** - One response format to document
- **Simpler testing** - Test response handling once, apply everywhere

### Standard Response Format

```typescript
// Success response
interface APISuccessResponse<T> {
    success: true;
    data: T;
    metadata?: {
        timestamp: string;
        requestId: string;
        [key: string]: any;
    };
}

// Error response
interface APIErrorResponse {
    success: false;
    error: {
        code: string;        // Machine-readable error code
        message: string;     // Human-readable message
        details?: any;       // Additional error context
    };
    metadata?: {
        timestamp: string;
        requestId: string;
        [key: string]: any;
    };
}
```

### Examples

#### ✅ Good: Consistent Format
```json
// Success - Get User
{
    "success": true,
    "data": {
        "userId": "user_123",
        "name": "John Doe",
        "email": "john@example.com"
    },
    "metadata": {
        "timestamp": "2025-10-02T12:00:00Z",
        "requestId": "req_abc123"
    }
}

// Success - List Users
{
    "success": true,
    "data": {
        "users": [...],
        "totalCount": 150,
        "page": 1,
        "pageSize": 20
    },
    "metadata": {
        "timestamp": "2025-10-02T12:00:00Z",
        "requestId": "req_abc124"
    }
}

// Error - Validation Failed
{
    "success": false,
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid email format",
        "details": {
            "field": "email",
            "providedValue": "invalid-email",
            "expectedFormat": "user@domain.com"
        }
    },
    "metadata": {
        "timestamp": "2025-10-02T12:00:00Z",
        "requestId": "req_abc125"
    }
}
```

**Benefits:**
- Client knows `success: true` means data is in `data` field
- Client knows errors are in `error.code` and `error.message`
- `requestId` always in same place for debugging
- Consistent across all endpoints

#### ❌ Bad: Inconsistent Formats
```json
// Endpoint 1: Object response
{
    "user": {...}
}

// Endpoint 2: Array response
[...]

// Endpoint 3: String status
"ok"

// Endpoint 4: Custom error format
{
    "errorCode": 400,
    "msg": "Bad request"
}
```

**Problems:**
- Client needs different parsing logic for each endpoint
- Hard to know if request succeeded
- Error handling inconsistent
- Debugging requires knowing each endpoint's unique format

---

## 2. Explicit Field Naming

### Principle

**Use clear, self-documenting field names. Avoid abbreviations and ambiguous terms.**

Field names should be immediately understandable without documentation.

### Why This Matters

- **Self-documenting APIs** - Reduce need for docs
- **Fewer mistakes** - Clear names prevent misuse
- **Better DX** - Developers know what fields mean
- **Easier onboarding** - New developers understand quickly

### Naming Guidelines

**✅ Good Names:**
- `userId` - Clear, specific
- `createdAt` - Obvious meaning
- `isActive` - Boolean intent clear
- `totalPrice` - Unambiguous

**❌ Bad Names:**
- `uid` - What kind of ID?
- `created` - Date? Boolean? String?
- `active` - Boolean? Status string?
- `price` - Unit price? Total? Before/after tax?

### Examples

#### ❌ Bad: Ambiguous Names
```json
{
    "id": "123",           // ID of what?
    "status": "active",    // Status of what?
    "date": "2025-10-02",  // Created? Updated? Expires?
    "price": 99.99,        // Currency? Before/after tax?
    "qty": 5               // Abbreviation unclear
}
```

#### ✅ Good: Explicit Names
```json
{
    "orderId": "order_123",
    "orderStatus": "active",
    "createdAt": "2025-10-02T12:00:00Z",
    "updatedAt": "2025-10-02T13:30:00Z",
    "totalPriceUSD": 99.99,
    "taxAmountUSD": 8.00,
    "quantity": 5
}
```

### Field Naming Conventions

**Booleans:**
- Prefix with `is`, `has`, `should`, `can`
- Examples: `isActive`, `hasPermission`, `shouldNotify`, `canEdit`

**Dates/Times:**
- Suffix with `At` for timestamps
- Examples: `createdAt`, `updatedAt`, `deletedAt`, `expiresAt`

**IDs:**
- Suffix with type: `userId`, `orderId`, `productId`
- Never just `id` unless context is obvious

**Counts/Quantities:**
- Use full word `count` or `quantity`, not `cnt` or `qty`
- Examples: `totalCount`, `itemQuantity`, `pageCount`

**Money:**
- Include currency: `priceUSD`, `totalEUR`
- Or separate fields: `amount` + `currency`

---

## 3. Proper Error Handling

### Principle

**Return meaningful errors with enough context for developers to fix the problem.**

Errors should tell developers:
1. What went wrong
2. Why it went wrong
3. How to fix it

### Why This Matters

- **Faster debugging** - Developers know exactly what's wrong
- **Better UX** - Specific errors enable better user messages
- **Reduced support load** - Developers can self-serve fixes
- **Easier testing** - Can verify specific error scenarios

### Error Response Template

```json
{
    "success": false,
    "error": {
        "code": "MACHINE_READABLE_CODE",
        "message": "Human-readable explanation",
        "details": {
            "field": "which field caused error",
            "providedValue": "what was sent",
            "expectedValue": "what was expected",
            "helpUrl": "https://docs.example.com/errors/code"
        }
    },
    "metadata": {
        "timestamp": "2025-10-02T12:00:00Z",
        "requestId": "req_abc123"
    }
}
```

### Examples

#### ❌ Bad: Vague Errors
```json
{
    "error": "Bad request"
}
```
**Problems:**
- What was bad about it?
- Which field?
- How to fix?

#### ✅ Good: Specific Errors
```json
{
    "success": false,
    "error": {
        "code": "INVALID_EMAIL_FORMAT",
        "message": "Email address must be in valid format (user@domain.com)",
        "details": {
            "field": "email",
            "providedValue": "john.doe@",
            "expectedFormat": "user@domain.com",
            "examples": ["john@example.com", "jane.doe@company.co.uk"]
        }
    },
    "metadata": {
        "requestId": "req_abc123"
    }
}
```

### Error Code Standards

**Format:** `CATEGORY_SPECIFIC_REASON`

**Categories:**
- `VALIDATION_*` - Input validation errors
- `AUTH_*` - Authentication/authorization errors
- `NOT_FOUND_*` - Resource not found
- `CONFLICT_*` - Resource state conflicts
- `RATE_LIMIT_*` - Rate limiting
- `INTERNAL_*` - Server errors

**Examples:**
```
VALIDATION_MISSING_FIELD
VALIDATION_INVALID_FORMAT
VALIDATION_OUT_OF_RANGE

AUTH_MISSING_TOKEN
AUTH_INVALID_TOKEN
AUTH_INSUFFICIENT_PERMISSIONS

NOT_FOUND_USER
NOT_FOUND_ORDER

CONFLICT_DUPLICATE_EMAIL
CONFLICT_RESOURCE_LOCKED

RATE_LIMIT_EXCEEDED

INTERNAL_DATABASE_ERROR
INTERNAL_SERVICE_UNAVAILABLE
```

### HTTP Status Codes

Match HTTP status to error category:

| Status | Category | Use For |
|--------|----------|---------|
| 400 | Bad Request | Validation errors |
| 401 | Unauthorized | Missing/invalid auth |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource state conflicts |
| 429 | Too Many Requests | Rate limiting |
| 500 | Internal Server Error | Server failures |
| 503 | Service Unavailable | Temporary outages |

---

## 4. Versioning Strategy

### Principle

**Plan for API evolution from day one. Use explicit versioning.**

APIs change. Have a strategy for evolving without breaking existing clients.

### Why This Matters

- **Prevents breaking changes** - Old clients continue working
- **Enables iteration** - Can improve API while supporting legacy
- **Clear migration path** - Clients know when to upgrade
- **Professional impression** - Shows maturity and thoughtfulness

### Versioning Approaches

#### Option 1: URL Path Versioning
```
https://api.example.com/v1/users
https://api.example.com/v2/users
```

**Pros:**
- Very explicit
- Easy to route
- Clear in logs

**Cons:**
- Seems heavyweight for minor changes
- URL changes between versions

#### Option 2: Header Versioning
```
GET /users
Accept: application/vnd.api+json; version=1
```

**Pros:**
- URL stays same
- Supports content negotiation
- More flexible

**Cons:**
- Less visible
- Harder to test manually
- Requires header support

#### Option 3: Query Parameter
```
https://api.example.com/users?version=1
```

**Pros:**
- Easy to test
- URL-based like path versioning
- Optional (can default to latest)

**Cons:**
- Pollutes query params
- Less RESTful

### Versioning Best Practices

1. **Major versions for breaking changes**
   - Changed response format
   - Removed fields
   - Changed field types
   - Changed behavior

2. **Minor versions or no version change for additions**
   - New optional fields
   - New endpoints
   - New optional parameters

3. **Document version lifecycle**
   ```markdown
   ## Version Lifecycle

   - v1: Released 2024-01-01, Deprecated 2025-01-01, Sunset 2026-01-01
   - v2: Released 2025-01-01, Current
   ```

4. **Provide migration guides**
   ```markdown
   ## Migrating from v1 to v2

   **Breaking Changes:**
   - `user.name` split into `user.firstName` and `user.lastName`
   - `created` renamed to `createdAt` with ISO 8601 format

   **Migration Example:**
   ```javascript
   // v1
   const name = user.name;
   const created = new Date(user.created);

   // v2
   const name = `${user.firstName} ${user.lastName}`;
   const created = new Date(user.createdAt);
   ```

---

## 5. Pagination Standards

### Principle

**Standardize pagination across all list endpoints.**

### Standard Pagination Format

```json
{
    "success": true,
    "data": {
        "items": [...],
        "pagination": {
            "page": 1,
            "pageSize": 20,
            "totalCount": 150,
            "totalPages": 8,
            "hasNextPage": true,
            "hasPreviousPage": false
        }
    }
}
```

### Query Parameters

```
GET /users?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc
```

**Standard parameters:**
- `page` - Current page (1-indexed)
- `pageSize` - Items per page (default: 20, max: 100)
- `sortBy` - Field to sort by
- `sortOrder` - `asc` or `desc`

---

## 6. Request Validation

### Principle

**Validate all inputs. Return specific errors for each validation failure.**

### Validation Response Format

```json
{
    "success": false,
    "error": {
        "code": "VALIDATION_FAILED",
        "message": "Request validation failed",
        "details": {
            "errors": [
                {
                    "field": "email",
                    "code": "INVALID_FORMAT",
                    "message": "Email must be valid format",
                    "providedValue": "invalid"
                },
                {
                    "field": "age",
                    "code": "OUT_OF_RANGE",
                    "message": "Age must be between 0 and 120",
                    "providedValue": 150,
                    "min": 0,
                    "max": 120
                }
            ]
        }
    }
}
```

---

## Summary

Apply these six principles to build developer-friendly APIs:

1. **Consistent Response Formats** → Predictable client code
2. **Explicit Field Naming** → Self-documenting APIs
3. **Proper Error Handling** → Specific, actionable errors
4. **Versioning Strategy** → Support evolution without breaking clients
5. **Pagination Standards** → Consistent list handling
6. **Request Validation** → Clear validation feedback

**Good APIs are a competitive advantage.** Invest in making them excellent.

---

**Questions?** Contact info@happyhippo.ai
