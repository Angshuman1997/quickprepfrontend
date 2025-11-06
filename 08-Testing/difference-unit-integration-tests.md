# Difference between unit and integration tests

## Question
Difference between unit and integration tests.

# Difference between unit and integration tests

## Question
Difference between unit and integration tests.

## Answer

Unit tests and integration tests are two fundamental types of software testing that serve different purposes in the testing pyramid. Understanding their differences is crucial for building robust test suites.

## Unit Tests

### Definition
Unit tests test individual units or components of code in isolation, typically a single function, method, or class.

### Characteristics
- **Isolation**: Test one unit at a time with mocked dependencies
- **Fast**: Execute quickly (milliseconds)
- **Deterministic**: Same input always produces same output
- **Focused**: Test specific functionality without external dependencies

### Example
```javascript
// math.js
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;
export const divide = (a, b) => {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
};
```

```javascript
// math.test.js
import { add, multiply, divide } from './math';

describe('Math functions', () => {
  describe('add', () => {
    it('should add two positive numbers', () => {
      expect(add(2, 3)).toBe(5);
    });

    it('should add negative numbers', () => {
      expect(add(-2, -3)).toBe(-5);
    });

    it('should handle zero', () => {
      expect(add(0, 5)).toBe(5);
    });
  });

  describe('divide', () => {
    it('should divide two numbers', () => {
      expect(divide(10, 2)).toBe(5);
    });

    it('should throw error for division by zero', () => {
      expect(() => divide(10, 0)).toThrow('Division by zero');
    });
  });
});
```

## Integration Tests

### Definition
Integration tests verify that different units work together correctly, testing the interaction between components, modules, or services.

### Characteristics
- **Interaction**: Test how units work together
- **Slower**: Take longer to execute (seconds)
- **Complex**: May involve databases, APIs, file systems
- **Real dependencies**: Use actual implementations (not mocks)

### Example
```javascript
// userService.js
export class UserService {
  constructor(database) {
    this.database = database;
  }

  async createUser(userData) {
    // Validate user data
    if (!userData.email || !userData.password) {
      throw new Error('Email and password required');
    }

    // Hash password
    const hashedPassword = await this.hashPassword(userData.password);

    // Save to database
    const user = await this.database.save({
      ...userData,
      password: hashedPassword,
      createdAt: new Date()
    });

    return user;
  }

  async hashPassword(password) {
    // Password hashing logic
    return `hashed_${password}`;
  }
}
```

```javascript
// userService.integration.test.js
import { UserService } from './userService';
import { MockDatabase } from '../mocks/database';

describe('UserService Integration', () => {
  let userService;
  let mockDatabase;

  beforeEach(() => {
    mockDatabase = new MockDatabase();
    userService = new UserService(mockDatabase);
  });

  describe('createUser', () => {
    it('should create user with valid data', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User'
      };

      const result = await userService.createUser(userData);

      expect(result).toHaveProperty('id');
      expect(result.email).toBe(userData.email);
      expect(result.password).not.toBe(userData.password); // Should be hashed
      expect(result).toHaveProperty('createdAt');
    });

    it('should throw error for missing email', async () => {
      const userData = { password: 'password123' };

      await expect(userService.createUser(userData))
        .rejects.toThrow('Email and password required');
    });

    it('should save user to database', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'password123'
      };

      await userService.createUser(userData);

      const savedUsers = mockDatabase.getAll('users');
      expect(savedUsers).toHaveLength(1);
      expect(savedUsers[0].email).toBe(userData.email);
    });
  });
});
```

## Key Differences

| Aspect | Unit Tests | Integration Tests |
|--------|------------|-------------------|
| **Scope** | Single function/method/class | Multiple components working together |
| **Dependencies** | Mocked/stubbed | Real implementations |
| **Speed** | Fast (ms) | Slower (seconds) |
| **Setup** | Minimal | Complex (databases, services) |
| **Reliability** | High (deterministic) | Lower (external factors) |
| **Debugging** | Easy (isolated) | Harder (multiple components) |
| **Coverage** | Code logic | System behavior |
| **Cost** | Low | Higher |

## Testing Pyramid

```
End-to-End Tests (Slow, Expensive)
    ↕️
Integration Tests (Medium)
    ↕️
Unit Tests (Fast, Cheap)
```

### Recommended Distribution
- **Unit Tests**: 70% of test suite
- **Integration Tests**: 20% of test suite
- **End-to-End Tests**: 10% of test suite

## When to Use Each

### Unit Tests For
- **Business logic**: Complex calculations, algorithms
- **Utility functions**: String manipulation, data transformation
- **Pure functions**: Functions with no side effects
- **Error handling**: Exception scenarios
- **Edge cases**: Boundary conditions

### Integration Tests For
- **API endpoints**: HTTP request/response handling
- **Database operations**: CRUD operations with real DB
- **Service interactions**: How services communicate
- **Component integration**: React components with stores/APIs
- **Workflows**: Multi-step business processes

## Advanced Patterns

### Unit Test with Mocks
```javascript
// Using Jest mocks
import { UserService } from './userService';

jest.mock('./database');
jest.mock('./passwordHasher');

describe('UserService Unit Tests', () => {
  let userService;
  let mockDatabase;
  let mockPasswordHasher;

  beforeEach(() => {
    mockDatabase = {
      save: jest.fn()
    };
    mockPasswordHasher = {
      hash: jest.fn().mockResolvedValue('hashed_password')
    };

    userService = new UserService(mockDatabase, mockPasswordHasher);
  });

  it('should hash password before saving', async () => {
    const userData = { email: 'test@test.com', password: '123' };

    await userService.createUser(userData);

    expect(mockPasswordHasher.hash).toHaveBeenCalledWith('123');
    expect(mockDatabase.save).toHaveBeenCalledWith(
      expect.objectContaining({
        email: 'test@test.com',
        password: 'hashed_password'
      })
    );
  });
});
```

### Integration Test with Real Dependencies
```javascript
// Using test database
import { UserService } from './userService';
import { createTestDatabase } from '../test-utils/database';

describe('UserService Integration Tests', () => {
  let userService;
  let testDatabase;

  beforeAll(async () => {
    testDatabase = await createTestDatabase();
    userService = new UserService(testDatabase);
  });

  afterAll(async () => {
    await testDatabase.cleanup();
  });

  beforeEach(async () => {
    await testDatabase.reset();
  });

  it('should persist user to database', async () => {
    const userData = {
      email: 'integration@test.com',
      password: 'testpass'
    };

    const user = await userService.createUser(userData);

    // Verify in database
    const savedUser = await testDatabase.findUserById(user.id);
    expect(savedUser.email).toBe(userData.email);
    expect(savedUser.password).not.toBe(userData.password);
  });
});
```

## Best Practices

### Unit Tests
- **Arrange-Act-Assert**: Clear test structure
- **One assertion per test**: Focused tests
- **Descriptive names**: What the test verifies
- **Fast execution**: No external dependencies
- **Independent**: No order dependencies

### Integration Tests
- **Realistic data**: Use production-like data
- **Cleanup**: Reset state between tests
- **Timeouts**: Handle async operations
- **Error scenarios**: Test failure conditions
- **Performance**: Monitor execution time

## Common Mistakes

### Unit Tests
- **Testing implementation**: Test behavior, not code structure
- **Over-mocking**: Don't mock everything
- **Brittle tests**: Tests that break with refactoring
- **Missing edge cases**: Only happy path testing

### Integration Tests
- **Slow tests**: Too many slow tests in suite
- **Flaky tests**: Tests that fail intermittently
- **No isolation**: Tests affecting each other
- **Over-testing**: Testing same thing multiple times

## Interview Tips
- **Unit tests**: Test individual units in isolation with mocks
- **Integration tests**: Test how units work together with real dependencies
- **Speed**: Unit tests are fast, integration tests are slower
- **Reliability**: Unit tests are more reliable, integration tests can be flaky
- **Coverage**: Unit tests cover logic, integration tests cover workflows
- **Pyramid**: More unit tests, fewer integration tests
- **When to use**: Unit for logic, integration for interactions