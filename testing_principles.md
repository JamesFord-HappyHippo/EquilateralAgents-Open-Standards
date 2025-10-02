# Testing Principles

**Fundamental approaches to effective testing**

These principles apply regardless of testing framework, language, or methodology. Focus on testing practices that catch real bugs and build confidence in your code.

---

## 1. Test Error States First

### Principle

**Prioritize testing failure scenarios over happy paths.**

In production, most code execution goes through error paths, not success paths. Your tests should reflect this reality.

### Why This Matters

- **Errors are majority of paths** - More code handles errors than success
- **Errors impact users most** - Users remember failures, not smooth experiences
- **Untested errors fail in production** - Error paths often have bugs because they're rarely tested
- **Recovery depends on errors** - Good error handling enables graceful degradation

### The Rule

```
FOR each function/endpoint/feature:
    1. List all possible error conditions
    2. Write tests for each error condition
    3. Verify specific error messages/codes
    4. Test error recovery/cleanup
    THEN test happy path
```

### Examples

#### ❌ Bad: Only Test Happy Path
```javascript
describe('transferMoney', () => {
    it('should transfer money between accounts', () => {
        const result = transferMoney('account1', 'account2', 100);
        expect(result.success).toBe(true);
    });
});
```

**Problems:**
- What if insufficient funds?
- What if account doesn't exist?
- What if amount is negative?
- What if transfer partially fails?

#### ✅ Good: Error States First
```javascript
describe('transferMoney', () => {
    // Error states tested first
    describe('error conditions', () => {
        it('should reject transfer with insufficient funds', () => {
            const result = transferMoney('account1', 'account2', 9999999);
            expect(result.success).toBe(false);
            expect(result.error.code).toBe('INSUFFICIENT_FUNDS');
            expect(result.error.message).toContain('Insufficient funds');
            expect(result.error.details.shortfall).toBe(9999899); // 9999999 - 100 balance
        });

        it('should reject transfer to non-existent account', () => {
            const result = transferMoney('account1', 'invalid', 100);
            expect(result.success).toBe(false);
            expect(result.error.code).toBe('INVALID_ACCOUNT');
        });

        it('should reject negative amounts', () => {
            const result = transferMoney('account1', 'account2', -100);
            expect(result.success).toBe(false);
            expect(result.error.code).toBe('INVALID_AMOUNT');
        });

        it('should rollback on partial failure', async () => {
            // Simulate: deduct succeeds, add fails
            mockAdd.mockRejectedValue(new Error('Network error'));

            const initialBalance1 = getBalance('account1');
            const initialBalance2 = getBalance('account2');

            const result = await transferMoney('account1', 'account2', 100);

            expect(result.success).toBe(false);
            expect(result.error.code).toBe('TRANSACTION_FAILED');

            // Verify rollback: balances unchanged
            expect(getBalance('account1')).toBe(initialBalance1);
            expect(getBalance('account2')).toBe(initialBalance2);
        });

        it('should handle locked accounts', () => {
            lockAccount('account1');
            const result = transferMoney('account1', 'account2', 100);
            expect(result.success).toBe(false);
            expect(result.error.code).toBe('ACCOUNT_LOCKED');
        });
    });

    // Happy path tested last
    describe('success cases', () => {
        it('should successfully transfer money', () => {
            const result = transferMoney('account1', 'account2', 100);
            expect(result.success).toBe(true);
            expect(result.transactionId).toBeDefined();
        });
    });
});
```

**Benefits:**
- All error conditions verified
- Specific error codes tested
- Rollback/cleanup behavior validated
- Confident error handling works

---

## 2. No Mocks in Integration Tests

### Principle

**Integration tests should test against real dependencies, not mocks.**

Mocks test that your code calls mocks correctly. Real integration tests verify your code works with actual services.

### Why This Matters

- **Mocks lie** - Mock behavior diverges from real services
- **Integration bugs missed** - Real API changes break your code, mocks don't
- **False confidence** - Tests pass with mocks, fail in production
- **Real contracts** - Only real services verify correct integration

### The Rule

```
Unit Tests:     Mock external dependencies
Integration:    Real databases, real APIs, real services
E2E:           Everything real, including UI
```

### Examples

#### ❌ Bad: Mocked "Integration" Test
```javascript
// This is actually a unit test with mocks
describe('User Registration Integration', () => {
    it('should register user and send email', async () => {
        // Mock database
        mockDatabase.insert.mockResolvedValue({ userId: 'user_123' });

        // Mock email service
        mockEmailService.send.mockResolvedValue({ sent: true });

        const result = await registerUser({
            email: 'test@example.com',
            password: 'password123'
        });

        expect(result.success).toBe(true);
        expect(mockDatabase.insert).toHaveBeenCalled();
        expect(mockEmailService.send).toHaveBeenCalled();
    });
});
```

**Problems:**
- Doesn't test database actually stores user
- Doesn't test email actually sends
- Doesn't catch real integration failures
- Just tests that mocks were called

#### ✅ Good: Real Integration Test
```javascript
// Real integration test with actual services
describe('User Registration Integration', () => {
    let testDatabase;
    let testEmailServer;

    beforeAll(async () => {
        // Use real test database
        testDatabase = await createTestDatabase();

        // Use real email testing service (e.g., MailHog, Ethereal)
        testEmailServer = await startTestEmailServer();
    });

    afterEach(async () => {
        // Clean up test data
        await testDatabase.clear();
        await testEmailServer.clearEmails();
    });

    it('should register user in database and send confirmation email', async () => {
        const userEmail = 'test@example.com';

        // Actually register user
        const result = await registerUser({
            email: userEmail,
            password: 'password123'
        });

        expect(result.success).toBe(true);
        expect(result.userId).toBeDefined();

        // Verify user actually in database
        const userInDb = await testDatabase.query(
            'SELECT * FROM users WHERE email = ?',
            [userEmail]
        );
        expect(userInDb).toBeDefined();
        expect(userInDb.email).toBe(userEmail);
        expect(userInDb.passwordHash).toBeDefined();
        expect(userInDb.passwordHash).not.toBe('password123'); // Hashed

        // Verify email actually sent
        const emails = await testEmailServer.getEmails();
        expect(emails.length).toBe(1);
        expect(emails[0].to).toBe(userEmail);
        expect(emails[0].subject).toContain('Confirm');
        expect(emails[0].body).toContain('confirmation');
    });

    it('should handle duplicate email registration', async () => {
        // Register user first time
        await registerUser({
            email: 'test@example.com',
            password: 'password123'
        });

        // Try to register again with same email
        const result = await registerUser({
            email: 'test@example.com',
            password: 'differentpass'
        });

        expect(result.success).toBe(false);
        expect(result.error.code).toBe('DUPLICATE_EMAIL');

        // Verify only one user in database
        const users = await testDatabase.query(
            'SELECT * FROM users WHERE email = ?',
            ['test@example.com']
        );
        expect(users.length).toBe(1); // Real database enforces constraint
    });
});
```

**Benefits:**
- Tests real database constraints
- Tests real email sending
- Catches actual integration bugs
- Verifies end-to-end flow works

### When Mocks ARE Acceptable

**External services outside your control:**
```javascript
// OK to mock: Third-party payment gateway (expensive, rate-limited)
describe('Payment Processing', () => {
    it('should handle payment gateway timeout', async () => {
        mockPaymentGateway.charge.mockRejectedValue(new Error('Timeout'));

        const result = await processPayment({ amount: 100 });

        expect(result.success).toBe(false);
        expect(result.error.code).toBe('PAYMENT_TIMEOUT');
    });
});
```

**But also have real integration tests:**
```javascript
// Also test against real payment sandbox
describe('Payment Integration (Sandbox)', () => {
    it('should process real payment in sandbox', async () => {
        const result = await processPayment({
            amount: 100,
            cardNumber: '4111111111111111', // Test card
            environment: 'sandbox'
        });

        expect(result.success).toBe(true);
        expect(result.transactionId).toBeDefined();
    });
});
```

---

## 3. Test Data Coherence

### Principle

**Test data should reflect real-world relationships and constraints.**

Test data isn't just random values—it should represent realistic scenarios with proper relationships.

### Why This Matters

- **Catches real bugs** - Unrealistic test data misses real problems
- **Validates constraints** - Real data tests foreign keys, validations
- **Represents reality** - Tests real user scenarios
- **Better coverage** - Complex relationships reveal edge cases

### Examples

#### ❌ Bad: Incoherent Test Data
```javascript
const testOrder = {
    orderId: 'order_123',
    customerId: 'customer_999',  // Customer doesn't exist
    items: [
        {
            productId: 'product_abc',  // Product doesn't exist
            quantity: -5,               // Negative quantity
            price: 0                    // Free product?
        }
    ],
    total: 1000,  // Doesn't match items total
    createdAt: '2025-01-01',
    updatedAt: '2020-01-01'  // Updated before created?
};
```

**Problems:**
- Invalid references (customer, product don't exist)
- Business logic violations (negative quantity, zero price)
- Data inconsistencies (total doesn't match, dates reversed)
- Doesn't test real constraints

#### ✅ Good: Coherent Test Data
```javascript
describe('Order Processing', () => {
    let testCustomer;
    let testProduct;

    beforeEach(async () => {
        // Create customer first
        testCustomer = await createTestCustomer({
            customerId: 'customer_test1',
            email: 'test@example.com',
            name: 'Test User',
            billingAddress: {
                street: '123 Test St',
                city: 'Testville',
                country: 'US',
                postalCode: '12345'
            }
        });

        // Create product with inventory
        testProduct = await createTestProduct({
            productId: 'product_test1',
            name: 'Test Widget',
            price: 29.99,
            inventory: 100,
            isActive: true
        });
    });

    it('should process valid order', async () => {
        const order = await createOrder({
            customerId: testCustomer.customerId,  // References real customer
            items: [
                {
                    productId: testProduct.productId,  // References real product
                    quantity: 2,                        // Positive quantity
                    price: testProduct.price            // Matches product price
                }
            ]
            // Total calculated from items
        });

        expect(order.success).toBe(true);
        expect(order.total).toBe(59.98); // 2 × 29.99
        expect(order.createdAt).toBeDefined();
        expect(new Date(order.updatedAt)).toBeGreaterThanOrEqual(
            new Date(order.createdAt)
        );

        // Verify inventory decreased
        const product = await getProduct(testProduct.productId);
        expect(product.inventory).toBe(98); // 100 - 2
    });

    it('should reject order when insufficient inventory', async () => {
        const order = await createOrder({
            customerId: testCustomer.customerId,
            items: [
                {
                    productId: testProduct.productId,
                    quantity: 101,  // More than inventory (100)
                    price: testProduct.price
                }
            ]
        });

        expect(order.success).toBe(false);
        expect(order.error.code).toBe('INSUFFICIENT_INVENTORY');

        // Verify inventory unchanged
        const product = await getProduct(testProduct.productId);
        expect(product.inventory).toBe(100);
    });
});
```

**Benefits:**
- Tests real database constraints (foreign keys)
- Tests business logic (inventory management)
- Data relationships are realistic
- Catches bugs mocked tests would miss

---

## 4. Failure Scenario Coverage

### Principle

**Explicitly test what happens when things go wrong.**

Don't just test that features work—test what happens when dependencies fail, networks timeout, or resources are exhausted.

### Failure Scenarios to Test

1. **Dependency failures**
   - Database down
   - API unavailable
   - Cache miss
   - Queue full

2. **Network issues**
   - Timeouts
   - Connection drops
   - Slow responses
   - Partial data

3. **Resource exhaustion**
   - Out of memory
   - Disk full
   - Connection pool exhausted
   - Rate limits exceeded

4. **Data issues**
   - Corrupt data
   - Missing data
   - Invalid format
   - Partial records

### Examples

```javascript
describe('User Profile Retrieval - Failure Scenarios', () => {
    it('should handle database timeout', async () => {
        mockDatabase.query.mockImplementation(() =>
            new Promise((_, reject) =>
                setTimeout(() => reject(new Error('Timeout')), 5000)
            )
        );

        const result = await getUserProfile('user_123');

        expect(result.success).toBe(false);
        expect(result.error.code).toBe('DATABASE_TIMEOUT');
    });

    it('should handle cache failure gracefully', async () => {
        mockCache.get.mockRejectedValue(new Error('Cache unavailable'));

        // Should fall back to database
        const result = await getUserProfile('user_123');

        expect(result.success).toBe(true); // Still works
        expect(result.data._fromCache).toBe(false);
        expect(mockDatabase.query).toHaveBeenCalled(); // Fell back to DB
    });

    it('should handle partial data corruption', async () => {
        mockDatabase.query.mockResolvedValue({
            userId: 'user_123',
            email: 'test@example.com',
            // profile is corrupted/missing
            profile: null
        });

        const result = await getUserProfile('user_123');

        // Should handle missing profile gracefully
        expect(result.success).toBe(true);
        expect(result.data.profile).toBeDefined();
        expect(result.data.profile.isComplete).toBe(false);
    });
});
```

---

## 5. Performance Baseline Testing

### Principle

**Establish and monitor performance baselines for critical operations.**

Know what "normal" performance looks like so you can detect degradation.

### Why This Matters

- **Catch regressions** - Performance degradation detected early
- **Set expectations** - Know if performance is acceptable
- **Guide optimization** - Data-driven performance decisions
- **SLA compliance** - Verify meeting performance requirements

### Examples

```javascript
describe('Performance Baselines', () => {
    it('should load user profile in < 100ms', async () => {
        const start = Date.now();

        await getUserProfile('user_123');

        const duration = Date.now() - start;
        expect(duration).toBeLessThan(100);
    });

    it('should handle 100 concurrent requests in < 5s', async () => {
        const start = Date.now();

        const requests = Array.from({ length: 100 }, (_, i) =>
            getUserProfile(`user_${i}`)
        );

        await Promise.all(requests);

        const duration = Date.now() - start;
        expect(duration).toBeLessThan(5000);
    });

    it('should process batch of 1000 items in < 30s', async () => {
        const items = Array.from({ length: 1000 }, (_, i) => ({
            id: `item_${i}`,
            data: 'test'
        }));

        const start = Date.now();

        await processBatch(items);

        const duration = Date.now() - start;
        expect(duration).toBeLessThan(30000);
    });
});
```

### Performance Testing Best Practices

1. **Test with realistic data volume**
   - Small dataset: 100 records
   - Medium dataset: 10,000 records
   - Large dataset: 100,000+ records

2. **Test concurrent load**
   - Sequential (baseline)
   - 10 concurrent users
   - 100 concurrent users
   - 1000 concurrent users

3. **Monitor resource usage**
   - Memory consumption
   - CPU usage
   - Database connections
   - Network throughput

---

## Summary

Apply these five principles for effective testing:

1. **Test Error States First** → Most code execution is error handling
2. **No Mocks in Integration Tests** → Real services catch real bugs
3. **Test Data Coherence** → Realistic data reveals real problems
4. **Failure Scenario Coverage** → Test what happens when things break
5. **Performance Baseline Testing** → Know what "normal" looks like

**Good tests give confidence.** Invest in testing practices that catch real bugs.

---

**Questions?** Contact info@happyhippo.ai
