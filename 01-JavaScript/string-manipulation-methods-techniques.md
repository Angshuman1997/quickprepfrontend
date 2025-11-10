# String Methods

## Basic Methods

**length** - Get string length:
```javascript
"Hello".length; // 5
```

**charAt()** - Get character at index:
```javascript
"Hello".charAt(0); // "H"
```

**substring()** - Extract part of string:
```javascript
"Hello World".substring(0, 5); // "Hello"
```

**split()** - Split into array:
```javascript
"hello,world".split(","); // ["hello", "world"]
```

## Search Methods

**indexOf()** - Find position:
```javascript
"Hello World".indexOf("World"); // 6
```

**includes()** - Check if contains:
```javascript
"Hello World".includes("World"); // true
```

**startsWith()** - Check start:
```javascript
"Hello World".startsWith("Hello"); // true
```

## Transform Methods

**toUpperCase()** - Convert to uppercase:
```javascript
"hello".toUpperCase(); // "HELLO"
```

**toLowerCase()** - Convert to lowercase:
```javascript
"HELLO".toLowerCase(); // "hello"
```

**replace()** - Replace text:
```javascript
"Hello World".replace("World", "Universe"); // "Hello Universe"
```

**trim()** - Remove whitespace:
```javascript
"  hello  ".trim(); // "hello"
```

## Interview Q&A

**Q: How do you check if a string contains a substring?**  
**A:** Use includes() method: str.includes("substring")

**Q: How do you convert a string to uppercase?**  
**A:** Use toUpperCase(): str.toUpperCase()

**Q: How do you split a string into an array?**  
**A:** Use split(): str.split(",") for comma-separated values.

### **String.fromCharCode()** - Create string from Unicode values
```javascript
console.log(String.fromCharCode(72, 101, 108, 108, 111)); // "Hello"
```

## 2. **Substring Extraction**

### **slice()** - Extract portion of string
```javascript
const str = "Hello World";

// Extract from index 6 to end
console.log(str.slice(6)); // "World"

// Extract from index 0 to 5 (not including 5)
console.log(str.slice(0, 5)); // "Hello"

// Extract last 5 characters
console.log(str.slice(-5)); // "World"

// Extract from index 6 to 8
console.log(str.slice(6, 9)); // "Wor"
```

### **substring()** - Similar to slice but doesn't accept negative indices
```javascript
const str = "Hello World";

console.log(str.substring(6)); // "World"
console.log(str.substring(0, 5)); // "Hello"
console.log(str.substring(6, 9)); // "Wor"

// Negative indices are treated as 0
console.log(str.substring(-5)); // "Hello World" (same as substring(0))
```

### **substr()** - Extract substring of specific length (deprecated but still works)
```javascript
const str = "Hello World";

console.log(str.substr(6, 5)); // "World" (start at 6, length 5)
console.log(str.substr(0, 5)); // "Hello"
console.log(str.substr(-5, 5)); // "World" (negative start counts from end)
```

## 3. **Searching and Finding**

### **indexOf()** - Find first occurrence of substring
```javascript
const str = "Hello World, Hello Universe";

console.log(str.indexOf("Hello")); // 0
console.log(str.indexOf("World")); // 6
console.log(str.indexOf("hello")); // -1 (case-sensitive)
console.log(str.indexOf("Hello", 6)); // 13 (search from index 6)
```

### **lastIndexOf()** - Find last occurrence of substring
```javascript
const str = "Hello World, Hello Universe";

console.log(str.lastIndexOf("Hello")); // 13
console.log(str.lastIndexOf("World")); // 6
```

### **includes()** - Check if string contains substring
```javascript
const str = "Hello World";

console.log(str.includes("World")); // true
console.log(str.includes("world")); // false (case-sensitive)
console.log(str.includes("Hello", 1)); // false (search from index 1)
```

### **startsWith()** - Check if string starts with substring
```javascript
const str = "Hello World";

console.log(str.startsWith("Hello")); // true
console.log(str.startsWith("World")); // false
console.log(str.startsWith("ello", 1)); // true (starts at index 1)
```

### **endsWith()** - Check if string ends with substring
```javascript
const str = "Hello World";

console.log(str.endsWith("World")); // true
console.log(str.endsWith("Hello")); // false
console.log(str.endsWith("rld", 9)); // true (check up to index 9)
```

## 4. **Case Conversion**

### **toUpperCase()** - Convert to uppercase
```javascript
const str = "Hello World";
console.log(str.toUpperCase()); // "HELLO WORLD"
```

### **toLowerCase()** - Convert to lowercase
```javascript
const str = "Hello World";
console.log(str.toLowerCase()); // "hello world"
```

### **toLocaleUpperCase() / toLocaleLowerCase()** - Locale-aware case conversion
```javascript
const str = "istanbul";
console.log(str.toUpperCase()); // "ISTANBUL"
console.log(str.toLocaleUpperCase('tr-TR')); // "İSTANBUL" (Turkish i)
```

## 5. **Trimming and Padding**

### **trim()** - Remove whitespace from both ends
```javascript
const str = "  Hello World  ";
console.log(str.trim()); // "Hello World"
```

### **trimStart() / trimLeft()** - Remove whitespace from start
```javascript
const str = "  Hello World  ";
console.log(str.trimStart()); // "Hello World  "
```

### **trimEnd() / trimRight()** - Remove whitespace from end
```javascript
const str = "  Hello World  ";
console.log(str.trimEnd()); // "  Hello World"
```

### **padStart()** - Pad string to target length from start
```javascript
const str = "5";
console.log(str.padStart(3, "0")); // "005"
console.log(str.padStart(5, "*")); // "****5"
```

### **padEnd()** - Pad string to target length from end
```javascript
const str = "5";
console.log(str.padEnd(3, "0")); // "500"
console.log(str.padEnd(5, "*")); // "5****"
```

## 6. **Replacement and Splitting**

### **replace()** - Replace first occurrence
```javascript
const str = "Hello World, Hello Universe";

console.log(str.replace("Hello", "Hi")); // "Hi World, Hello Universe"
console.log(str.replace(/Hello/g, "Hi")); // "Hi World, Hi Universe" (global replace)
```

### **replaceAll()** - Replace all occurrences
```javascript
const str = "Hello World, Hello Universe";

console.log(str.replaceAll("Hello", "Hi")); // "Hi World, Hi Universe"
```

### **split()** - Split string into array
```javascript
const str = "apple,banana,orange";

console.log(str.split(",")); // ["apple", "banana", "orange"]
console.log(str.split(",", 2)); // ["apple", "banana"] (limit to 2 elements)

const sentence = "Hello World";
console.log(sentence.split(" ")); // ["Hello", "World"]
console.log(sentence.split("")); // ["H", "e", "l", "l", "o", " ", "W", "o", "r", "l", "d"]
```

## 7. **Concatenation and Repeating**

### **concat()** - Concatenate strings
```javascript
const str1 = "Hello";
const str2 = "World";

console.log(str1.concat(" ", str2)); // "Hello World"
console.log(str1 + " " + str2); // "Hello World" (more common)
```

### **repeat()** - Repeat string
```javascript
const str = "Ha";
console.log(str.repeat(3)); // "HaHaHa"

console.log("*".repeat(5)); // "*****"
```

## 8. **Advanced Techniques**

### **Template Literals** - Modern string interpolation
```javascript
const name = "Alice";
const age = 30;

// Old way
const greeting = "Hello, my name is " + name + " and I am " + age + " years old.";

// New way (template literals)
const greeting2 = `Hello, my name is ${name} and I am ${age} years old.`;

console.log(greeting2); // "Hello, my name is Alice and I am 30 years old."

// Multi-line strings
const multiLine = `
  This is a
  multi-line string
  with proper formatting
`;
```

### **String Destructuring**
```javascript
const str = "Hello";
const [first, second, ...rest] = str;

console.log(first);  // "H"
console.log(second); // "e"
console.log(rest);   // ["l", "l", "o"]
```

### **Regular Expressions with Strings**
```javascript
const str = "The quick brown fox jumps over the lazy dog";

// Test if string matches pattern
console.log(/fox/.test(str)); // true

// Find all matches
const matches = str.match(/the/gi); // ["The", "the"] (case-insensitive, global)
console.log(matches);

// Replace with regex
const result = str.replace(/the/gi, "a");
console.log(result); // "a quick brown fox jumps over a lazy dog"

// Split with regex
const words = str.split(/\s+/);
console.log(words); // ["The", "quick", "brown", "fox", "jumps", "over", "the", "lazy", "dog"]
```

## Real React Examples:

### 1. **Form Input Validation and Formatting**
```javascript
function UserForm() {
    const [formData, setFormData] = useState({
        name: '',
        email: '',
        phone: ''
    });

    const formatName = (name) => {
        // Capitalize first letter of each word
        return name
            .toLowerCase()
            .split(' ')
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(' ');
    };

    const formatPhone = (phone) => {
        // Remove all non-digits
        const digits = phone.replace(/\D/g, '');

        // Format as (XXX) XXX-XXXX
        if (digits.length >= 10) {
            return `(${digits.slice(0, 3)}) ${digits.slice(3, 6)}-${digits.slice(6, 10)}`;
        }
        return digits;
    };

    const validateEmail = (email) => {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email);
    };

    const handleChange = (field, value) => {
        let formattedValue = value;

        if (field === 'name') {
            formattedValue = formatName(value);
        } else if (field === 'phone') {
            formattedValue = formatPhone(value);
        }

        setFormData(prev => ({
            ...prev,
            [field]: formattedValue
        }));
    };

    const handleSubmit = (e) => {
        e.preventDefault();

        if (!validateEmail(formData.email)) {
            alert('Please enter a valid email');
            return;
        }

        console.log('Form submitted:', formData);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                placeholder="Full Name"
                value={formData.name}
                onChange={(e) => handleChange('name', e.target.value)}
            />
            <input
                type="email"
                placeholder="Email"
                value={formData.email}
                onChange={(e) => handleChange('email', e.target.value)}
            />
            <input
                type="tel"
                placeholder="Phone"
                value={formData.phone}
                onChange={(e) => handleChange('phone', e.target.value)}
            />
            <button type="submit">Submit</button>
        </form>
    );
}
```

### 2. **Search and Filter Functionality**
```javascript
function ProductList({ products }) {
    const [searchTerm, setSearchTerm] = useState('');
    const [sortBy, setSortBy] = useState('name');

    // Filter products based on search term
    const filteredProducts = products.filter(product => {
        const term = searchTerm.toLowerCase();
        return product.name.toLowerCase().includes(term) ||
               product.description.toLowerCase().includes(term) ||
               product.category.toLowerCase().includes(term);
    });

    // Sort products
    const sortedProducts = [...filteredProducts].sort((a, b) => {
        if (sortBy === 'name') {
            return a.name.localeCompare(b.name);
        } else if (sortBy === 'price') {
            return a.price - b.price;
        }
        return 0;
    });

    // Highlight search terms in results
    const highlightText = (text, highlight) => {
        if (!highlight.trim()) return text;

        const regex = new RegExp(`(${highlight.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
        const parts = text.split(regex);

        return parts.map((part, index) =>
            regex.test(part) ? <mark key={index}>{part}</mark> : part
        );
    };

    return (
        <div>
            <div className="controls">
                <input
                    type="text"
                    placeholder="Search products..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                />
                <select value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
                    <option value="name">Sort by Name</option>
                    <option value="price">Sort by Price</option>
                </select>
            </div>

            <div className="products">
                {sortedProducts.map(product => (
                    <div key={product.id} className="product">
                        <h3>{highlightText(product.name, searchTerm)}</h3>
                        <p>{highlightText(product.description, searchTerm)}</p>
                        <span className="price">${product.price}</span>
                        <span className="category">{product.category}</span>
                    </div>
                ))}
            </div>

            <p>Showing {sortedProducts.length} of {products.length} products</p>
        </div>
    );
}
```

### 3. **URL and Path Manipulation**
```javascript
function Breadcrumb({ currentPath }) {
    // Convert path to breadcrumb segments
    const segments = currentPath
        .split('/')
        .filter(segment => segment.length > 0);

    // Create breadcrumb items
    const breadcrumbs = segments.map((segment, index) => {
        const path = '/' + segments.slice(0, index + 1).join('/');
        const label = segment
            .split('-')
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(' ');

        return { path, label };
    });

    // Add home breadcrumb
    breadcrumbs.unshift({ path: '/', label: 'Home' });

    return (
        <nav aria-label="breadcrumb">
            <ol className="breadcrumb">
                {breadcrumbs.map((crumb, index) => (
                    <li key={crumb.path} className="breadcrumb-item">
                        {index === breadcrumbs.length - 1 ? (
                            <span>{crumb.label}</span>
                        ) : (
                            <a href={crumb.path}>{crumb.label}</a>
                        )}
                    </li>
                ))}
            </ol>
        </nav>
    );
}

// Usage
<Breadcrumb currentPath="/products/electronics/laptops/dell-xps-13" />
```

### 4. **Text Processing and Formatting**
```javascript
function TextProcessor() {
    const [text, setText] = useState('');
    const [processedText, setProcessedText] = useState('');

    const processText = (inputText, operation) => {
        switch (operation) {
            case 'uppercase':
                return inputText.toUpperCase();

            case 'lowercase':
                return inputText.toLowerCase();

            case 'titlecase':
                return inputText
                    .toLowerCase()
                    .split(' ')
                    .map(word => word.charAt(0).toUpperCase() + word.slice(1))
                    .join(' ');

            case 'reverse':
                return inputText.split('').reverse().join('');

            case 'remove-spaces':
                return inputText.replace(/\s+/g, '');

            case 'word-count':
                const words = inputText.trim().split(/\s+/).filter(word => word.length > 0);
                return `${words.length} words`;

            case 'character-count':
                return `${inputText.length} characters`;

            case 'extract-numbers':
                return inputText.replace(/[^0-9]/g, '');

            case 'extract-letters':
                return inputText.replace(/[^a-zA-Z]/g, '');

            default:
                return inputText;
        }
    };

    const handleProcess = (operation) => {
        setProcessedText(processText(text, operation));
    };

    return (
        <div>
            <textarea
                value={text}
                onChange={(e) => setText(e.target.value)}
                placeholder="Enter text to process..."
                rows={6}
            />

            <div className="buttons">
                <button onClick={() => handleProcess('uppercase')}>Uppercase</button>
                <button onClick={() => handleProcess('lowercase')}>Lowercase</button>
                <button onClick={() => handleProcess('titlecase')}>Title Case</button>
                <button onClick={() => handleProcess('reverse')}>Reverse</button>
                <button onClick={() => handleProcess('remove-spaces')}>Remove Spaces</button>
                <button onClick={() => handleProcess('word-count')}>Word Count</button>
                <button onClick={() => handleProcess('character-count')}>Character Count</button>
                <button onClick={() => handleProcess('extract-numbers')}>Extract Numbers</button>
                <button onClick={() => handleProcess('extract-letters')}>Extract Letters</button>
            </div>

            <div className="result">
                <h3>Result:</h3>
                <pre>{processedText}</pre>
            </div>
        </div>
    );
}
```

## Performance Considerations:

- **String concatenation** with `+` is optimized in modern JavaScript engines
- **Template literals** are preferred for complex string building
- **Regular expressions** are powerful but can be expensive for large strings
- **String methods** create new strings - consider memory usage for large operations
- **Caching** results of expensive string operations can improve performance

## Common Patterns:

1. **URL slug creation**: `str.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '')`
2. **Email validation**: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
3. **Phone number formatting**: Remove non-digits, then format with substrings
4. **Truncate text**: `str.length > limit ? str.slice(0, limit) + '...' : str`
5. **Remove HTML tags**: `str.replace(/<[^>]*>/g, '')`

### Interview Tip:
*"JavaScript strings are immutable, so methods return new strings. Use slice/substring for extraction, indexOf/includes for searching, and replace/replaceAll for substitution. Template literals are the modern way to build complex strings with interpolation."*