# Difference between unit and integration tests

## Simple Answer
Unit tests test individual functions or components in isolation, while integration tests verify that different parts of your application work together correctly.

## Key Differences

| Aspect | Unit Tests | Integration Tests |
|--------|------------|-------------------|
| **What they test** | Single function/method | Multiple components working together |
| **Dependencies** | Mocked/faked | Real implementations |
| **Speed** | Fast (milliseconds) | Slower (seconds) |
| **Setup** | Simple | Complex (may need DB, APIs) |
| **Reliability** | Very reliable | Can be flaky due to external factors |

## Unit Tests Example

```javascript
// Function to test
function add(a, b) {
  return a + b;
}

// Unit test - isolated, no dependencies
describe('add function', () => {
  it('adds two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('handles negatives', () => {
    expect(add(-1, 1)).toBe(0);
  });
});
```

## Integration Tests Example

```javascript
// User service that depends on database
class UserService {
  constructor(database) {
    this.database = database;
  }

  async createUser(userData) {
    // Validate and save to database
    const user = await this.database.save(userData);
    return user;
  }
}

// Integration test - tests real interaction
describe('UserService', () => {
  it('creates user in database', async () => {
    const mockDb = new MockDatabase();
    const service = new UserService(mockDb);

    const user = await service.createUser({
      email: 'test@example.com',
      name: 'Test User'
    });

    expect(user.id).toBeDefined();
    expect(mockDb.getAllUsers()).toHaveLength(1);
  });
});
```

## Testing Pyramid

```
E2E Tests (Slow, few)
    ↕️
Integration Tests (Medium)
    ↕️
Unit Tests (Fast, many)
```

**Recommended ratio**: 70% unit, 20% integration, 10% end-to-end

## When to Use Each

### Unit Tests For:
- Pure functions (no side effects)
- Business logic and calculations
- Error handling
- Edge cases

### Integration Tests For:
- API calls and database operations
- Component interactions
- Service communications
- Complete workflows

## Interview Q&A

**Q: Why are unit tests faster than integration tests?**
A: Unit tests run in isolation with mocked dependencies, while integration tests use real databases, APIs, or services which take time to set up and execute.

**Q: When should you use mocks in testing?**
A: Use mocks in unit tests to isolate the code you're testing. In integration tests, you typically want real dependencies to test actual interactions.

**Q: What's the testing pyramid and why is it important?**
A: The pyramid shows you should have many fast unit tests, fewer integration tests, and very few slow end-to-end tests. This gives you quick feedback and reliable tests.

**Q: Can you give an example of something you'd unit test vs integration test?**
A: I'd unit test a function that calculates taxes (pure logic), but integration test the complete checkout process that saves to database and sends confirmation email.

## Best Practices
- **Unit tests**: One assertion per test, descriptive names, test edge cases
- **Integration tests**: Use realistic data, clean up after tests, handle async operations
- **Both**: Write tests before or alongside code, run them frequently</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\difference-unit-integration-tests-simple.md