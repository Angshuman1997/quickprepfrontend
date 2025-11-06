# Capgemini Interview Preparation - Complete Question Bank

A comprehensive collection of 143+ interview questions covering JavaScript, React, Next.js, TypeScript, State Management, Performance, Testing, and more.

## 📁 Section Overview

- **[00-JavaScript](#00-javascript)** - 16 questions
- **[01-React-Core-Advanced](#01-react-core-advanced)** - 23 questions  
- **[02-NextJS-App-Router](#02-nextjs-app-router)** - 22 questions
- **[03-TypeScript-Practical](#03-typescript-practical)** - 11 questions
- **[04-State-Management](#04-state-management)** - 18 questions
- **[05-Performance-Optimization](#05-performance-optimization)** - 13 questions
- **[06-Micro-Frontend-Architecture](#06-micro-frontend-architecture)** - 7 questions
- **[07-Bundlers](#07-bundlers)** - 5 questions
- **[08-Testing](#08-testing)** - 7 questions
- **[09-HTML-CSS-UI](#09-html-css-ui)** - 7 questions
- **[10-Leadership-Agile-Communication](#10-leadership-agile-communication)** - 4 questions
- **[11-GenAI-Agents](#11-genai-agents)** - 10 questions

---

## 00-JavaScript

### Core Concepts
1. What is hoisting? How do var, let, and const differ during hoisting?
2. What is the Temporal Dead Zone?
3. What is a closure? Where do you use closures in React?

### Event Loop & Asynchronous Programming
4. In JavaScript's event loop, which executes first: microtasks or macrotasks?
5. What will be the output order in this case: Promise vs setTimeout?
6. What is async/await? How does it relate to Promises and Promise chaining?
7. What are callbacks and callback hell? How do you solve callback hell?

### Functions & Advanced Concepts
8. What are the different types of functions in JavaScript?
9. What is a higher-order function? Provide examples.
10. Explain call, apply, and bind methods. What are prototypes?

### Data Structures & Algorithms
11. What is hashing? Explain with JavaScript examples.
12. Common array manipulation methods and techniques
13. Common string manipulation methods and techniques

### Coding Challenges
14. Write code to count the frequency of elements in an array.
15. Write code to return the shortest word length in a sentence.
16. Write code to merge two arrays and return a unique sorted result.

---

## 01-React-Core-Advanced

### Core Concepts
1. What is the Virtual DOM and why does React use it?
2. Explain Reconciliation and how React updates the UI efficiently.
3. What is React Fiber? What problem did it solve?
4. What does JSX compile to under the hood?

### State & Props Management
5. Difference between state, props, and context.
6. Controlled vs Uncontrolled components — when do you use each?
7. When should you use useRef instead of state?

### Component Patterns
8. What is forwardRef and when would you use it?
9. What is a Higher-Order Component (HOC)? Why are hooks preferred now?
10. What are Portals? Practical example.
11. What is an Error Boundary? Can it catch errors inside event handlers?
12. How to handle errors in React (simple and direct)

### Hooks & Lifecycle
12. Explain the lifecycle of useEffect, including cleanup.
13. Difference between useCallback and useMemo.
14. How to create custom hooks for data fetching?
15. Custom hooks for form handling - best practices
16. When to extract logic into custom hooks?

### Performance & Optimization
17. What is React.memo? When does it NOT help?
18. What causes unnecessary re-renders? How do you detect & fix them?
19. How do you optimize large lists in React?
20. How to show loading spinners during API calls?

### Architecture & Best Practices
21. How do you manage deeply nested component communication?
22. How do you structure large component files to keep them maintainable?

---

## 02-NextJS-App-Router

### App Router Fundamentals
1. Difference between Pages Router and App Router.
2. What are Server Components vs Client Components?
3. When must we write "use client"?

### File Conventions
4. Purpose of layout.js
5. Purpose of page.js
6. Purpose of loading.js
7. Purpose of error.js
8. Purpose of template.js
9. How Next.js handles errors using error.jsx
10. How Next.js handles loading states using loading.jsx
11. If the dashboard has nested folders
12. Which errors are not handled by error.jsx and loading.jsx

### Data Fetching & Rendering
13. How is data fetched in App Router without getServerSideProps?
14. Explain SSR vs SSG vs ISR.
15. How does revalidation work in Next.js?
16. What is fetch({ cache: 'no-store' }) used for?

### API & Server Features
17. How do Route Handlers work? (app/api/*/route.js)
18. How do Server Actions work? When do you use them?
19. How to stream API responses to UI in Next.js 13+/14?
20. How do you handle authentication in Server Components?

### Performance & Optimization
21. How does Next.js reduce bundle size using Server Components?
22. How do dynamic routes work in App Router?

---

## 03-TypeScript-Practical

### Basic Types & Interfaces
1. Difference between type and interface.
2. How to extend an interface?
3. Explain Generics with a simple custom hook example.

### Utility Types
4. Use cases for Partial<T>
5. Use cases for Pick<T>
6. Use cases for Omit<T>
7. Use cases for Record<K, T>
8. Use cases for ReturnType<T>

### React & API Integration
9. How do you type React component props?
10. How do you type async functions and server actions?
11. How do you define a reusable type for API responses?

---

## 04-State-Management

### Context vs Redux
1. When do you prefer Context API over Redux?
2. When would you choose Zustand instead of Redux?

### Redux Toolkit Basics
3. What problem does Redux Toolkit solve?
4. What does createSlice() do?
5. How to handle async API calls using createAsyncThunk?
6. What is RTK Query? When is it better than Thunks?

### Advanced Redux Patterns
7. Explain store splitting / feature-based state modules.
8. How to handle loading states with Redux Toolkit?
9. Error handling patterns in Redux Toolkit
10. How to create custom Redux middleware?
11. How to implement retry logic with Redux?
12. How to cancel API requests in Redux?

### Axios & API Integration
13. How to set up Axios with base URL and default headers?
14. What are Axios interceptors and how to use them?
15. Axios vs fetch comparison - when to use each?

### API Architecture
16. How to structure API service layers?
17. How to handle authentication tokens in API calls?
18. How to implement optimistic updates?

---

## 05-Performance-Optimization

### Performance Analysis
1. How do you identify performance issues in React?
2. How do you avoid unnecessary re-renders?
3. When to use memoization and when NOT to?

### User Experience Patterns
4. Explain Debounce vs Throttle.
5. How do you implement pagination efficiently?
6. How do you implement infinite scrolling in React?
7. Infinite scrolling vs pagination - when to use each?
8. How to optimize infinite scrolling performance?

### List & Data Optimization
9. What is list virtualization and when do you use it?
10. How to implement API response caching?
11. How to handle API rate limiting?

### Bundle & Loading Optimization
12. How does dynamic import reduce bundle size?
13. How does Next.js streaming improve performance?

---

## 06-Micro-Frontend-Architecture

### Fundamentals
1. What are Micro Frontends and why use them?
2. Explain Webpack Module Federation.
3. What is a host and what is a remote?

### Implementation
4. How do you share dependencies (e.g., React) across MFEs?
5. How do MFEs communicate with each other?
6. How do you ensure consistent design across MFEs?
7. How do you version and deploy MFEs independently?

---

## 07-Bundlers

### Bundler Comparison
1. Difference: Vite vs Webpack
2. Why is Vite dev server faster?
3. What is Turbopack and how does Next.js use it?

### Optimization Techniques
4. What is tree-shaking?
5. What is code splitting and when does it occur?

---

## 08-Testing

### Test Types & Strategy
1. Difference between unit and integration tests.
2. How do you test a component that makes API calls?
3. How do you test a custom hook?
4. How do you test components that use context/state?

### Mocking & Error Handling
5. How do you mock fetch or axios in tests?
6. How to handle different types of API errors?
7. How to implement global error handling for APIs?

---

## 09-HTML-CSS-UI

### CSS Layout & Styling
1. Flexbox vs CSS Grid use cases.
2. Difference between CSS Modules vs Styled Components vs Tailwind.
3. What is BEM naming convention?
4. What is the CSS Box Model?
5. Explain margin collapsing in CSS
6. Difference between inline, block, and inline-block elements

### Accessibility
7. Accessibility must-haves (aria, labels, focus control).

---

## 10-Leadership-Agile-Communication

### Project Management
1. How do you estimate tasks in a sprint?
2. How do you ensure quality during fast deadlines?

### Team Collaboration
3. How do you handle disagreements in code reviews?
4. How do you mentor junior developers?

---

## 11-GenAI-Agents

### GenAI Integration
1. Have you integrated any GenAI features in projects?
2. If your app uses an LLM, where should the API call be done in Next.js — client or server — and why?
3. How do you call an LLM safely from a web app?
4. How do you avoid exposing API keys on the client?

### Agents & Tool Calling
5. What is an agent vs a simple LLM API call?
6. What are tool calling and function calling?
7. What is "function calling" / "tool calling" in LLMs?

### Advanced Features
8. Basic RAG: When is retrieval needed?
9. How to stream LLM responses in UI?
10. How would you stream LLM output into a React UI?

---

## 🎯 How to Use This Guide

1. **Pick a section** based on your interview focus
2. **Navigate to the folder** for detailed answers
3. **Practice coding questions** with actual implementations
4. **Review frequently** to reinforce learning
5. **Track your progress** by marking completed topics

## 📝 Answer Format

Each question file contains:
- **Question statement**
- **Placeholder for detailed answer**
- **Space for code examples**
- **Additional notes section**

## 🚀 Quick Navigation

Use your VS Code file explorer or Ctrl+P to quickly jump between questions. Each question is in its own markdown file for easy reference and note-taking.

---

**Total Questions: 143+**  
**Last Updated:** November 2025  
**Target:** Capgemini Interview Preparation