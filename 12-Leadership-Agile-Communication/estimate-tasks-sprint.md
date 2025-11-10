# Estimate Tasks in Sprint

## Simple Estimation Methods

### Planning Poker
```javascript
// Team members pick cards simultaneously
const cards = [1, 2, 3, 5, 8, 13, 20, '?'];

// Discuss differences, re-vote if needed
// Reach consensus on final estimate
```

### T-Shirt Sizing
```javascript
const sizes = {
  XS: '2-4 hours',    // Simple task
  S: 'Half day',      // Small feature
  M: '1-2 days',      // Medium feature
  L: '2-3 days',      // Large feature
  XL: '3-5 days'      // Complex feature
};
```

### Story Points (Recommended)
```javascript
// Use Fibonacci: 1, 2, 3, 5, 8, 13, 21
const examples = {
  1: 'Add a button to existing page',
  3: 'Create a simple form with validation',
  5: 'Build a new page with basic functionality',
  8: 'Implement user authentication',
  13: 'Build a complex dashboard with charts'
};
```

## Estimation Process

### Story Refinement First
```javascript
// INVEST criteria for good stories
const invest = {
  independent: 'Can be developed separately',
  negotiable: 'Details can be discussed',
  valuable: 'Provides user value',
  estimable: 'Team can estimate effort',
  small: 'Fits in one sprint',
  testable: 'Can be verified'
};
```

### Factors to Consider
```javascript
const factors = [
  'Technical complexity',
  'UI/UX design effort',
  'Testing requirements',
  'Dependencies on other teams',
  'Risk and uncertainty',
  'Team experience with technology'
];
```

## Team Velocity

### Calculate Velocity
```javascript
// Points completed per sprint
const sprintHistory = [
  { sprint: 1, points: 45 },
  { sprint: 2, points: 52 },
  { sprint: 3, points: 48 }
];

const averageVelocity = 48; // points per sprint
```

### Sprint Planning
```javascript
// Don't commit more than velocity
const sprintPlanning = {
  velocity: 48,
  committed: 40,    // Leave buffer for uncertainty
  buffer: 8         // 15-20% buffer recommended
};
```

## Common Mistakes

### ❌ Estimating in Hours
```javascript
// Problems with hour estimates
const hourProblems = [
  'False precision (2.5 hours?)',
  'Doesnt account for meetings/interruptions',
  'Hard to compare across team members',
  'Focus on time instead of complexity'
];
```

### ❌ No Reference Stories
```javascript
// Always have reference points
const referenceStories = {
  small: 'Login form (3 points)',
  medium: 'User profile page (5 points)',
  large: 'Shopping cart (8 points)'
};
```

## Interview Questions & Answers

**Q: How do you estimate tasks in agile?**
**A:** "I use Planning Poker with story points. The team discusses the work, picks cards simultaneously, discusses differences, and reaches consensus. We use Fibonacci sequence (1,2,3,5,8,13) to represent complexity."

**Q: Why story points instead of hours?**
**A:** "Story points focus on relative complexity rather than time. They account for uncertainty, interruptions, and varying team member speeds. Hours create false precision."

**Q: How do you improve estimation accuracy?**
**A:** "Track velocity over sprints, use reference stories for calibration, consider risk factors, and continuously refine based on actual vs estimated work."

**Q: What if estimates are consistently wrong?**
**A:** "Review the process - are we breaking stories small enough? Are we considering all factors? Adjust velocity calculations and add buffers for uncertainty."

**Q: How do you handle large, complex tasks?**
**A:** "Break them down into smaller, estimable stories. If something is truly large (13+ points), consider splitting it across multiple sprints."

**Q: What's the role of velocity in planning?**
**A:** "Velocity shows how many story points the team completes per sprint. We use it to predict capacity and avoid over-committing."

## Summary
- **Planning Poker**: Team consensus technique
- **Story Points**: Relative complexity (1,2,3,5,8,13)
- **Velocity**: Historical delivery capacity
- **Buffer**: Always leave room for uncertainty
- **Refine**: Continuously improve accuracy