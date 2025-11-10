# Axios vs Fetch

## Question
What's the difference between Axios and fetch?

## Answer
Fetch is built-in browser API, Axios is a library. Axios is easier for most use cases.

## Fetch (Built-in)
```javascript
// GET request
fetch('/api/users')
  .then(response => response.json())
  .then(data => console.log(data));

// POST request
fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'John' })
})
  .then(response => response.json())
  .then(data => console.log(data));
```

## Axios (Library)
```javascript
// GET request
axios.get('/api/users')
  .then(response => console.log(response.data));

// POST request
axios.post('/api/users', { name: 'John' })
  .then(response => console.log(response.data));
```

## Key Differences

| Feature | Fetch | Axios |
|---------|-------|-------|
| Built-in | ✅ Yes | ❌ No (need to install) |
| JSON parsing | ❌ Manual `.json()` | ✅ Automatic |
| Error handling | ❌ Manual status check | ✅ Automatic for HTTP errors |
| Timeout | ❌ Manual | ✅ Built-in |
| Interceptors | ❌ No | ✅ Yes |

## When to Use Each

**Use Fetch:**
- No extra dependencies wanted
- Simple requests
- Bundle size is critical

**Use Axios:**
- Complex applications
- Many API calls
- Need interceptors or better error handling
- Prefer cleaner syntax

## Interview Q&A

**Q: What's the main difference between Axios and fetch?**

A: Fetch is built-in but requires manual JSON parsing and error checking. Axios handles JSON automatically and has better error handling.

**Q: When would you choose Axios over fetch?**

A: For apps with many API calls, when you need automatic JSON handling, interceptors, or better error management.

**Q: Can fetch do everything Axios can?**

A: Yes, but Axios makes it much easier and less code.