# Handle Disagreements in Code Reviews

## Constructive Communication

### "Yes, And..." Technique
```javascript
// Instead of: "This is wrong"
// Use: "I see your approach, and here's another consideration..."

const goodFeedback = {
  acknowledge: "I understand you chose this for performance",
  add: "and have you considered readability?",
  collaborate: "What do you think about this alternative?"
};
```

### Feedback Sandwich
```javascript
// Positive → Constructive → Positive
const sandwich = {
  positive: "Error handling is well implemented",
  constructive: "Consider extracting to a custom hook",
  positive: "Component structure is clean"
};
```

## Resolution Strategies

### Step-by-Step Process
```javascript
const resolutionSteps = [
  "Ask questions to understand their perspective",
  "Explain your reasoning clearly",
  "Find common ground on goals",
  "Brainstorm alternatives together",
  "Make decision based on agreed criteria"
];
```

### Decision Frameworks
```javascript
const frameworks = {
  teamStandards: "Follow established coding guidelines",
  dataDriven: "Measure performance, run benchmarks",
  consensus: "Discuss until team agrees",
  compromise: "Combine best of both approaches"
};
```

## Common Disagreement Types

### Technical Choices
```javascript
// Example: Redux vs Context API
const discussion = {
  approach1: "Redux - predictable state management",
  approach2: "Context - simpler for this use case",
  resolution: "Use Context for component state, Redux for app state"
};
```

### Implementation Style
```javascript
// Example: Functional vs Class components
const discussion = {
  approach1: "Class components - familiar pattern",
  approach2: "Functional + hooks - modern React",
  resolution: "Use functional components for new code"
};
```

### Scope and Size
```javascript
// Example: Large PR vs multiple small PRs
const discussion = {
  approach1: "One big PR - shows complete feature",
  approach2: "Multiple small PRs - easier to review",
  resolution: "Break down large features into smaller PRs"
};
```

## When to Escalate

### Escalation Triggers
```javascript
const escalateWhen = [
  "Blocks progress for days",
  "Violates company standards",
  "Affects multiple teams",
  "Same disagreement repeats"
];
```

### Escalation Process
```javascript
const escalation = {
  level1: "Author + Reviewer discussion",
  level2: "Include team lead",
  level3: "Architecture review board"
};
```

## Prevention Strategies

### Team Guidelines
```markdown
## Code Review Guidelines
- Focus on code, not person
- Provide specific, actionable feedback
- Suggest alternatives with reasoning
- Acknowledge good practices
- Use questions instead of commands
```

### Regular Calibration
```javascript
// Monthly sessions to align on standards
const calibration = [
  "Review recent disagreements",
  "Update coding guidelines",
  "Share best practices",
  "Discuss emerging patterns"
];
```

## Interview Questions & Answers

**Q: How do you handle disagreements in code reviews?**
**A:** "I focus on constructive communication using the 'Yes, And...' technique. I ask questions to understand their perspective, explain my reasoning, and work together to find the best solution based on our shared goals."

**Q: What's your approach to giving feedback?**
**A:** "I use the feedback sandwich - start with positive, provide constructive feedback, end with positive. I focus on the code and suggest alternatives with reasoning rather than saying something is 'wrong'."

**Q: When do you escalate a disagreement?**
**A:** "When it blocks progress, violates team standards, or affects multiple systems. I follow our escalation process: first try to resolve 1-on-1, then involve team lead, finally architecture board if needed."

**Q: How do you prevent disagreements?**
**A:** "We maintain clear coding guidelines, have regular calibration sessions to align on standards, and encourage ongoing discussion about best practices."

**Q: What's the most important thing in code review disagreements?**
**A:** "Maintaining positive relationships while improving code quality. The goal is better code and team learning, not 'winning' the argument."

## Summary
- **Communicate constructively** - Focus on code, not person
- **Seek understanding** - Ask questions, listen actively
- **Find common ground** - Identify shared goals
- **Escalate appropriately** - Know when to involve others
- **Learn continuously** - Use disagreements as growth opportunities