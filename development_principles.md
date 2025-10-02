# Core Development Principles

**Universal principles for building quality software**

These principles apply to any technology stack, framework, or language. They represent fundamental approaches to software development that lead to reliable, maintainable systems.

---

## 1. No Mocks, No Fallback Data

### Principle

**Production code should never hide failures with mock data or fallback values.**

When a real dependency fails (API, database, service), the failure must be visible and obvious—not masked by fallback behavior.

### Why This Matters

- **Hidden failures never get fixed** - If your code works with mock data when the API is down, you'll never know the API is unreliable
- **False confidence** - Developers think features work when they actually don't
- **Production surprises** - Mocks don't match reality; users see failures that tests didn't catch
- **Root causes ignored** - Symptoms are masked, underlying problems persist

### The Rule

```
IF real_dependency_fails THEN
    fail_loudly_and_obviously
NOT
    return_mock_data_and_pretend_it_worked
```

### Examples

#### ❌ Bad Practice
```javascript
async function getUserProfile(userId) {
    try {
        return await api.getUser(userId);
    } catch (error) {
        // Silently fall back to mock data
        console.log('API failed, using mock');
        return {
            id: userId,
            name: 'Test User',
            email: 'test@example.com'
        };
    }
}
```

**Problems:**
- API failures invisible to monitoring
- Developers don't know API is broken
- Users get fake data
- Real issues never addressed

#### ✅ Good Practice
```javascript
async function getUserProfile(userId) {
    try {
        return await api.getUser(userId);
    } catch (error) {
        console.error('❌ Failed to fetch user profile:', {
            userId,
            error: error.message,
            timestamp: new Date().toISOString()
        });
        throw new Error(`Unable to load user profile: ${error.message}`);
    }
}
```

**Benefits:**
- API failures immediately visible
- Monitoring alerts trigger
- Developers fix root cause
- Users get honest error message

### When Fallbacks ARE Acceptable

**Cache scenarios with explicit staleness:**
```javascript
async function getUserProfile(userId) {
    try {
        const fresh = await api.getUser(userId);
        await cache.set(userId, fresh);
        return fresh;
    } catch (error) {
        const cached = await cache.get(userId);
        if (cached && cached.timestamp > Date.now() - 300000) { // 5 min
            console.warn('⚠️ Using cached data due to API failure', {
                userId,
                cacheAge: Date.now() - cached.timestamp
            });
            return { ...cached.data, _fromCache: true, _stale: true };
        }
        throw new Error('User profile unavailable');
    }
}
```

**Key differences:**
- User knows data might be stale (`_stale: true`)
- Explicit time limit on staleness
- Warning logged for monitoring
- Still fails if cache too old

---

## 2. Fail Fast, Fail Loud

### Principle

**Make failures obvious and immediate. Don't silently degrade or mask problems.**

When something goes wrong, the failure should be:
- **Fast** - Detected immediately, not after downstream effects
- **Loud** - Visible in logs, monitoring, user feedback
- **Specific** - Clear about what failed and why

### Why This Matters

- **Problems get fixed** - Visible failures get addressed, hidden ones persist
- **Debugging is easier** - Fail at the source, not 10 layers downstream
- **Users get honesty** - "Service unavailable" is better than broken features
- **Monitoring works** - Alerts trigger on real problems

### The Rule

```
IF validation_fails OR dependency_unavailable OR data_invalid THEN
    throw_specific_error_immediately
NOT
    log_warning_and_continue_with_broken_state
```

### Examples

#### ❌ Bad Practice
```javascript
function processOrder(order) {
    if (!order.items || order.items.length === 0) {
        console.log('Warning: No items in order, skipping');
        return { success: true };  // ❌ Silent failure
    }

    if (!order.customerId) {
        console.log('Warning: No customer ID, using anonymous');
        order.customerId = 'ANONYMOUS';  // ❌ Hiding the problem
    }

    return processPayment(order);
}
```

**Problems:**
- Empty orders marked as successful
- Missing customer data silently replaced
- Downstream systems get bad data
- No alerts or monitoring triggers

#### ✅ Good Practice
```javascript
function processOrder(order) {
    // Fail fast on invalid input
    if (!order || typeof order !== 'object') {
        throw new Error('Invalid order: order must be an object');
    }

    if (!order.items || order.items.length === 0) {
        throw new Error('Invalid order: no items provided');
    }

    if (!order.customerId) {
        throw new Error('Invalid order: customerId required');
    }

    if (!order.items.every(item => item.price > 0)) {
        throw new Error('Invalid order: all items must have positive price');
    }

    return processPayment(order);
}
```

**Benefits:**
- Invalid orders rejected immediately
- Clear error messages for each case
- Upstream systems know submission failed
- Monitoring captures validation failures

### Failing Loud in Practice

```javascript
// Log errors prominently
console.error('❌ CRITICAL: Payment processing failed', {
    orderId: order.id,
    customerId: order.customerId,
    amount: order.total,
    error: error.message,
    timestamp: new Date().toISOString()
});

// Emit metrics/events
metrics.increment('payment.failures', {
    errorType: error.code,
    paymentMethod: order.paymentMethod
});

// Alert monitoring systems
alerting.sendAlert('payment-failure', {
    severity: 'high',
    details: error
});
```

---

## 3. Error-First Design

### Principle

**Design error states before implementing happy paths.**

Before writing code for the success case, identify and design for all the ways things can fail.

### Why This Matters

- **Errors are the majority** - In production, most code paths are error handling
- **Users experience errors** - Error states define UX quality more than happy paths
- **Better architecture** - Considering failures upfront leads to more robust design
- **Recovery is possible** - Well-designed errors enable graceful degradation

### The Rule

```
BEFORE writing happy_path:
    1. List all possible failures
    2. Design specific error responses for each
    3. Define recovery strategies
    4. Plan user experience for each error
THEN write happy_path
```

### Examples

#### ❌ Bad Practice (Happy Path First)
```javascript
// Designed for success only
function transferMoney(fromAccount, toAccount, amount) {
    const balance = getBalance(fromAccount);
    deduct(fromAccount, amount);
    add(toAccount, amount);
    return { success: true, newBalance: balance - amount };
}
```

**Problems:**
- What if insufficient funds?
- What if account doesn't exist?
- What if deduct succeeds but add fails?
- What if amount is negative?

#### ✅ Good Practice (Error-First Design)
```javascript
/**
 * Transfer money between accounts
 *
 * Possible failures:
 * - INSUFFICIENT_FUNDS: fromAccount balance < amount
 * - INVALID_ACCOUNT: fromAccount or toAccount doesn't exist
 * - INVALID_AMOUNT: amount <= 0 or not a number
 * - ACCOUNT_LOCKED: either account is frozen/locked
 * - TRANSACTION_FAILED: partial transaction (deduct succeeded, add failed)
 * - NETWORK_ERROR: database/service unavailable
 */
function transferMoney(fromAccount, toAccount, amount) {
    // Validate inputs (fail fast)
    if (!fromAccount || !toAccount) {
        return {
            success: false,
            error: 'INVALID_ACCOUNT',
            message: 'Both accounts required',
            recovery: 'Verify account numbers and try again'
        };
    }

    if (typeof amount !== 'number' || amount <= 0) {
        return {
            success: false,
            error: 'INVALID_AMOUNT',
            message: 'Amount must be positive number',
            recovery: 'Enter a valid amount'
        };
    }

    // Check account status
    const fromStatus = getAccountStatus(fromAccount);
    if (fromStatus.locked) {
        return {
            success: false,
            error: 'ACCOUNT_LOCKED',
            message: 'Source account is locked',
            recovery: 'Contact support to unlock account',
            supportPhone: '1-800-SUPPORT'
        };
    }

    // Check sufficient funds
    const balance = getBalance(fromAccount);
    if (balance < amount) {
        return {
            success: false,
            error: 'INSUFFICIENT_FUNDS',
            message: `Insufficient funds: balance ${balance}, requested ${amount}`,
            recovery: 'Add funds or reduce transfer amount',
            currentBalance: balance,
            shortfall: amount - balance
        };
    }

    // Execute transaction (with rollback capability)
    try {
        const transaction = beginTransaction();

        deduct(fromAccount, amount, transaction);
        add(toAccount, amount, transaction);

        transaction.commit();

        return {
            success: true,
            newBalance: balance - amount,
            transactionId: transaction.id,
            timestamp: new Date().toISOString()
        };

    } catch (error) {
        transaction.rollback();

        return {
            success: false,
            error: 'TRANSACTION_FAILED',
            message: 'Transfer failed, no money moved',
            recovery: 'Try again or contact support if problem persists',
            technicalDetails: error.message
        };
    }
}
```

**Benefits:**
- Every failure mode has specific handling
- Clear error messages guide users
- Recovery paths defined for each error
- Partial transaction prevented with rollback

### Error-First Checklist

Before implementing any feature:

- [ ] List all possible failure modes
- [ ] Define error response for each failure
- [ ] Design user experience for each error state
- [ ] Plan recovery strategy (retry, fallback, manual intervention)
- [ ] Consider partial failure scenarios
- [ ] Define monitoring/alerting for each error type
- [ ] Document error codes and meanings

---

## 4. Explicit Over Implicit

### Principle

**Code should be obvious, not clever. Prefer clear, explicit code over implicit "magic."**

### Why This Matters

- **Maintenance is easier** - Future developers understand intent immediately
- **Fewer bugs** - Explicit code has fewer hidden assumptions
- **Better debugging** - Clear code paths make problems obvious
- **Onboarding is faster** - New team members can contribute quickly

### Examples

#### ❌ Implicit (Clever but Unclear)
```javascript
// What does this do? Have to trace through to understand
const results = data.map(x => x.y || defaults[x.type]?.y ?? fallback(x));
```

#### ✅ Explicit (Clear Intent)
```javascript
// Intent is obvious from reading
const results = data.map(item => {
    // Try item value first
    if (item.y !== null && item.y !== undefined) {
        return item.y;
    }

    // Try type-specific default
    const typeDefault = defaults[item.type];
    if (typeDefault && typeDefault.y !== undefined) {
        return typeDefault.y;
    }

    // Fall back to computed value
    return fallback(item);
});
```

---

## 5. Consistent Error Handling

### Principle

**Use the same error handling patterns throughout your application.**

### Why This Matters

- **Predictable debugging** - Know where to look for errors
- **Consistent logging** - Standard format for all errors
- **Unified monitoring** - All errors flow through same instrumentation
- **Easier testing** - Test error handling once, apply everywhere

### Example Pattern

```javascript
// Standard error handler
function handleError(error, context) {
    // 1. Log with context
    console.error('❌ Error:', {
        message: error.message,
        code: error.code,
        context,
        timestamp: new Date().toISOString(),
        stack: error.stack
    });

    // 2. Report to monitoring
    monitoring.reportError(error, context);

    // 3. Return standard error response
    return {
        success: false,
        error: error.code || 'INTERNAL_ERROR',
        message: error.message,
        timestamp: new Date().toISOString(),
        requestId: context.requestId
    };
}

// Use everywhere
try {
    const result = await riskyOperation();
    return result;
} catch (error) {
    return handleError(error, { operation: 'riskyOperation', userId });
}
```

---

## 6. Developer Experience Matters

### Principle

**Optimize for the humans who will maintain this code.**

### Why This Matters

- **Code is read 10x more than written** - Make reading easy
- **Debugging happens often** - Make debugging easy
- **Teams change** - Make onboarding easy
- **You'll forget** - Make future-you's life easier

### Practical Guidelines

**1. Write error messages for developers:**
```javascript
// ❌ Bad: Cryptic
throw new Error('Invalid input');

// ✅ Good: Specific and helpful
throw new Error(
    `Invalid order: expected object with 'items' array and 'customerId' string, ` +
    `got ${typeof order} with keys: ${Object.keys(order || {}).join(', ')}`
);
```

**2. Add context to logs:**
```javascript
// ❌ Bad: No context
console.log('Processing payment');

// ✅ Good: Useful context
console.log('Processing payment:', {
    orderId: order.id,
    amount: order.total,
    customerId: order.customerId,
    paymentMethod: order.paymentMethod
});
```

**3. Document non-obvious decisions:**
```javascript
// ✅ Good: Explain why
// Using 5-minute cache TTL because:
// - User data changes infrequently
// - API has rate limit of 100 req/min
// - 5min = good balance of freshness vs API load
const CACHE_TTL = 300000;
```

**4. Make debugging easy:**
```javascript
// Add request IDs to everything
const requestId = generateRequestId();
console.log('Starting operation:', { requestId, userId });
try {
    const result = await operation({ requestId });
    console.log('Operation succeeded:', { requestId, result });
    return result;
} catch (error) {
    console.error('Operation failed:', { requestId, error });
    throw error;
}
```

---

## Summary

These six principles work together to create robust, maintainable software:

1. **No Mocks, No Fallback Data** → Failures are visible and get fixed
2. **Fail Fast, Fail Loud** → Problems are obvious and immediate
3. **Error-First Design** → Error states are well-designed and recoverable
4. **Explicit Over Implicit** → Code is clear and maintainable
5. **Consistent Error Handling** → Debugging and monitoring are predictable
6. **Developer Experience Matters** → Future maintainers (including you) say thank you

**Apply these principles from day one.** They're easier to implement upfront than to retrofit later.

---

**Questions?** Contact info@happyhippo.ai
