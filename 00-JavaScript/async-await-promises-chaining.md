# async/await vs Promises

## Simple Answer
async/await is easier syntax for working with Promises. Makes async code look synchronous.

## Basic Example

**Promise:**
```javascript
function getUser(id) {
  return fetch(`/api/user/${id}`)
    .then(response => response.json())
    .then(user => {
      console.log(user);
      return user;
    });
}
```

**async/await:**
```javascript
async function getUser(id) {
  const response = await fetch(`/api/user/${id}`);
  const user = await response.json();
  console.log(user);
  return user;
}
```

## Key Points

- `async` functions always return a Promise
- `await` pauses execution until Promise resolves
- Use `try/catch` for error handling

## In React
```javascript
function UserComponent({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    async function loadUser() {
      try {
        const userData = await fetchUser(userId);
        setUser(userData);
      } catch (error) {
        console.error('Failed to load user');
      }
    }
    loadUser();
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

## Interview Q&A

**Q: What's async/await?**  
**A:** It's syntax that makes working with Promises easier. Code looks synchronous but runs asynchronously.

**Q: How does it relate to Promises?**  
**A:** async/await is built on Promises. async functions return Promises, await works with Promises.

**Q: When would you use async/await vs Promise chaining?**  
**A:** async/await for complex logic with multiple steps. Promise chaining for simple transformations.