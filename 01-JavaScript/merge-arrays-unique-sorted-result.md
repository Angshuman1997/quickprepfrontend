# Merge Arrays: Unique & Sorted

## Simple Solution
```javascript
function mergeUniqueSorted(arr1, arr2) {
  // Combine arrays
  const combined = [...arr1, ...arr2];
  
  // Remove duplicates with Set, then sort
  return [...new Set(combined)].sort((a, b) => a - b);
}

// Example
const arr1 = [3, 1, 4, 1, 5];
const arr2 = [9, 2, 6, 5, 3];
console.log(mergeUniqueSorted(arr1, arr2)); // [1, 2, 3, 4, 5, 6, 9]
```

## Alternative: Using filter
```javascript
function mergeUniqueSorted(arr1, arr2) {
  const combined = arr1.concat(arr2);
  return combined
    .filter((item, index) => combined.indexOf(item) === index)
    .sort((a, b) => a - b);
}
```

## Interview Q&A

**Q: How do you merge two arrays and remove duplicates?**  
**A:** Use spread operator to combine, then Set to remove duplicates: `[...new Set([...arr1, ...arr2])]`

**Q: How do you sort the result?**  
**A:** Call .sort() on the array. For numbers: .sort((a, b) => a - b)

**Q: What's the most efficient way?**  
**A:** Using Set is usually fastest for large arrays since it handles uniqueness during creation.

console.log(mergeUniqueSorted(arr1, arr2));
// [1, 2, 3, 4, 5, 6, 9]

const strings1 = ['banana', 'apple', 'cherry'];
const strings2 = ['date', 'apple', 'elderberry'];

console.log(mergeUniqueSorted(strings1, strings2));
// ['apple', 'banana', 'cherry', 'date', 'elderberry']
```

## 2. **Using Spread Operator and Set**

```javascript
function mergeUniqueSorted(...arrays) {
    // Merge all arrays and remove duplicates
    const unique = [...new Set(arrays.flat())];

    // Sort the result
    return unique.sort((a, b) => {
        if (typeof a === 'number' && typeof b === 'number') {
            return a - b;
        }
        if (typeof a === 'string' && typeof b === 'string') {
            return a.localeCompare(b);
        }
        // Mixed types: convert to strings for comparison
        return String(a).localeCompare(String(b));
    });
}

// Usage with multiple arrays
const result = mergeUniqueSorted([3, 1, 4], [1, 5, 9], [2, 6, 5]);
console.log(result); // [1, 2, 3, 4, 5, 6, 9]
```

## 3. **Using reduce() for Complex Merging**

```javascript
function mergeUniqueSorted(arr1, arr2, comparator) {
    // Create a map to track unique items
    const uniqueMap = new Map();

    // Add items from first array
    arr1.forEach(item => uniqueMap.set(item, item));

    // Add items from second array
    arr2.forEach(item => uniqueMap.set(item, item));

    // Convert to array and sort
    const uniqueArray = Array.from(uniqueMap.values());

    return uniqueArray.sort(comparator || ((a, b) => {
        if (typeof a === 'number' && typeof b === 'number') {
            return a - b;
        }
        return String(a).localeCompare(String(b));
    }));
}

// Custom comparator example
const numbers = [3, 1, 4, 1, 5];
const moreNumbers = [9, 2, 6, 5, 3];

const result = mergeUniqueSorted(numbers, moreNumbers);
console.log(result); // [1, 2, 3, 4, 5, 6, 9]
```

## 4. **Handling Objects by Property**

```javascript
function mergeUniqueSortedByKey(arr1, arr2, key) {
    // Create a map using the specified key
    const uniqueMap = new Map();

    // Add items from both arrays
    [...arr1, ...arr2].forEach(item => {
        const keyValue = item[key];
        uniqueMap.set(keyValue, item);
    });

    // Convert to array and sort by the key
    return Array.from(uniqueMap.values()).sort((a, b) => {
        const aVal = a[key];
        const bVal = b[key];

        if (typeof aVal === 'number' && typeof bVal === 'number') {
            return aVal - bVal;
        }
        return String(aVal).localeCompare(String(bVal));
    });
}

// Example with objects
const users1 = [
    { id: 1, name: 'Alice', age: 25 },
    { id: 2, name: 'Bob', age: 30 }
];

const users2 = [
    { id: 1, name: 'Alice', age: 25 }, // Duplicate
    { id: 3, name: 'Charlie', age: 20 }
];

const mergedUsers = mergeUniqueSortedByKey(users1, users2, 'id');
console.log(mergedUsers);
// [
//   { id: 1, name: 'Alice', age: 25 },
//   { id: 2, name: 'Bob', age: 30 },
//   { id: 3, name: 'Charlie', age: 20 }
// ]
```

## 5. **Merging Sorted Arrays Efficiently**

```javascript
function mergeSortedUnique(arr1, arr2) {
    // Assume input arrays are already sorted
    const result = [];
    let i = 0, j = 0;

    while (i < arr1.length && j < arr2.length) {
        if (arr1[i] < arr2[j]) {
            // Avoid duplicates
            if (result.length === 0 || result[result.length - 1] !== arr1[i]) {
                result.push(arr1[i]);
            }
            i++;
        } else if (arr1[i] > arr2[j]) {
            if (result.length === 0 || result[result.length - 1] !== arr2[j]) {
                result.push(arr2[j]);
            }
            j++;
        } else {
            // Equal elements
            if (result.length === 0 || result[result.length - 1] !== arr1[i]) {
                result.push(arr1[i]);
            }
            i++;
            j++;
        }
    }

    // Add remaining elements from arr1
    while (i < arr1.length) {
        if (result.length === 0 || result[result.length - 1] !== arr1[i]) {
            result.push(arr1[i]);
        }
        i++;
    }

    // Add remaining elements from arr2
    while (j < arr2.length) {
        if (result.length === 0 || result[result.length - 1] !== arr2[j]) {
            result.push(arr2[j]);
        }
        j++;
    }

    return result;
}

// Example with pre-sorted arrays
const sorted1 = [1, 3, 5, 7, 9];
const sorted2 = [2, 3, 5, 6, 8, 9];

console.log(mergeSortedUnique(sorted1, sorted2));
// [1, 2, 3, 5, 6, 7, 8, 9]
```

## 6. **Advanced Techniques**

### **Merging with Custom Uniqueness Criteria**

```javascript
function mergeUniqueSortedCustom(arr1, arr2, isEqual, comparator) {
    const combined = [...arr1, ...arr2];
    const unique = [];

    for (const item of combined) {
        const exists = unique.some(existing => isEqual(existing, item));
        if (!exists) {
            unique.push(item);
        }
    }

    return unique.sort(comparator);
}

// Example: Merge users by email (case-insensitive)
const usersA = [
    { email: 'alice@example.com', name: 'Alice' },
    { email: 'bob@example.com', name: 'Bob' }
];

const usersB = [
    { email: 'ALICE@EXAMPLE.COM', name: 'Alice Smith' }, // Same person
    { email: 'charlie@example.com', name: 'Charlie' }
];

const merged = mergeUniqueSortedCustom(
    usersA,
    usersB,
    (a, b) => a.email.toLowerCase() === b.email.toLowerCase(),
    (a, b) => a.email.localeCompare(b.email)
);

console.log(merged);
// [{ email: 'alice@example.com', name: 'Alice' }, { email: 'bob@example.com', name: 'Bob' }, { email: 'charlie@example.com', name: 'Charlie' }]
```

### **Merging Large Arrays with Memory Efficiency**

```javascript
function mergeUniqueSortedEfficient(arr1, arr2) {
    // Use a Set for O(1) lookup time
    const seen = new Set(arr1);
    const result = [...arr1];

    // Add unique elements from arr2
    for (const item of arr2) {
        if (!seen.has(item)) {
            seen.add(item);
            result.push(item);
        }
    }

    // Sort the result
    return result.sort((a, b) => {
        if (typeof a === 'number' && typeof b === 'number') {
            return a - b;
        }
        return String(a).localeCompare(String(b));
    });
}

const largeArr1 = Array.from({ length: 1000 }, (_, i) => i * 2);
const largeArr2 = Array.from({ length: 1000 }, (_, i) => i * 3);

const start = Date.now();
const result = mergeUniqueSortedEfficient(largeArr1, largeArr2);
const end = Date.now();

console.log(`Merged ${result.length} unique elements in ${end - start}ms`);
```

## Real React Examples:

### 1. **Merging Tag Lists from Multiple Sources**

```javascript
function TagManager({ initialTags, userTags }) {
    const [allTags, setAllTags] = useState([]);

    useEffect(() => {
        // Merge and deduplicate tags from different sources
        const merged = [...initialTags, ...userTags];
        const unique = [...new Set(merged)].sort((a, b) => a.localeCompare(b));
        setAllTags(unique);
    }, [initialTags, userTags]);

    const addTag = (newTag) => {
        if (!allTags.includes(newTag)) {
            setAllTags(prev => [...prev, newTag].sort((a, b) => a.localeCompare(b)));
        }
    };

    const removeTag = (tagToRemove) => {
        setAllTags(prev => prev.filter(tag => tag !== tagToRemove));
    };

    return (
        <div>
            <div className="tags">
                {allTags.map(tag => (
                    <span key={tag} className="tag">
                        {tag}
                        <button onClick={() => removeTag(tag)}>×</button>
                    </span>
                ))}
            </div>
            <input
                type="text"
                placeholder="Add new tag..."
                onKeyPress={(e) => {
                    if (e.key === 'Enter' && e.target.value.trim()) {
                        addTag(e.target.value.trim());
                        e.target.value = '';
                    }
                }}
            />
        </div>
    );
}

// Usage
const initialTags = ['react', 'javascript', 'web-development'];
const userTags = ['typescript', 'react', 'node.js'];

<TagManager initialTags={initialTags} userTags={userTags} />
```

### 2. **Merging Shopping Cart Items**

```javascript
function ShoppingCart() {
    const [cartItems, setCartItems] = useState([]);

    // Function to add items to cart
    const addToCart = (newItems) => {
        setCartItems(prevItems => {
            // Merge existing and new items
            const combined = [...prevItems, ...newItems];

            // Group by product ID and sum quantities
            const grouped = combined.reduce((acc, item) => {
                const existing = acc.find(i => i.id === item.id);
                if (existing) {
                    existing.quantity += item.quantity;
                } else {
                    acc.push({ ...item });
                }
                return acc;
            }, []);

            // Sort by product name
            return grouped.sort((a, b) => a.name.localeCompare(b.name));
        });
    };

    // Calculate total
    const total = cartItems.reduce((sum, item) => sum + (item.price * item.quantity), 0);

    return (
        <div className="cart">
            <h2>Shopping Cart</h2>
            {cartItems.map(item => (
                <div key={item.id} className="cart-item">
                    <span>{item.name}</span>
                    <span>Qty: {item.quantity}</span>
                    <span>${(item.price * item.quantity).toFixed(2)}</span>
                </div>
            ))}
            <div className="total">Total: ${total.toFixed(2)}</div>
        </div>
    );
}

// Example usage
const newItems = [
    { id: 1, name: 'Laptop', price: 1000, quantity: 1 },
    { id: 2, name: 'Mouse', price: 50, quantity: 2 }
];

// Add items (would merge with existing cart)
cart.addToCart(newItems);
```

### 3. **Merging Search Results from Multiple APIs**

```javascript
function SearchResults({ queries }) {
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);

    const searchMultipleSources = async (searchQueries) => {
        setLoading(true);
        try {
            // Search multiple APIs in parallel
            const searchPromises = searchQueries.map(query =>
                fetch(`/api/search?q=${encodeURIComponent(query)}`)
                    .then(res => res.json())
            );

            const searchResults = await Promise.all(searchPromises);

            // Merge and deduplicate results
            const allResults = searchResults.flat();
            const uniqueResults = allResults.filter((item, index, arr) =>
                arr.findIndex(i => i.id === item.id) === index
            );

            // Sort by relevance score
            const sortedResults = uniqueResults.sort((a, b) => b.score - a.score);

            setResults(sortedResults);
        } catch (error) {
            console.error('Search failed:', error);
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => {
        if (queries.length > 0) {
            searchMultipleSources(queries);
        }
    }, [queries]);

    return (
        <div className="search-results">
            {loading ? (
                <div>Loading...</div>
            ) : (
                results.map(result => (
                    <div key={result.id} className="result-item">
                        <h3>{result.title}</h3>
                        <p>{result.description}</p>
                        <span>Score: {result.score}</span>
                    </div>
                ))
            )}
        </div>
    );
}
```

### 4. **Merging User Permissions**

```javascript
function UserPermissions({ rolePermissions, userPermissions }) {
    const [effectivePermissions, setEffectivePermissions] = useState([]);

    useEffect(() => {
        // Merge role-based and user-specific permissions
        const allPermissions = [...rolePermissions, ...userPermissions];

        // Remove duplicates and sort
        const uniquePermissions = [...new Set(allPermissions)]
            .sort((a, b) => a.localeCompare(b));

        setEffectivePermissions(uniquePermissions);
    }, [rolePermissions, userPermissions]);

    const hasPermission = (permission) => {
        return effectivePermissions.includes(permission);
    };

    return (
        <div className="permissions">
            <h3>Effective Permissions</h3>
            <ul>
                {effectivePermissions.map(perm => (
                    <li key={perm}>{perm}</li>
                ))}
            </ul>

            <div className="permission-checks">
                <p>Can edit: {hasPermission('edit') ? 'Yes' : 'No'}</p>
                <p>Can delete: {hasPermission('delete') ? 'Yes' : 'No'}</p>
                <p>Can admin: {hasPermission('admin') ? 'Yes' : 'No'}</p>
            </div>
        </div>
    );
}

// Example data
const rolePermissions = ['read', 'write', 'edit'];
const userPermissions = ['edit', 'delete', 'admin'];

<UserPermissions
    rolePermissions={rolePermissions}
    userPermissions={userPermissions}
/>
```

### 5. **Merging Contact Lists**

```javascript
function ContactMerger({ contactLists }) {
    const [mergedContacts, setMergedContacts] = useState([]);

    useEffect(() => {
        // Flatten all contact lists
        const allContacts = contactLists.flat();

        // Remove duplicates based on email (case-insensitive)
        const uniqueContacts = allContacts.filter((contact, index, arr) => {
            const email = contact.email.toLowerCase();
            return arr.findIndex(c => c.email.toLowerCase() === email) === index;
        });

        // Sort by name
        const sortedContacts = uniqueContacts.sort((a, b) => {
            return a.name.localeCompare(b.name);
        });

        setMergedContacts(sortedContacts);
    }, [contactLists]);

    return (
        <div className="contacts">
            <h2>Merged Contacts ({mergedContacts.length})</h2>
            {mergedContacts.map(contact => (
                <div key={contact.email} className="contact">
                    <div className="name">{contact.name}</div>
                    <div className="email">{contact.email}</div>
                    <div className="phone">{contact.phone}</div>
                </div>
            ))}
        </div>
    );
}

// Example usage
const contacts1 = [
    { name: 'Alice Johnson', email: 'alice@example.com', phone: '555-0101' },
    { name: 'Bob Smith', email: 'bob@example.com', phone: '555-0102' }
];

const contacts2 = [
    { name: 'Alice Johnson', email: 'ALICE@EXAMPLE.COM', phone: '555-0101' }, // Duplicate
    { name: 'Charlie Brown', email: 'charlie@example.com', phone: '555-0103' }
];

<ContactMerger contactLists={[contacts1, contacts2]} />
```

## Performance Considerations:

- **Set approach**: O(n + m) time, good for large arrays
- **Sort after merge**: O((n+m) log(n+m)) time due to sorting
- **Pre-sorted merge**: O(n + m) time if inputs are sorted
- **Memory usage**: Set uses more memory but provides O(1) lookup
- **Large datasets**: Consider streaming/chunking for very large arrays

## Common Patterns:

1. **Simple merge**: `[...new Set([...arr1, ...arr2])].sort()`
2. **Object merge by key**: Use Map with key property
3. **Custom uniqueness**: Provide custom equality function
4. **Sorted merge**: Use two-pointer technique for pre-sorted arrays
5. **Memory efficient**: Use Set for uniqueness, then convert to sorted array

### Interview Tip:
*"Use Set for O(1) uniqueness checks, then sort. For objects, define uniqueness criteria. Pre-sorted arrays can be merged in O(n+m) time using two pointers. Always consider data types for proper sorting."*