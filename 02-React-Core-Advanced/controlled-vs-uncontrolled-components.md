# Controlled vs Uncontrolled components — when do you use each?

## Question
Controlled vs Uncontrolled components — when do you use each?

## Answer

In React, components can be classified as **controlled** or **uncontrolled** based on how they manage form input elements and their state. This distinction is crucial for understanding how React handles user input and form data.

## Controlled Components

**Controlled components** are components where React controls the input's value through state. The component's state is the single source of truth.

### Characteristics:
- **State-driven**: Input value comes from component state
- **Predictable**: React controls all changes
- **Validation**: Easy to implement real-time validation
- **Testing**: Easier to test due to predictable state

### Basic Example:

```javascript
function ControlledInput() {
    const [value, setValue] = useState('');

    const handleChange = (event) => {
        setValue(event.target.value);
    };

    return (
        <div>
            <input
                type="text"
                value={value}
                onChange={handleChange}
                placeholder="Type something..."
            />
            <p>You typed: {value}</p>
        </div>
    );
}
```

### Advanced Controlled Component:

```javascript
function ControlledForm() {
    const [formData, setFormData] = useState({
        name: '',
        email: '',
        age: ''
    });

    const [errors, setErrors] = useState({});

    const handleChange = (event) => {
        const { name, value } = event.target;

        // Update form data
        setFormData(prev => ({
            ...prev,
            [name]: value
        }));

        // Clear error when user starts typing
        if (errors[name]) {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    };

    const validateForm = () => {
        const newErrors = {};

        if (!formData.name.trim()) {
            newErrors.name = 'Name is required';
        }

        if (!formData.email.trim()) {
            newErrors.email = 'Email is required';
        } else if (!/\S+@\S+\.\S+/.test(formData.email)) {
            newErrors.email = 'Email is invalid';
        }

        if (!formData.age || formData.age < 0) {
            newErrors.age = 'Age must be a positive number';
        }

        setErrors(newErrors);
        return Object.keys(newErrors).length === 0;
    };

    const handleSubmit = (event) => {
        event.preventDefault();
        if (validateForm()) {
            console.log('Form submitted:', formData);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <div>
                <input
                    type="text"
                    name="name"
                    value={formData.name}
                    onChange={handleChange}
                    placeholder="Name"
                />
                {errors.name && <span className="error">{errors.name}</span>}
            </div>

            <div>
                <input
                    type="email"
                    name="email"
                    value={formData.email}
                    onChange={handleChange}
                    placeholder="Email"
                />
                {errors.email && <span className="error">{errors.email}</span>}
            </div>

            <div>
                <input
                    type="number"
                    name="age"
                    value={formData.age}
                    onChange={handleChange}
                    placeholder="Age"
                />
                {errors.age && <span className="error">{errors.age}</span>}
            </div>

            <button type="submit">Submit</button>
        </form>
    );
}
```

## Uncontrolled Components

**Uncontrolled components** are components where the DOM itself manages the input's state. React doesn't control the value directly.

### Characteristics:
- **DOM-driven**: Input value managed by DOM
- **Refs**: Use `useRef` to access DOM values
- **Performance**: Slightly better for simple cases
- **Integration**: Good for integrating with non-React code

### Basic Example:

```javascript
function UncontrolledInput() {
    const inputRef = useRef(null);

    const handleSubmit = (event) => {
        event.preventDefault();
        const value = inputRef.current.value;
        console.log('Input value:', value);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                ref={inputRef}
                defaultValue="Default value"
                placeholder="Type something..."
            />
            <button type="submit">Submit</button>
        </form>
    );
}
```

### Advanced Uncontrolled Component:

```javascript
function UncontrolledFileUpload() {
    const fileRef = useRef(null);
    const [selectedFile, setSelectedFile] = useState(null);

    const handleFileChange = () => {
        const file = fileRef.current.files[0];
        setSelectedFile(file);
    };

    const handleSubmit = (event) => {
        event.preventDefault();
        if (selectedFile) {
            console.log('Uploading file:', selectedFile.name);
            // Upload logic here
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="file"
                ref={fileRef}
                onChange={handleFileChange}
                accept="image/*"
            />
            {selectedFile && (
                <div>
                    <p>Selected: {selectedFile.name}</p>
                    <p>Size: {(selectedFile.size / 1024).toFixed(2)} KB</p>
                </div>
            )}
            <button type="submit" disabled={!selectedFile}>
                Upload
            </button>
        </form>
    );
}
```

## Key Differences

| Aspect | Controlled Components | Uncontrolled Components |
|--------|----------------------|------------------------|
| **State Management** | React state | DOM state |
| **Value Source** | `value` prop from state | `defaultValue` prop |
| **Updates** | `onChange` handler updates state | DOM handles updates |
| **Access** | Through state variables | Through refs |
| **Validation** | Real-time, immediate | On form submission |
| **Performance** | Slightly slower (re-renders) | Slightly faster |
| **Testing** | Easier to test | Harder to test |
| **Integration** | Pure React | Third-party libraries |

## When to Use Each

### Use Controlled Components When:
- Need real-time validation
- Form data needs to be controlled by React
- Need to transform input values
- Component state affects other parts of UI
- Need to enforce specific input formats
- Building complex forms with dependencies

### Use Uncontrolled Components When:
- Simple forms without validation
- Integrating with non-React code
- File uploads (uncontrolled by nature)
- Performance-critical scenarios
- Working with third-party DOM libraries

## Hybrid Approach

Sometimes you might want to mix both approaches:

```javascript
function HybridForm() {
    const [email, setEmail] = useState('');
    const passwordRef = useRef(null);

    const handleEmailChange = (event) => {
        setEmail(event.target.value);
    };

    const handleSubmit = (event) => {
        event.preventDefault();
        const password = passwordRef.current.value;

        // Email is controlled, password is uncontrolled
        console.log('Email:', email);
        console.log('Password:', password);
    };

    return (
        <form onSubmit={handleSubmit}>
            {/* Controlled input */}
            <input
                type="email"
                value={email}
                onChange={handleEmailChange}
                placeholder="Email (controlled)"
            />

            {/* Uncontrolled input */}
            <input
                type="password"
                ref={passwordRef}
                placeholder="Password (uncontrolled)"
            />

            <button type="submit">Login</button>
        </form>
    );
}
```

## Real React Examples:

### 1. **Advanced Form with Controlled Components**

```javascript
function AdvancedForm() {
    const [formData, setFormData] = useState({
        firstName: '',
        lastName: '',
        email: '',
        phone: '',
        country: 'US',
        newsletter: false
    });

    const [touched, setTouched] = useState({});
    const [errors, setErrors] = useState({});

    const validationRules = {
        firstName: (value) => !value.trim() ? 'First name is required' : '',
        lastName: (value) => !value.trim() ? 'Last name is required' : '',
        email: (value) => {
            if (!value.trim()) return 'Email is required';
            if (!/\S+@\S+\.\S+/.test(value)) return 'Invalid email format';
            return '';
        },
        phone: (value) => {
            if (!value.trim()) return '';
            if (!/^\+?[\d\s\-\(\)]+$/.test(value)) return 'Invalid phone format';
            return '';
        }
    };

    const handleChange = (event) => {
        const { name, value, type, checked } = event.target;

        setFormData(prev => ({
            ...prev,
            [name]: type === 'checkbox' ? checked : value
        }));

        // Clear error when user starts correcting
        if (errors[name]) {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    };

    const handleBlur = (event) => {
        const { name } = event.target;

        setTouched(prev => ({
            ...prev,
            [name]: true
        }));

        // Validate on blur
        const error = validationRules[name]?.(formData[name]);
        if (error) {
            setErrors(prev => ({
                ...prev,
                [name]: error
            }));
        }
    };

    const handleSubmit = (event) => {
        event.preventDefault();

        // Validate all fields
        const newErrors = {};
        Object.keys(validationRules).forEach(field => {
            const error = validationRules[field](formData[field]);
            if (error) newErrors[field] = error;
        });

        setErrors(newErrors);
        setTouched(Object.keys(formData).reduce((acc, key) => ({ ...acc, [key]: true }), {}));

        if (Object.keys(newErrors).length === 0) {
            console.log('Form submitted:', formData);
        }
    };

    const isFieldValid = (fieldName) => {
        return touched[fieldName] && !errors[fieldName];
    };

    const isFieldInvalid = (fieldName) => {
        return touched[fieldName] && errors[fieldName];
    };

    return (
        <form onSubmit={handleSubmit} className="advanced-form">
            <div className="form-row">
                <div className="form-group">
                    <label>First Name *</label>
                    <input
                        type="text"
                        name="firstName"
                        value={formData.firstName}
                        onChange={handleChange}
                        onBlur={handleBlur}
                        className={isFieldInvalid('firstName') ? 'error' : isFieldValid('firstName') ? 'success' : ''}
                    />
                    {errors.firstName && <span className="error-text">{errors.firstName}</span>}
                </div>

                <div className="form-group">
                    <label>Last Name *</label>
                    <input
                        type="text"
                        name="lastName"
                        value={formData.lastName}
                        onChange={handleChange}
                        onBlur={handleBlur}
                        className={isFieldInvalid('lastName') ? 'error' : isFieldValid('lastName') ? 'success' : ''}
                    />
                    {errors.lastName && <span className="error-text">{errors.lastName}</span>}
                </div>
            </div>

            <div className="form-group">
                <label>Email *</label>
                <input
                    type="email"
                    name="email"
                    value={formData.email}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    className={isFieldInvalid('email') ? 'error' : isFieldValid('email') ? 'success' : ''}
                />
                {errors.email && <span className="error-text">{errors.email}</span>}
            </div>

            <div className="form-group">
                <label>Phone</label>
                <input
                    type="tel"
                    name="phone"
                    value={formData.phone}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="+1 (555) 123-4567"
                    className={isFieldInvalid('phone') ? 'error' : isFieldValid('phone') ? 'success' : ''}
                />
                {errors.phone && <span className="error-text">{errors.phone}</span>}
            </div>

            <div className="form-group">
                <label>Country</label>
                <select
                    name="country"
                    value={formData.country}
                    onChange={handleChange}
                >
                    <option value="US">United States</option>
                    <option value="CA">Canada</option>
                    <option value="UK">United Kingdom</option>
                    <option value="AU">Australia</option>
                </select>
            </div>

            <div className="form-group checkbox">
                <label>
                    <input
                        type="checkbox"
                        name="newsletter"
                        checked={formData.newsletter}
                        onChange={handleChange}
                    />
                    Subscribe to newsletter
                </label>
            </div>

            <button type="submit" className="submit-btn">
                Create Account
            </button>
        </form>
    );
}
```

### 2. **Uncontrolled File Upload with Preview**

```javascript
function FileUploadWithPreview() {
    const fileRef = useRef(null);
    const [preview, setPreview] = useState(null);
    const [uploading, setUploading] = useState(false);

    const handleFileSelect = () => {
        const file = fileRef.current.files[0];
        if (file && file.type.startsWith('image/')) {
            const reader = new FileReader();
            reader.onload = (e) => setPreview(e.target.result);
            reader.readAsDataURL(file);
        }
    };

    const handleUpload = async () => {
        const file = fileRef.current.files[0];
        if (!file) return;

        setUploading(true);
        try {
            const formData = new FormData();
            formData.append('file', file);

            const response = await fetch('/api/upload', {
                method: 'POST',
                body: formData
            });

            if (response.ok) {
                alert('File uploaded successfully!');
                // Reset form
                fileRef.current.value = '';
                setPreview(null);
            } else {
                throw new Error('Upload failed');
            }
        } catch (error) {
            alert('Upload failed: ' + error.message);
        } finally {
            setUploading(false);
        }
    };

    const clearFile = () => {
        fileRef.current.value = '';
        setPreview(null);
    };

    return (
        <div className="file-upload">
            <input
                type="file"
                ref={fileRef}
                onChange={handleFileSelect}
                accept="image/*"
                style={{ display: 'none' }}
            />

            {!preview ? (
                <button
                    onClick={() => fileRef.current.click()}
                    className="upload-btn"
                >
                    Choose Image
                </button>
            ) : (
                <div className="preview-container">
                    <img src={preview} alt="Preview" className="preview" />
                    <div className="preview-actions">
                        <button onClick={clearFile} disabled={uploading}>
                            Remove
                        </button>
                        <button onClick={handleUpload} disabled={uploading}>
                            {uploading ? 'Uploading...' : 'Upload'}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
}
```

### 3. **Controlled Search Component**

```javascript
function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);
    const [debouncedQuery, setDebouncedQuery] = useState('');

    // Debounce search query
    useEffect(() => {
        const timer = setTimeout(() => {
            setDebouncedQuery(query);
        }, 300);

        return () => clearTimeout(timer);
    }, [query]);

    // Perform search when debounced query changes
    useEffect(() => {
        if (debouncedQuery.trim()) {
            performSearch(debouncedQuery);
        } else {
            setResults([]);
        }
    }, [debouncedQuery]);

    const performSearch = async (searchQuery) => {
        setLoading(true);
        try {
            const response = await fetch(`/api/search?q=${encodeURIComponent(searchQuery)}`);
            const data = await response.json();
            setResults(data.results);
        } catch (error) {
            console.error('Search failed:', error);
            setResults([]);
        } finally {
            setLoading(false);
        }
    };

    const handleClear = () => {
        setQuery('');
        setResults([]);
    };

    return (
        <div className="search-component">
            <div className="search-input-container">
                <input
                    type="text"
                    value={query}
                    onChange={(e) => setQuery(e.target.value)}
                    placeholder="Search..."
                    className="search-input"
                />
                {query && (
                    <button onClick={handleClear} className="clear-btn">
                        ×
                    </button>
                )}
            </div>

            {loading && <div className="loading">Searching...</div>}

            {results.length > 0 && (
                <div className="search-results">
                    <h4>Results for "{debouncedQuery}"</h4>
                    {results.map((result, index) => (
                        <div key={index} className="result-item">
                            <h5>{result.title}</h5>
                            <p>{result.description}</p>
                        </div>
                    ))}
                </div>
            )}

            {query && !loading && results.length === 0 && (
                <div className="no-results">
                    No results found for "{query}"
                </div>
            )}
        </div>
    );
}
```

### 4. **Controlled vs Uncontrolled Comparison**

```javascript
function ComparisonDemo() {
    const [controlledValue, setControlledValue] = useState('');
    const uncontrolledRef = useRef(null);

    const handleControlledChange = (event) => {
        setControlledValue(event.target.value.toUpperCase()); // Transform input
    };

    const compareValues = () => {
        const uncontrolledValue = uncontrolledRef.current.value;
        console.log('Controlled:', controlledValue);
        console.log('Uncontrolled:', uncontrolledValue);
        console.log('Are they equal?', controlledValue === uncontrolledValue);
    };

    return (
        <div className="comparison-demo">
            <h3>Controlled vs Uncontrolled Comparison</h3>

            <div className="input-group">
                <label>Controlled Input (transforms to uppercase):</label>
                <input
                    type="text"
                    value={controlledValue}
                    onChange={handleControlledChange}
                    placeholder="Type here..."
                />
                <p>Current value: {controlledValue}</p>
            </div>

            <div className="input-group">
                <label>Uncontrolled Input (raw input):</label>
                <input
                    type="text"
                    ref={uncontrolledRef}
                    placeholder="Type here..."
                />
            </div>

            <button onClick={compareValues}>Compare Values</button>
        </div>
    );
}
```

## Best Practices

### For Controlled Components:
1. **Always provide onChange handler** with setState
2. **Use controlled components for forms** that need validation
3. **Consider performance** for large forms (might cause re-renders)
4. **Use for complex logic** like input transformation

### For Uncontrolled Components:
1. **Use refs to access values** when needed
2. **Good for simple forms** without validation
3. **Better performance** for large forms
4. **Use defaultValue** instead of value for initial values

### General Tips:
- **Mix approaches** when appropriate
- **Test thoroughly** - uncontrolled components are harder to test
- **Consider user experience** - controlled components provide better UX
- **Use libraries** like Formik or React Hook Form for complex forms

### Interview Tip:
*"Controlled components give React full control over form state, enabling real-time validation and predictable behavior. Uncontrolled components let the DOM manage state, which can be simpler but harder to test. Choose controlled components when you need validation, transformations, or complex form logic."*