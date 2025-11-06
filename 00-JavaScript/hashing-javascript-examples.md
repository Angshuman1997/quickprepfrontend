# What is hashing? Explain with JavaScript examples.

## Question
What is hashing? Explain with JavaScript examples.

## Answer

**Hashing** is the process of converting input data (of any size) into a fixed-size string of characters, typically a hash value or hash code. The output is usually a short, fixed-length string that represents the input data. Hash functions are designed to be fast to compute and have the property that even a small change in input produces a significantly different hash value.

## 1. **Basic Concepts of Hashing**

### **Properties of Good Hash Functions:**
- **Deterministic**: Same input always produces same hash
- **Fast**: Quick to compute
- **Avalanche Effect**: Small input changes produce large hash changes
- **Preimage Resistance**: Hard to find input from hash
- **Collision Resistance**: Hard to find two inputs with same hash

### **Common Use Cases:**
- Password storage
- Data integrity verification
- Digital signatures
- Caching
- Data structures (Hash tables)
- File comparison
- Blockchain/cryptocurrency

## 2. **Built-in JavaScript Hashing**

JavaScript doesn't have built-in cryptographic hashing in the standard library, but you can use the Web Crypto API in browsers or Node.js crypto module.

### **Using Web Crypto API (Browser):**
```javascript
async function hashString(message) {
    const encoder = new TextEncoder();
    const data = encoder.encode(message);
    const hashBuffer = await crypto.subtle.digest('SHA-256', data);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    return hashHex;
}

// Usage
hashString('Hello World').then(hash => {
    console.log(hash); // "a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e49"
});
```

### **Using Node.js Crypto Module:**
```javascript
const crypto = require('crypto');

function hashString(message) {
    return crypto.createHash('sha256').update(message).digest('hex');
}

console.log(hashString('Hello World'));
// "a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e49"
```

## 3. **Simple Hash Functions (Non-Cryptographic)**

### **Simple String Hash (djb2 algorithm):**
```javascript
function simpleHash(str) {
    let hash = 5381;

    for (let i = 0; i < str.length; i++) {
        hash = ((hash << 5) + hash) + str.charCodeAt(i); // hash * 33 + char
    }

    return hash >>> 0; // Convert to unsigned 32-bit integer
}

console.log(simpleHash('hello'));     // 99162322
console.log(simpleHash('world'));     // 113318802
console.log(simpleHash('hello world')); // 478340078
```

### **FNV-1a Hash:**
```javascript
function fnv1aHash(str) {
    const FNV_PRIME = 0x01000193;
    const FNV_OFFSET_BASIS = 0x811c9dc5;
    let hash = FNV_OFFSET_BASIS;

    for (let i = 0; i < str.length; i++) {
        hash ^= str.charCodeAt(i);
        hash = (hash * FNV_PRIME) >>> 0;
    }

    return hash;
}

console.log(fnv1aHash('hello')); // 1335831723
console.log(fnv1aHash('world')); // 933488787
```

## 4. **Hash Tables / Objects as Hash Maps**

### **Using Objects as Hash Maps:**
```javascript
class SimpleHashTable {
    constructor() {
        this.table = {};
    }

    // Simple hash function for demonstration
    hash(key) {
        let hash = 0;
        for (let i = 0; i < key.length; i++) {
            hash += key.charCodeAt(i);
        }
        return hash % 100; // Simple modulo for bucket
    }

    set(key, value) {
        const hashKey = this.hash(key);
        if (!this.table[hashKey]) {
            this.table[hashKey] = [];
        }
        // Handle collisions with separate chaining
        const bucket = this.table[hashKey];
        for (let i = 0; i < bucket.length; i++) {
            if (bucket[i][0] === key) {
                bucket[i][1] = value; // Update existing
                return;
            }
        }
        bucket.push([key, value]); // Add new
    }

    get(key) {
        const hashKey = this.hash(key);
        const bucket = this.table[hashKey];
        if (bucket) {
            for (let i = 0; i < bucket.length; i++) {
                if (bucket[i][0] === key) {
                    return bucket[i][1];
                }
            }
        }
        return undefined;
    }

    remove(key) {
        const hashKey = this.hash(key);
        const bucket = this.table[hashKey];
        if (bucket) {
            for (let i = 0; i < bucket.length; i++) {
                if (bucket[i][0] === key) {
                    bucket.splice(i, 1);
                    return true;
                }
            }
        }
        return false;
    }
}

// Usage
const hashTable = new SimpleHashTable();
hashTable.set('name', 'Alice');
hashTable.set('age', 30);
hashTable.set('city', 'New York');

console.log(hashTable.get('name'));  // "Alice"
console.log(hashTable.get('age'));   // 30
console.log(hashTable.get('city'));  // "New York"
```

## 5. **Password Hashing**

### **Secure Password Hashing with PBKDF2:**
```javascript
const crypto = require('crypto');

// Function to hash password
function hashPassword(password, saltRounds = 10000) {
    const salt = crypto.randomBytes(16).toString('hex');
    const hash = crypto.pbkdf2Sync(password, salt, saltRounds, 64, 'sha512').toString('hex');

    return {
        salt,
        hash,
        iterations: saltRounds
    };
}

// Function to verify password
function verifyPassword(password, storedHash, storedSalt, iterations) {
    const hash = crypto.pbkdf2Sync(password, storedSalt, iterations, 64, 'sha512').toString('hex');
    return hash === storedHash;
}

// Usage
const password = 'mySecurePassword123';

// Hash password
const hashedData = hashPassword(password);
console.log('Hashed password data:', hashedData);

// Verify password
const isValid = verifyPassword(password, hashedData.hash, hashedData.salt, hashedData.iterations);
console.log('Password verification:', isValid); // true

const isInvalid = verifyPassword('wrongPassword', hashedData.hash, hashedData.salt, hashedData.iterations);
console.log('Wrong password verification:', isInvalid); // false
```

## 6. **File Integrity Checking**

### **Calculate File Hash:**
```javascript
const crypto = require('crypto');
const fs = require('fs');

function calculateFileHash(filePath, algorithm = 'sha256') {
    return new Promise((resolve, reject) => {
        const hash = crypto.createHash(algorithm);
        const stream = fs.createReadStream(filePath);

        stream.on('data', (data) => {
            hash.update(data);
        });

        stream.on('end', () => {
            resolve(hash.digest('hex'));
        });

        stream.on('error', (error) => {
            reject(error);
        });
    });
}

// Usage
calculateFileHash('./example.txt')
    .then(hash => {
        console.log('File hash:', hash);
    })
    .catch(error => {
        console.error('Error calculating hash:', error);
    });
```

## 7. **Caching with Hash Keys**

### **URL-based Caching:**
```javascript
class URLCache {
    constructor() {
        this.cache = new Map();
    }

    generateCacheKey(url, params = {}) {
        const sortedParams = Object.keys(params)
            .sort()
            .map(key => `${key}=${params[key]}`)
            .join('&');

        const fullUrl = sortedParams ? `${url}?${sortedParams}` : url;
        return crypto.createHash('md5').update(fullUrl).digest('hex');
    }

    get(url, params = {}) {
        const key = this.generateCacheKey(url, params);
        return this.cache.get(key);
    }

    set(url, params = {}, data) {
        const key = this.generateCacheKey(url, params);
        this.cache.set(key, {
            data,
            timestamp: Date.now()
        });
    }

    clear() {
        this.cache.clear();
    }
}

// Usage
const cache = new URLCache();

// Cache API response
cache.set('/api/users', { page: 1, limit: 10 }, [{ id: 1, name: 'Alice' }]);

// Retrieve from cache
const cachedData = cache.get('/api/users', { page: 1, limit: 10 });
console.log('Cached data:', cachedData);
```

## 8. **Hash-based Data Structures**

### **Hash Set for Unique Values:**
```javascript
class HashSet {
    constructor() {
        this.items = {};
    }

    hash(value) {
        // Simple hash for demonstration
        return typeof value + ':' + String(value);
    }

    add(value) {
        const key = this.hash(value);
        this.items[key] = value;
    }

    has(value) {
        const key = this.hash(value);
        return this.items.hasOwnProperty(key);
    }

    remove(value) {
        const key = this.hash(value);
        if (this.has(value)) {
            delete this.items[key];
            return true;
        }
        return false;
    }

    values() {
        return Object.values(this.items);
    }

    size() {
        return Object.keys(this.items).length;
    }
}

// Usage
const set = new HashSet();
set.add('apple');
set.add('banana');
set.add('apple'); // Duplicate, won't be added

console.log(set.has('apple'));  // true
console.log(set.has('orange')); // false
console.log(set.values());      // ['apple', 'banana']
console.log(set.size());        // 2
```

## Real React Examples:

### 1. **Password Hashing in User Registration**
```javascript
function UserRegistration() {
    const [formData, setFormData] = useState({
        username: '',
        password: '',
        confirmPassword: ''
    });

    const hashPassword = async (password) => {
        const encoder = new TextEncoder();
        const data = encoder.encode(password);
        const hashBuffer = await crypto.subtle.digest('SHA-256', data);
        const hashArray = Array.from(new Uint8Array(hashBuffer));
        return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    };

    const handleSubmit = async (e) => {
        e.preventDefault();

        if (formData.password !== formData.confirmPassword) {
            alert('Passwords do not match');
            return;
        }

        try {
            const hashedPassword = await hashPassword(formData.password);

            // Send to server (in real app, use proper password hashing on server)
            const userData = {
                username: formData.username,
                passwordHash: hashedPassword
            };

            console.log('User registration data:', userData);
            // API call here...

        } catch (error) {
            console.error('Error hashing password:', error);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                placeholder="Username"
                value={formData.username}
                onChange={(e) => setFormData(prev => ({ ...prev, username: e.target.value }))}
                required
            />
            <input
                type="password"
                placeholder="Password"
                value={formData.password}
                onChange={(e) => setFormData(prev => ({ ...prev, password: e.target.value }))}
                required
            />
            <input
                type="password"
                placeholder="Confirm Password"
                value={formData.confirmPassword}
                onChange={(e) => setFormData(prev => ({ ...prev, confirmPassword: e.target.value }))}
                required
            />
            <button type="submit">Register</button>
        </form>
    );
}
```

### 2. **File Upload with Integrity Check**
```javascript
function FileUpload() {
    const [file, setFile] = useState(null);
    const [uploadProgress, setUploadProgress] = useState(0);

    const calculateFileHash = async (file) => {
        const buffer = await file.arrayBuffer();
        const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
        const hashArray = Array.from(new Uint8Array(hashBuffer));
        return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    };

    const handleFileChange = (e) => {
        const selectedFile = e.target.files[0];
        setFile(selectedFile);
    };

    const handleUpload = async () => {
        if (!file) return;

        try {
            // Calculate hash before upload
            const fileHash = await calculateFileHash(file);
            console.log('File hash:', fileHash);

            // Simulate upload with progress
            const formData = new FormData();
            formData.append('file', file);
            formData.append('hash', fileHash);

            // Upload logic here...
            // You can verify file integrity on server by recalculating hash

        } catch (error) {
            console.error('Error processing file:', error);
        }
    };

    return (
        <div>
            <input
                type="file"
                onChange={handleFileChange}
                accept="image/*, .pdf, .doc, .docx"
            />
            <button onClick={handleUpload} disabled={!file}>
                Upload File
            </button>
            {uploadProgress > 0 && (
                <div className="progress-bar">
                    <div
                        className="progress-fill"
                        style={{ width: `${uploadProgress}%` }}
                    ></div>
                </div>
            )}
        </div>
    );
}
```

### 3. **Caching API Responses**
```javascript
function useApiCache() {
    const [cache, setCache] = useState(new Map());

    const generateCacheKey = (url, params = {}) => {
        const sortedParams = Object.keys(params)
            .sort()
            .map(key => `${key}=${params[key]}`)
            .join('&');
        const fullUrl = sortedParams ? `${url}?${sortedParams}` : url;
        return btoa(fullUrl); // Simple encoding for demo
    };

    const fetchWithCache = async (url, params = {}) => {
        const cacheKey = generateCacheKey(url, params);
        const cached = cache.get(cacheKey);

        if (cached && Date.now() - cached.timestamp < 5 * 60 * 1000) { // 5 minutes
            return cached.data;
        }

        try {
            const queryString = new URLSearchParams(params).toString();
            const fullUrl = queryString ? `${url}?${queryString}` : url;

            const response = await fetch(fullUrl);
            const data = await response.json();

            setCache(prev => new Map(prev).set(cacheKey, {
                data,
                timestamp: Date.now()
            }));

            return data;
        } catch (error) {
            console.error('API fetch error:', error);
            throw error;
        }
    };

    const clearCache = () => {
        setCache(new Map());
    };

    return { fetchWithCache, clearCache };
}

// Usage in component
function UserList() {
    const { fetchWithCache } = useApiCache();
    const [users, setUsers] = useState([]);
    const [loading, setLoading] = useState(false);

    const loadUsers = async (page = 1, limit = 10) => {
        setLoading(true);
        try {
            const data = await fetchWithCache('/api/users', { page, limit });
            setUsers(data);
        } catch (error) {
            console.error('Error loading users:', error);
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => {
        loadUsers();
    }, []);

    return (
        <div>
            {loading ? (
                <div>Loading...</div>
            ) : (
                <ul>
                    {users.map(user => (
                        <li key={user.id}>{user.name}</li>
                    ))}
                </ul>
            )}
        </div>
    );
}
```

### 4. **Unique ID Generation**
```javascript
function generateUniqueId() {
    const timestamp = Date.now().toString();
    const random = Math.random().toString(36).substring(2);
    const combined = timestamp + random;

    // Simple hash for uniqueness
    let hash = 0;
    for (let i = 0; i < combined.length; i++) {
        const char = combined.charCodeAt(i);
        hash = ((hash << 5) - hash) + char;
        hash = hash & hash; // Convert to 32-bit integer
    }

    return Math.abs(hash).toString(36);
}

// Usage
console.log(generateUniqueId()); // "1a2b3c4d5e"
console.log(generateUniqueId()); // "6f7g8h9i0j"
```

## Security Considerations:

- **Never use simple hash functions** for passwords - they're not secure
- **Use established libraries** like bcrypt, argon2 for password hashing
- **Salt passwords** to prevent rainbow table attacks
- **Use appropriate hash algorithms** (SHA-256 for integrity, bcrypt for passwords)
- **Consider timing attacks** when comparing hashes

## Performance Considerations:

- **Hash functions are fast** but not instantaneous
- **Cache hash results** when possible
- **Choose appropriate algorithms** based on use case
- **Consider memory usage** for large hash tables

### Interview Tip:
*"Hashing converts variable-length input to fixed-length output. Use cryptographic hashes (SHA-256) for security, simple hashes for data structures. JavaScript objects are hash tables internally. Never store plain passwords - always hash them with salt."*