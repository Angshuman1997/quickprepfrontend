# How to structure API service layers?

## Question
How to structure API service layers?

## Answer
API service layers separate data fetching logic from UI components. Instead of making HTTP calls directly in components, you create dedicated service functions that handle API communication.

## Why Use Service Layers?

**Problems without service layers:**
- Components mix UI logic with HTTP calls
- Same API code repeated in multiple places
- Hard to test components with real API calls
- API changes require updating many files

**Benefits:**
- Clean separation: Components focus on UI, services handle data
- Reusable: Same service used across multiple components
- Testable: Easy to mock services for testing
- Maintainable: API changes only in service layer

## Basic Structure

```
src/
├── services/
│   ├── api.ts          # API service functions
│   └── types.ts        # Type definitions
```

## Simple Example

**API Service:**
```typescript
// services/api.ts
const API_BASE = 'https://api.example.com';

export const apiService = {
  async getUsers() {
    const response = await fetch(`${API_BASE}/users`);
    if (!response.ok) throw new Error('Failed to fetch users');
    return response.json();
  },

  async createUser(userData) {
    const response = await fetch(`${API_BASE}/users`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(userData)
    });
    if (!response.ok) throw new Error('Failed to create user');
    return response.json();
  }
};
```

**Component using service:**
```typescript
// components/UserList.tsx
import { apiService } from '../services/api';

function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    const fetchUsers = async () => {
      setLoading(true);
      try {
        const data = await apiService.getUsers();
        setUsers(data);
      } catch (error) {
        console.error('Error fetching users:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchUsers();
  }, []);

  // Component focuses only on UI rendering
}
```

## Interview Q&A

**Q: Why use service layers instead of direct API calls in components?**

A: Service layers separate concerns - components handle UI, services handle data fetching. This makes code more reusable, testable, and maintainable. API changes only need updates in one place.

**Q: How do you structure a simple API service?**

A: Create an object with methods for each API endpoint. Each method handles the fetch call, error checking, and returns the data. Components call these service methods instead of making fetch calls directly.

**Q: What's the main benefit of service layers?**

A: Cleaner components that focus on presentation logic, not data fetching. Same API logic can be reused across multiple components, and it's much easier to test and maintain.