# Ensure Quality During Fast Deadlines

## Simple Strategy

### Prioritize Features (MoSCoW Method)
- **Must Have**: Critical functionality (can't ship without)
- **Should Have**: Important features (add value)
- **Could Have**: Nice-to-haves (can be deferred)
- **Won't Have**: Remove if needed

### Quality Gates
```javascript
// Minimum quality requirements
const mustHaveQuality = {
  unitTests: 'Critical paths tested',
  security: 'No high-risk issues',
  functionality: 'Core features work',
  errors: 'Basic error handling'
};
```

## Risk Assessment

### Quick Impact vs Effort Check
```javascript
// Focus on high-impact, low-effort fixes
const priorities = [
  'Fix critical bugs',           // High impact, low effort
  'Add input validation',        // High impact, low effort
  'Skip advanced animations',    // Low impact, high effort
  'Defer documentation'          // Low impact, medium effort
];
```

## Testing Strategy

### Automated Testing Pyramid
```javascript
// Unit Tests (Fast, run often)
describe('UserService', () => {
  it('validates email format', () => {
    expect(validateEmail('test@')).toBe(false);
  });
});

// Integration Tests (Medium speed, critical flows)
describe('User API', () => {
  it('creates user successfully', async () => {
    const response = await api.post('/users', validData);
    expect(response.status).toBe(201);
  });
});

// E2E Tests (Slow, only critical paths)
describe('Registration', () => {
  it('completes full flow', () => {
    // Test only the happy path
  });
});
```

### CI/CD Pipeline
```yaml
# Fast quality checks
jobs:
  test:
    steps:
      - run: npm run lint        # Code quality
      - run: npm run test:unit   # Unit tests
      - run: npm run build       # Build check
```

## Code Quality Practices

### Code Review Checklist
```markdown
## Fast Deadline Review
- [ ] Core functionality works
- [ ] No security vulnerabilities
- [ ] Error handling exists
- [ ] Critical paths tested
- [ ] No breaking changes
```

### Pair Programming
```javascript
// Quick pairing sessions (25-45 min)
// Focus on critical code paths
// One writes, one reviews
// Switch roles frequently
```

## Time Management

### Time Boxing
```javascript
// Allocate specific time slots
const timeBoxes = [
  { task: 'Core feature', time: '2 hours', deliverable: 'Working MVP' },
  { task: 'Error handling', time: '1 hour', deliverable: 'Graceful failures' },
  { task: 'Testing', time: '1 hour', deliverable: 'Critical tests pass' }
];
```

## Communication

### Status Updates
```javascript
// Regular stakeholder updates
const update = {
  completed: 'What finished today',
  blockers: 'Any obstacles',
  next: 'Tomorrow's priorities',
  risks: 'Quality concerns or timeline issues'
};
```

## Interview Questions & Answers

**Q: How do you maintain quality during tight deadlines?**
**A:** "I prioritize using MoSCoW method - focus on must-have features and quality gates. I ensure critical paths are tested and core functionality works, while deferring nice-to-haves."

**Q: What gets sacrificed first?**
**A:** "Documentation, advanced features, and comprehensive test coverage. I never sacrifice security, core functionality, or basic error handling."

**Q: How do you handle technical debt?**
**A:** "Document it clearly and schedule cleanup in the next sprint. I add TODO comments and create tickets for follow-up work."

**Q: What's your testing strategy for fast deadlines?**
**A:** "Focus on unit tests for critical logic, integration tests for key APIs, and minimal E2E tests for main user flows. Automate everything possible."

**Q: How do you communicate trade-offs to stakeholders?**
**A:** "Be transparent about what's being deferred and why. Show the impact on quality and timeline, and get explicit approval for compromises."

## Summary
- **Prioritize**: Must-have features first
- **Test**: Critical paths only, automate everything
- **Communicate**: Keep stakeholders informed
- **Document**: Technical debt for later cleanup
- **Balance**: Speed without sacrificing core quality