# Callbacks and Callback Hell

## What are Callbacks?
Functions passed as arguments to other functions, called when async operation completes.

```javascript
function fetchData(callback) {
  setTimeout(() => {
    const data = { name: "John" };
    callback(data);
  }, 1000);
}

fetchData((user) => {
  console.log(user); // { name: "John" }
});
```

## Callback Hell
Too many nested callbacks make code hard to read.

```javascript
// ❌ Bad - Callback Hell
getUser(userId, (user) => {
  getPosts(user.id, (posts) => {
    getComments(posts[0].id, (comments) => {
      // Deep nesting, hard to read
    });
  });
});
```

## Solutions

**Promises:**
```javascript
// ✅ Better - Promises
getUser(userId)
  .then(user => getPosts(user.id))
  .then(posts => getComments(posts[0].id))
  .then(comments => {
    // Much cleaner!
  });
```

**async/await:**
```javascript
// ✅ Best - async/await
async function loadData(userId) {
  const user = await getUser(userId);
  const posts = await getPosts(user.id);
  const comments = await getComments(posts[0].id);
  // Clean and readable!
}
```

## Interview Q&A

**Q: What is callback hell?**  
**A:** When you have many nested callbacks, making code hard to read and maintain.

**Q: How do you solve callback hell?**  
**A:** Use Promises with .then() chaining, or async/await syntax.

**Q: What's the difference between callbacks and Promises?**  
**A:** Callbacks are functions passed to async functions. Promises represent future values and avoid nesting.
getUser(userId, (user) => {
    console.log("User:", user);
    getUserPosts(user.id, (posts) => {
        console.log("Posts:", posts);
        getPostComments(posts[0].id, (comments) => {
            console.log("Comments:", comments);
            getCommentAuthor(comments[0].authorId, (author) => {
                console.log("Author:", author);
                // Even more nesting...
            }, (error) => console.error("Author error:", error));
        }, (error) => console.error("Comments error:", error));
    }, (error) => console.error("Posts error:", error));
}, (error) => console.error("User error:", error));
```

Problems with callback hell:
- **Hard to read** - Deep nesting
- **Hard to debug** - Complex error handling
- **Hard to maintain** - Difficult to modify
- **Hard to test** - Complex dependencies

## Solutions to Callback Hell:

### 1. **Named Functions (Extract callbacks):**
```javascript
// ✅ Solution 1: Extract callbacks into named functions
function handleUser(user) {
    console.log("User:", user);
    getUserPosts(user.id, handlePosts, handleError);
}

function handlePosts(posts) {
    console.log("Posts:", posts);
    getPostComments(posts[0].id, handleComments, handleError);
}

function handleComments(comments) {
    console.log("Comments:", comments);
    // Continue...
}

function handleError(error) {
    console.error("Error:", error);
}

getUser(userId, handleUser, handleError);
```

### 2. **Promises:**
```javascript
// ✅ Solution 2: Use Promises
function getUser(userId) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (userId) {
                resolve({ id: userId, name: "John" });
            } else {
                reject(new Error("Invalid user ID"));
            }
        }, 1000);
    });
}

function getUserPosts(userId) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve([{ id: 1, title: "Post 1" }]);
        }, 500);
    });
}

// Chain promises instead of nesting
getUser(userId)
    .then(user => {
        console.log("User:", user);
        return getUserPosts(user.id);
    })
    .then(posts => {
        console.log("Posts:", posts);
        return getPostComments(posts[0].id);
    })
    .then(comments => {
        console.log("Comments:", comments);
    })
    .catch(error => {
        console.error("Error:", error);
    });
```

### 3. **Async/Await (Modern Solution):**
```javascript
// ✅ Solution 3: Use async/await (recommended)
async function loadUserData(userId) {
    try {
        const user = await getUser(userId);
        console.log("User:", user);

        const posts = await getUserPosts(user.id);
        console.log("Posts:", posts);

        const comments = await getPostComments(posts[0].id);
        console.log("Comments:", comments);

        const author = await getCommentAuthor(comments[0].authorId);
        console.log("Author:", author);

    } catch (error) {
        console.error("Error:", error);
    }
}

// Usage
loadUserData(123);
```

### 4. **Promise.all() for Parallel Operations:**
```javascript
// ✅ Solution 4: Run operations in parallel
async function loadUserDashboard(userId) {
    try {
        // Start all requests simultaneously
        const [user, posts, notifications] = await Promise.all([
            getUser(userId),
            getUserPosts(userId),
            getUserNotifications(userId)
        ]);

        console.log("User:", user);
        console.log("Posts:", posts);
        console.log("Notifications:", notifications);

    } catch (error) {
        console.error("Error:", error);
    }
}
```

### 5. **Modular Approach (Separate Functions):**
```javascript
// ✅ Solution 5: Break into smaller, focused functions
class UserService {
    async loadUserProfile(userId) {
        const user = await this.getUser(userId);
        const posts = await this.getUserPosts(userId);
        return { user, posts };
    }

    async loadPostDetails(postId) {
        const post = await this.getPost(postId);
        const comments = await this.getPostComments(postId);
        const author = await this.getCommentAuthor(comments[0]?.authorId);
        return { post, comments, author };
    }

    async loadUserDashboard(userId) {
        const profile = await this.loadUserProfile(userId);
        const recentActivity = await this.getRecentActivity(userId);
        return { ...profile, recentActivity };
    }
}
```

## Real React Example:

### ❌ Callback Hell in React (avoid this):
```javascript
// Bad: Callback hell in React
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);

    useEffect(() => {
        fetchUser(userId, (userData) => {
            setUser(userData);
            fetchUserPosts(userId, (posts) => {
                setUserPosts(posts);
                fetchUserFriends(userId, (friends) => {
                    setUserFriends(friends);
                    // More nesting...
                }, (error) => setError(error));
            }, (error) => setError(error));
        }, (error) => setError(error));
    }, [userId]);
}
```

### ✅ Clean Solution with async/await:
```javascript
// Good: Clean async/await in React
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [friends, setFriends] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        async function loadUserData() {
            try {
                setLoading(true);
                setError(null);

                // Load data in parallel for better performance
                const [userData, postsData, friendsData] = await Promise.all([
                    fetchUser(userId),
                    fetchUserPosts(userId),
                    fetchUserFriends(userId)
                ]);

                setUser(userData);
                setPosts(postsData);
                setFriends(friendsData);

            } catch (err) {
                setError(err.message);
            } finally {
                setLoading(false);
            }
        }

        loadUserData();
    }, [userId]);

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <div>
            <h1>{user.name}</h1>
            <div>Posts: {posts.length}</div>
            <div>Friends: {friends.length}</div>
        </div>
    );
}
```

## When to Use Callbacks vs Other Solutions:

### Use Callbacks When:
- Simple, single-level async operations
- Working with existing callback-based APIs
- Building utility functions that need flexibility

### Use Promises/Async-Await When:
- Complex async workflows
- Multiple dependent operations
- Better error handling needed
- Code readability is important

### Interview Tip:
*"Callback hell is like a pyramid - it starts small but gets unmanageable. The solution is to flatten it using Promises or async/await. Always prefer async/await in modern JavaScript for better readability and error handling."*