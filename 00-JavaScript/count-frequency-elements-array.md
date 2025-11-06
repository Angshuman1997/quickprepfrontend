# Write code to count the frequency of elements in an array

## Question
Write code to count the frequency of elements in an array.

## Answer

Counting the frequency of elements in an array is a common programming task. There are several approaches, each with different performance characteristics and use cases.

## 1. **Using a Regular Object (Most Common Approach)**

```javascript
function countFrequency(arr) {
    const frequency = {};

    for (let i = 0; i < arr.length; i++) {
        const element = arr[i];
        if (frequency[element]) {
            frequency[element]++;
        } else {
            frequency[element] = 1;
        }
    }

    return frequency;
}

// Example
const numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
console.log(countFrequency(numbers));
// { '1': 1, '2': 2, '3': 3, '4': 4 }
```

### **Using Object Property Access Shorthand:**
```javascript
function countFrequency(arr) {
    const frequency = {};

    for (const element of arr) {
        frequency[element] = (frequency[element] || 0) + 1;
    }

    return frequency;
}
```

## 2. **Using Map (Better for Complex Keys)**

```javascript
function countFrequency(arr) {
    const frequency = new Map();

    for (const element of arr) {
        frequency.set(element, (frequency.get(element) || 0) + 1);
    }

    return frequency;
}

// Example with mixed types
const mixed = ['apple', 'banana', 'apple', 1, 1, true, true, true];
const freq = countFrequency(mixed);
console.log(freq.get('apple')); // 2
console.log(freq.get(1));       // 2
console.log(freq.get(true));    // 3
```

## 3. **Using reduce() Method**

```javascript
function countFrequency(arr) {
    return arr.reduce((frequency, element) => {
        frequency[element] = (frequency[element] || 0) + 1;
        return frequency;
    }, {});
}

// Example
const fruits = ['apple', 'banana', 'apple', 'orange', 'banana', 'apple'];
console.log(countFrequency(fruits));
// { apple: 3, banana: 2, orange: 1 }
```

## 4. **Using forEach() Method**

```javascript
function countFrequency(arr) {
    const frequency = {};

    arr.forEach(element => {
        frequency[element] = (frequency[element] || 0) + 1;
    });

    return frequency;
}
```

## 5. **Handling Different Data Types**

### **Strings:**
```javascript
const words = ['hello', 'world', 'hello', 'javascript', 'world', 'hello'];
console.log(countFrequency(words));
// { hello: 3, world: 2, javascript: 1 }
```

### **Numbers:**
```javascript
const scores = [85, 92, 78, 85, 92, 85, 88];
console.log(countFrequency(scores));
// { '78': 1, '85': 3, '88': 1, '92': 2 }
```

### **Objects (Using JSON.stringify as key):**
```javascript
function countObjectFrequency(arr) {
    const frequency = {};

    arr.forEach(obj => {
        const key = JSON.stringify(obj);
        frequency[key] = (frequency[key] || 0) + 1;
    });

    return frequency;
}

const objects = [
    { name: 'Alice', age: 25 },
    { name: 'Bob', age: 30 },
    { name: 'Alice', age: 25 },
    { name: 'Charlie', age: 35 }
];

console.log(countObjectFrequency(objects));
// { '{"name":"Alice","age":25}': 2, '{"name":"Bob","age":30}': 1, '{"name":"Charlie","age":35}': 1 }
```

## 6. **Advanced Techniques**

### **Most Frequent Element:**
```javascript
function findMostFrequent(arr) {
    const frequency = countFrequency(arr);
    let maxCount = 0;
    let mostFrequent = null;

    for (const [element, count] of Object.entries(frequency)) {
        if (count > maxCount) {
            maxCount = count;
            mostFrequent = element;
        }
    }

    return { element: mostFrequent, count: maxCount };
}

const numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
console.log(findMostFrequent(numbers)); // { element: '4', count: 4 }
```

### **Sort by Frequency:**
```javascript
function sortByFrequency(arr) {
    const frequency = countFrequency(arr);

    return arr.slice().sort((a, b) => {
        const freqA = frequency[a];
        const freqB = frequency[b];

        if (freqA !== freqB) {
            return freqB - freqA; // Higher frequency first
        }

        // If frequencies are equal, maintain original order or sort by value
        return a - b; // For numbers
    });
}

const numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
console.log(sortByFrequency(numbers)); // [4, 4, 4, 4, 3, 3, 3, 2, 2, 1]
```

### **Remove Duplicates While Preserving Order:**
```javascript
function removeDuplicates(arr) {
    const seen = new Set();
    return arr.filter(item => {
        if (seen.has(item)) {
            return false;
        }
        seen.add(item);
        return true;
    });
}

const duplicates = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
console.log(removeDuplicates(duplicates)); // [1, 2, 3, 4]
```

### **Group by Frequency:**
```javascript
function groupByFrequency(arr) {
    const frequency = countFrequency(arr);
    const groups = {};

    for (const [element, count] of Object.entries(frequency)) {
        if (!groups[count]) {
            groups[count] = [];
        }
        groups[count].push(element);
    }

    return groups;
}

const numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
console.log(groupByFrequency(numbers));
// { '1': ['1'], '2': ['2'], '3': ['3'], '4': ['4'] }
```

## Real React Examples:

### 1. **Word Frequency Counter**
```javascript
function WordFrequencyCounter() {
    const [text, setText] = useState('');
    const [wordFrequency, setWordFrequency] = useState({});

    const countWords = (inputText) => {
        const words = inputText
            .toLowerCase()
            .replace(/[^\w\s]/g, '') // Remove punctuation
            .split(/\s+/)
            .filter(word => word.length > 0);

        const frequency = countFrequency(words);
        setWordFrequency(frequency);
    };

    useEffect(() => {
        if (text.trim()) {
            countWords(text);
        } else {
            setWordFrequency({});
        }
    }, [text]);

    const sortedWords = Object.entries(wordFrequency)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 10); // Top 10 words

    return (
        <div>
            <textarea
                value={text}
                onChange={(e) => setText(e.target.value)}
                placeholder="Enter text to analyze..."
                rows={6}
            />

            <div className="frequency-results">
                <h3>Top Words by Frequency:</h3>
                {sortedWords.length > 0 ? (
                    <ul>
                        {sortedWords.map(([word, count]) => (
                            <li key={word}>
                                <span className="word">{word}</span>
                                <span className="count">({count})</span>
                            </li>
                        ))}
                    </ul>
                ) : (
                    <p>No words to analyze</p>
                )}
            </div>
        </div>
    );
}
```

### 2. **Shopping Cart Item Counter**
```javascript
function ShoppingCart() {
    const [cart, setCart] = useState([
        { id: 1, name: 'Laptop', price: 1000 },
        { id: 2, name: 'Mouse', price: 50 },
        { id: 1, name: 'Laptop', price: 1000 }, // Duplicate item
        { id: 3, name: 'Keyboard', price: 80 },
        { id: 2, name: 'Mouse', price: 50 }     // Another duplicate
    ]);

    // Count frequency of each item
    const itemCounts = useMemo(() => {
        return cart.reduce((counts, item) => {
            counts[item.id] = (counts[item.id] || 0) + 1;
            return counts;
        }, {});
    }, [cart]);

    // Get unique items with quantities
    const uniqueItems = useMemo(() => {
        const frequency = countFrequency(cart.map(item => item.id));

        return cart
            .filter((item, index, arr) => {
                return arr.findIndex(i => i.id === item.id) === index;
            })
            .map(item => ({
                ...item,
                quantity: frequency[item.id]
            }));
    }, [cart]);

    const totalItems = cart.length;
    const uniqueItemCount = uniqueItems.length;
    const totalPrice = uniqueItems.reduce((sum, item) => sum + (item.price * item.quantity), 0);

    return (
        <div className="cart">
            <h2>Shopping Cart</h2>
            <p>Total items: {totalItems} ({uniqueItemCount} unique)</p>

            <div className="cart-items">
                {uniqueItems.map(item => (
                    <div key={item.id} className="cart-item">
                        <span>{item.name}</span>
                        <span>Quantity: {item.quantity}</span>
                        <span>${item.price * item.quantity}</span>
                    </div>
                ))}
            </div>

            <div className="cart-total">
                <strong>Total: ${totalPrice}</strong>
            </div>
        </div>
    );
}
```

### 3. **Vote Counter / Poll Results**
```javascript
function PollResults({ votes }) {
    // votes is an array of vote objects: [{ candidate: 'Alice' }, { candidate: 'Bob' }, ...]

    const voteCounts = useMemo(() => {
        return countFrequency(votes.map(vote => vote.candidate));
    }, [votes]);

    const totalVotes = votes.length;
    const sortedCandidates = Object.entries(voteCounts)
        .sort(([, a], [, b]) => b - a);

    const winner = sortedCandidates[0]?.[0] || 'No votes yet';
    const maxVotes = sortedCandidates[0]?.[1] || 0;

    return (
        <div className="poll-results">
            <h2>Poll Results</h2>
            <p>Total votes: {totalVotes}</p>
            <p>Winner: {winner} ({maxVotes} votes)</p>

            <div className="results-chart">
                {sortedCandidates.map(([candidate, count]) => {
                    const percentage = totalVotes > 0 ? ((count / totalVotes) * 100).toFixed(1) : 0;
                    const isWinner = count === maxVotes && maxVotes > 0;

                    return (
                        <div key={candidate} className={`result-bar ${isWinner ? 'winner' : ''}`}>
                            <div className="candidate-name">{candidate}</div>
                            <div className="bar-container">
                                <div
                                    className="bar"
                                    style={{ width: `${percentage}%` }}
                                ></div>
                                <span className="percentage">{percentage}% ({count} votes)</span>
                            </div>
                        </div>
                    );
                })}
            </div>
        </div>
    );
}
```

### 4. **Tag Cloud Component**
```javascript
function TagCloud({ posts }) {
    // posts is an array of post objects with tags: [{ tags: ['react', 'javascript'] }, ...]

    const tagFrequency = useMemo(() => {
        const allTags = posts.flatMap(post => post.tags || []);
        return countFrequency(allTags);
    }, [posts]);

    // Calculate font sizes based on frequency
    const getFontSize = (count) => {
        const minSize = 12;
        const maxSize = 24;
        const minFreq = Math.min(...Object.values(tagFrequency));
        const maxFreq = Math.max(...Object.values(tagFrequency));

        if (minFreq === maxFreq) return minSize;

        return minSize + ((count - minFreq) / (maxFreq - minFreq)) * (maxSize - minSize);
    };

    const sortedTags = Object.entries(tagFrequency)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 20); // Show top 20 tags

    return (
        <div className="tag-cloud">
            {sortedTags.map(([tag, count]) => (
                <span
                    key={tag}
                    className="tag"
                    style={{
                        fontSize: `${getFontSize(count)}px`,
                        opacity: 0.7 + (count / Math.max(...Object.values(tagFrequency))) * 0.3
                    }}
                    onClick={() => console.log(`Filter by: ${tag}`)}
                >
                    {tag} ({count})
                </span>
            ))}
        </div>
    );
}
```

### 5. **Analytics Dashboard**
```javascript
function UserActivityDashboard({ activities }) {
    // activities: [{ userId: 1, action: 'login' }, { userId: 2, action: 'view' }, ...]

    const userActivityCount = useMemo(() => {
        return countFrequency(activities.map(activity => activity.userId));
    }, [activities]);

    const actionFrequency = useMemo(() => {
        return countFrequency(activities.map(activity => activity.action));
    }, [activities]);

    const topUsers = Object.entries(userActivityCount)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 5);

    const topActions = Object.entries(actionFrequency)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 5);

    return (
        <div className="dashboard">
            <div className="metric">
                <h3>Total Activities</h3>
                <span className="number">{activities.length}</span>
            </div>

            <div className="metric">
                <h3>Unique Users</h3>
                <span className="number">{Object.keys(userActivityCount).length}</span>
            </div>

            <div className="section">
                <h3>Most Active Users</h3>
                <ul>
                    {topUsers.map(([userId, count]) => (
                        <li key={userId}>User {userId}: {count} activities</li>
                    ))}
                </ul>
            </div>

            <div className="section">
                <h3>Popular Actions</h3>
                <ul>
                    {topActions.map(([action, count]) => (
                        <li key={action}>{action}: {count} times</li>
                    ))}
                </ul>
            </div>
        </div>
    );
}
```

## Performance Considerations:

- **Time Complexity**: All methods are O(n) where n is array length
- **Space Complexity**: O(m) where m is number of unique elements
- **For large arrays**: Consider using Maps for better performance with complex keys
- **Memory usage**: Objects use more memory than Maps for simple keys

## Common Use Cases:

1. **Word frequency analysis** in text processing
2. **Vote counting** in polls and elections
3. **Shopping cart** item quantities
4. **Analytics** and user behavior tracking
5. **Duplicate detection** and removal
6. **Tag clouds** and categorization
7. **Histogram generation** for data visualization

### Interview Tip:
*"Use objects for simple keys (strings/numbers), Maps for complex keys. The pattern `frequency[element] = (frequency[element] || 0) + 1` is efficient and commonly used. Consider performance when dealing with large datasets."*