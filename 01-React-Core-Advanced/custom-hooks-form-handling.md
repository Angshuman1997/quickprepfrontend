# Custom hooks for form handling - best practices

## Question
Custom hooks for form handling - best practices

## Answer

Custom hooks for form handling are essential in React applications for managing form state, validation, and submission logic. They help reduce code duplication and provide a clean, reusable way to handle complex form interactions.

## Basic Form Hook

### Simple form state management

```javascript
import { useState, useCallback } from 'react';

function useForm(initialValues = {}) {
    const [values, setValues] = useState(initialValues);
    const [errors, setErrors] = useState({});
    const [touched, setTouched] = useState({});
    const [isSubmitting, setIsSubmitting] = useState(false);

    const handleChange = useCallback((event) => {
        const { name, value, type, checked } = event.target;

        // Handle different input types
        const inputValue = type === 'checkbox' ? checked : value;

        setValues(prev => ({
            ...prev,
            [name]: inputValue
        }));

        // Clear error when user starts typing
        if (errors[name]) {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    }, [errors]);

    const handleBlur = useCallback((event) => {
        const { name } = event.target;
        setTouched(prev => ({
            ...prev,
            [name]: true
        }));
    }, []);

    const handleSubmit = useCallback((onSubmit) => {
        return async (event) => {
            event.preventDefault();
            setIsSubmitting(true);

            try {
                await onSubmit(values);
                // Reset form on success
                setValues(initialValues);
                setErrors({});
                setTouched({});
            } catch (error) {
                // Handle submission errors
                console.error('Form submission failed:', error);
            } finally {
                setIsSubmitting(false);
            }
        };
    }, [values, initialValues]);

    const setValue = useCallback((name, value) => {
        setValues(prev => ({
            ...prev,
            [name]: value
        }));
    }, []);

    const setError = useCallback((name, error) => {
        setErrors(prev => ({
            ...prev,
            [name]: error
        }));
    }, []);

    const reset = useCallback(() => {
        setValues(initialValues);
        setErrors({});
        setTouched({});
        setIsSubmitting(false);
    }, [initialValues]);

    return {
        values,
        errors,
        touched,
        isSubmitting,
        handleChange,
        handleBlur,
        handleSubmit,
        setValue,
        setError,
        reset
    };
}

// Usage
function LoginForm() {
    const {
        values,
        errors,
        touched,
        isSubmitting,
        handleChange,
        handleBlur,
        handleSubmit
    } = useForm({
        email: '',
        password: ''
    });

    const onSubmit = async (formData) => {
        // Simulate API call
        await new Promise(resolve => setTimeout(resolve, 1000));
        console.log('Login attempt:', formData);
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <div>
                <input
                    type="email"
                    name="email"
                    value={values.email}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Email"
                />
                {touched.email && errors.email && <span>{errors.email}</span>}
            </div>

            <div>
                <input
                    type="password"
                    name="password"
                    value={values.password}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Password"
                />
                {touched.password && errors.password && <span>{errors.password}</span>}
            </div>

            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? 'Logging in...' : 'Login'}
            </button>
        </form>
    );
}
```

## Advanced Form Hook with Validation

### Hook with built-in validation

```javascript
function useFormWithValidation(initialValues = {}, validationRules = {}) {
    const [values, setValues] = useState(initialValues);
    const [errors, setErrors] = useState({});
    const [touched, setTouched] = useState({});
    const [isSubmitting, setIsSubmitting] = useState(false);
    const [isValid, setIsValid] = useState(false);

    // Validate single field
    const validateField = useCallback((name, value) => {
        const rules = validationRules[name];
        if (!rules) return '';

        if (rules.required && !value) {
            return `${name} is required`;
        }

        if (rules.minLength && value.length < rules.minLength) {
            return `${name} must be at least ${rules.minLength} characters`;
        }

        if (rules.maxLength && value.length > rules.maxLength) {
            return `${name} must be no more than ${rules.maxLength} characters`;
        }

        if (rules.pattern && !rules.pattern.test(value)) {
            return rules.message || `${name} format is invalid`;
        }

        if (rules.custom) {
            return rules.custom(value, values);
        }

        return '';
    }, [validationRules, values]);

    // Validate all fields
    const validateForm = useCallback(() => {
        const newErrors = {};
        let formIsValid = true;

        Object.keys(validationRules).forEach(name => {
            const error = validateField(name, values[name]);
            if (error) {
                newErrors[name] = error;
                formIsValid = false;
            }
        });

        setErrors(newErrors);
        setIsValid(formIsValid);
        return formIsValid;
    }, [values, validationRules, validateField]);

    // Update validation when values change
    useEffect(() => {
        validateForm();
    }, [validateForm]);

    const handleChange = useCallback((event) => {
        const { name, value, type, checked } = event.target;
        const inputValue = type === 'checkbox' ? checked : value;

        setValues(prev => ({
            ...prev,
            [name]: inputValue
        }));

        // Clear error for this field
        if (errors[name]) {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    }, [errors]);

    const handleBlur = useCallback((event) => {
        const { name } = event.target;
        setTouched(prev => ({
            ...prev,
            [name]: true
        }));

        // Validate field on blur
        const error = validateField(name, values[name]);
        if (error) {
            setErrors(prev => ({
                ...prev,
                [name]: error
            }));
        }
    }, [validateField, values]);

    const handleSubmit = useCallback((onSubmit) => {
        return async (event) => {
            event.preventDefault();

            if (!validateForm()) {
                // Mark all fields as touched to show errors
                setTouched(Object.keys(values).reduce((acc, key) => ({
                    ...acc,
                    [key]: true
                }), {}));
                return;
            }

            setIsSubmitting(true);

            try {
                await onSubmit(values);
                // Reset form on success
                setValues(initialValues);
                setErrors({});
                setTouched({});
                setIsValid(false);
            } catch (error) {
                console.error('Form submission failed:', error);
            } finally {
                setIsSubmitting(false);
            }
        };
    }, [values, initialValues, validateForm]);

    const setValue = useCallback((name, value) => {
        setValues(prev => ({
            ...prev,
            [name]: value
        }));
    }, []);

    const reset = useCallback(() => {
        setValues(initialValues);
        setErrors({});
        setTouched({});
        setIsSubmitting(false);
        setIsValid(false);
    }, [initialValues]);

    return {
        values,
        errors,
        touched,
        isSubmitting,
        isValid,
        handleChange,
        handleBlur,
        handleSubmit,
        setValue,
        reset,
        validateForm
    };
}

// Usage with validation
function RegistrationForm() {
    const validationRules = {
        name: { required: true, minLength: 2, maxLength: 50 },
        email: {
            required: true,
            pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'Please enter a valid email address'
        },
        password: { required: true, minLength: 8 },
        confirmPassword: {
            required: true,
            custom: (value, allValues) => {
                if (value !== allValues.password) {
                    return 'Passwords do not match';
                }
                return '';
            }
        }
    };

    const {
        values,
        errors,
        touched,
        isSubmitting,
        isValid,
        handleChange,
        handleBlur,
        handleSubmit
    } = useFormWithValidation({
        name: '',
        email: '',
        password: '',
        confirmPassword: ''
    }, validationRules);

    const onSubmit = async (formData) => {
        await new Promise(resolve => setTimeout(resolve, 1000));
        console.log('Registration:', formData);
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <div>
                <input
                    type="text"
                    name="name"
                    value={values.name}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Full Name"
                />
                {touched.name && errors.name && <span className="error">{errors.name}</span>}
            </div>

            <div>
                <input
                    type="email"
                    name="email"
                    value={values.email}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Email"
                />
                {touched.email && errors.email && <span className="error">{errors.email}</span>}
            </div>

            <div>
                <input
                    type="password"
                    name="password"
                    value={values.password}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Password"
                />
                {touched.password && errors.password && <span className="error">{errors.password}</span>}
            </div>

            <div>
                <input
                    type="password"
                    name="confirmPassword"
                    value={values.confirmPassword}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Confirm Password"
                />
                {touched.confirmPassword && errors.confirmPassword && <span className="error">{errors.confirmPassword}</span>}
            </div>

            <button type="submit" disabled={isSubmitting || !isValid}>
                {isSubmitting ? 'Registering...' : 'Register'}
            </button>
        </form>
    );
}
```

## Form Hook with Field Arrays

### Hook for dynamic form fields

```javascript
function useFieldArray(initialFields = []) {
    const [fields, setFields] = useState(initialFields);

    const addField = useCallback((field = {}) => {
        const newField = {
            id: Date.now(),
            ...field
        };
        setFields(prev => [...prev, newField]);
    }, []);

    const removeField = useCallback((id) => {
        setFields(prev => prev.filter(field => field.id !== id));
    }, []);

    const updateField = useCallback((id, updates) => {
        setFields(prev => prev.map(field =>
            field.id === id ? { ...field, ...updates } : field
        ));
    }, []);

    const moveField = useCallback((fromIndex, toIndex) => {
        setFields(prev => {
            const newFields = [...prev];
            const [movedField] = newFields.splice(fromIndex, 1);
            newFields.splice(toIndex, 0, movedField);
            return newFields;
        });
    }, []);

    const resetFields = useCallback(() => {
        setFields(initialFields);
    }, [initialFields]);

    return {
        fields,
        addField,
        removeField,
        updateField,
        moveField,
        resetFields
    };
}

// Usage
function DynamicForm() {
    const { fields, addField, removeField, updateField } = useFieldArray([
        { id: 1, name: '', email: '' }
    ]);

    const handleFieldChange = (id, field, value) => {
        updateField(id, { [field]: value });
    };

    return (
        <div>
            {fields.map((field, index) => (
                <div key={field.id} className="field-group">
                    <input
                        type="text"
                        placeholder="Name"
                        value={field.name}
                        onChange={(e) => handleFieldChange(field.id, 'name', e.target.value)}
                    />
                    <input
                        type="email"
                        placeholder="Email"
                        value={field.email}
                        onChange={(e) => handleFieldChange(field.id, 'email', e.target.value)}
                    />
                    <button onClick={() => removeField(field.id)}>Remove</button>
                </div>
            ))}

            <button onClick={() => addField({ name: '', email: '' })}>
                Add Person
            </button>
        </div>
    );
}
```

## Form Hook with Async Validation

### Hook with debounced async validation

```javascript
function useAsyncValidation(initialValues = {}, asyncValidators = {}) {
    const [values, setValues] = useState(initialValues);
    const [errors, setErrors] = useState({});
    const [validating, setValidating] = useState({});
    const [touched, setTouched] = useState({});

    const validateFieldAsync = useCallback(async (name, value) => {
        const validator = asyncValidators[name];
        if (!validator) return null;

        try {
            setValidating(prev => ({ ...prev, [name]: true }));
            const error = await validator(value, values);
            return error;
        } catch (err) {
            return 'Validation failed';
        } finally {
            setValidating(prev => ({ ...prev, [name]: false }));
        }
    }, [asyncValidators, values]);

    const handleChange = useCallback((event) => {
        const { name, value, type, checked } = event.target;
        const inputValue = type === 'checkbox' ? checked : value;

        setValues(prev => ({
            ...prev,
            [name]: inputValue
        }));

        // Clear error when user starts typing
        if (errors[name]) {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    }, [errors]);

    const handleBlur = useCallback(async (event) => {
        const { name } = event.target;
        const value = values[name];

        setTouched(prev => ({
            ...prev,
            [name]: true
        }));

        // Perform async validation
        const error = await validateFieldAsync(name, value);
        if (error) {
            setErrors(prev => ({
                ...prev,
                [name]: error
            }));
        } else {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    }, [values, validateFieldAsync]);

    return {
        values,
        errors,
        validating,
        touched,
        handleChange,
        handleBlur,
        setValue: (name, value) => setValues(prev => ({ ...prev, [name]: value }))
    };
}

// Usage with async validation
function UserForm() {
    const asyncValidators = {
        username: async (value) => {
            if (!value) return 'Username is required';

            // Simulate API call to check username availability
            await new Promise(resolve => setTimeout(resolve, 500));

            if (value === 'admin') {
                return 'Username is taken';
            }

            return null;
        },
        email: async (value) => {
            if (!value) return 'Email is required';

            // Simulate API call to check email
            await new Promise(resolve => setTimeout(resolve, 300));

            if (value === 'taken@example.com') {
                return 'Email is already registered';
            }

            return null;
        }
    };

    const {
        values,
        errors,
        validating,
        touched,
        handleChange,
        handleBlur
    } = useAsyncValidation({
        username: '',
        email: ''
    }, asyncValidators);

    return (
        <form>
            <div>
                <input
                    type="text"
                    name="username"
                    value={values.username}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Username"
                />
                {validating.username && <span>Checking...</span>}
                {touched.username && errors.username && <span className="error">{errors.username}</span>}
            </div>

            <div>
                <input
                    type="email"
                    name="email"
                    value={values.email}
                    onChange={handleChange}
                    onBlur={handleBlur}
                    placeholder="Email"
                />
                {validating.email && <span>Checking...</span>}
                {touched.email && errors.email && <span className="error">{errors.email}</span>}
            </div>
        </form>
    );
}
```

## Real React Examples:

### 1. **Complete Registration Form**

```javascript
function useRegistrationForm() {
    const [step, setStep] = useState(1);
    const totalSteps = 3;

    const form1 = useFormWithValidation({
        firstName: '',
        lastName: '',
        dateOfBirth: ''
    }, {
        firstName: { required: true, minLength: 2 },
        lastName: { required: true, minLength: 2 },
        dateOfBirth: { required: true }
    });

    const form2 = useFormWithValidation({
        email: '',
        phone: '',
        address: ''
    }, {
        email: {
            required: true,
            pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'Invalid email format'
        },
        phone: {
            required: true,
            pattern: /^\+?[\d\s\-\(\)]+$/,
            message: 'Invalid phone format'
        },
        address: { required: true, minLength: 10 }
    });

    const form3 = useFormWithValidation({
        password: '',
        confirmPassword: '',
        termsAccepted: false
    }, {
        password: { required: true, minLength: 8 },
        confirmPassword: {
            required: true,
            custom: (value, allValues) => {
                if (value !== allValues.password) {
                    return 'Passwords do not match';
                }
                return '';
            }
        },
        termsAccepted: {
            custom: (value) => {
                if (!value) return 'You must accept the terms';
                return '';
            }
        }
    });

    const nextStep = () => {
        if (step === 1 && form1.isValid) setStep(2);
        if (step === 2 && form2.isValid) setStep(3);
    };

    const prevStep = () => {
        if (step > 1) setStep(step - 1);
    };

    const submitAll = async () => {
        if (!form3.isValid) return;

        const allData = {
            ...form1.values,
            ...form2.values,
            ...form3.values
        };

        try {
            // Submit to API
            await submitRegistration(allData);
            alert('Registration successful!');
        } catch (error) {
            alert('Registration failed: ' + error.message);
        }
    };

    return {
        step,
        totalSteps,
        form1,
        form2,
        form3,
        nextStep,
        prevStep,
        submitAll,
        isComplete: step === totalSteps
    };
}

function MultiStepRegistration() {
    const {
        step,
        totalSteps,
        form1,
        form2,
        form3,
        nextStep,
        prevStep,
        submitAll,
        isComplete
    } = useRegistrationForm();

    const renderStep = () => {
        switch (step) {
            case 1:
                return (
                    <div>
                        <h3>Personal Information</h3>
                        <input
                            name="firstName"
                            value={form1.values.firstName}
                            onChange={form1.handleChange}
                            onBlur={form1.handleBlur}
                            placeholder="First Name"
                        />
                        {form1.errors.firstName && <span>{form1.errors.firstName}</span>}

                        <input
                            name="lastName"
                            value={form1.values.lastName}
                            onChange={form1.handleChange}
                            onBlur={form1.handleBlur}
                            placeholder="Last Name"
                        />
                        {form1.errors.lastName && <span>{form1.errors.lastName}</span>}

                        <input
                            type="date"
                            name="dateOfBirth"
                            value={form1.values.dateOfBirth}
                            onChange={form1.handleChange}
                            onBlur={form1.handleBlur}
                        />
                        {form1.errors.dateOfBirth && <span>{form1.errors.dateOfBirth}</span>}
                    </div>
                );

            case 2:
                return (
                    <div>
                        <h3>Contact Information</h3>
                        <input
                            type="email"
                            name="email"
                            value={form2.values.email}
                            onChange={form2.handleChange}
                            onBlur={form2.handleBlur}
                            placeholder="Email"
                        />
                        {form2.errors.email && <span>{form2.errors.email}</span>}

                        <input
                            type="tel"
                            name="phone"
                            value={form2.values.phone}
                            onChange={form2.handleChange}
                            onBlur={form2.handleBlur}
                            placeholder="Phone"
                        />
                        {form2.errors.phone && <span>{form2.errors.phone}</span>}

                        <textarea
                            name="address"
                            value={form2.values.address}
                            onChange={form2.handleChange}
                            onBlur={form2.handleBlur}
                            placeholder="Address"
                        />
                        {form2.errors.address && <span>{form2.errors.address}</span>}
                    </div>
                );

            case 3:
                return (
                    <div>
                        <h3>Security</h3>
                        <input
                            type="password"
                            name="password"
                            value={form3.values.password}
                            onChange={form3.handleChange}
                            onBlur={form3.handleBlur}
                            placeholder="Password"
                        />
                        {form3.errors.password && <span>{form3.errors.password}</span>}

                        <input
                            type="password"
                            name="confirmPassword"
                            value={form3.values.confirmPassword}
                            onChange={form3.handleChange}
                            onBlur={form3.handleBlur}
                            placeholder="Confirm Password"
                        />
                        {form3.errors.confirmPassword && <span>{form3.errors.confirmPassword}</span>}

                        <label>
                            <input
                                type="checkbox"
                                name="termsAccepted"
                                checked={form3.values.termsAccepted}
                                onChange={form3.handleChange}
                            />
                            I accept the terms and conditions
                        </label>
                        {form3.errors.termsAccepted && <span>{form3.errors.termsAccepted}</span>}
                    </div>
                );

            default:
                return null;
        }
    };

    return (
        <div className="multi-step-form">
            <div className="progress">
                Step {step} of {totalSteps}
            </div>

            {renderStep()}

            <div className="form-actions">
                {step > 1 && (
                    <button onClick={prevStep}>Previous</button>
                )}

                {step < totalSteps ? (
                    <button
                        onClick={nextStep}
                        disabled={
                            (step === 1 && !form1.isValid) ||
                            (step === 2 && !form2.isValid)
                        }
                    >
                        Next
                    </button>
                ) : (
                    <button
                        onClick={submitAll}
                        disabled={!form3.isValid || form3.isSubmitting}
                    >
                        {form3.isSubmitting ? 'Submitting...' : 'Complete Registration'}
                    </button>
                )}
            </div>
        </div>
    );
}
```

### 2. **Advanced Search Form with Filters**

```javascript
function useSearchForm(initialFilters = {}) {
    const [searchQuery, setSearchQuery] = useState('');
    const [debouncedQuery, setDebouncedQuery] = useState('');
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);

    const filters = useFormWithValidation(initialFilters, {
        category: {},
        priceMin: {
            custom: (value, allValues) => {
                if (value && allValues.priceMax && parseFloat(value) > parseFloat(allValues.priceMax)) {
                    return 'Min price cannot be greater than max price';
                }
                return '';
            }
        },
        priceMax: {
            custom: (value, allValues) => {
                if (value && allValues.priceMin && parseFloat(value) < parseFloat(allValues.priceMin)) {
                    return 'Max price cannot be less than min price';
                }
                return '';
            }
        },
        rating: {},
        inStock: {}
    });

    // Debounce search query
    useEffect(() => {
        const timer = setTimeout(() => {
            setDebouncedQuery(searchQuery);
        }, 300);

        return () => clearTimeout(timer);
    }, [searchQuery]);

    // Perform search when query or filters change
    useEffect(() => {
        if (!debouncedQuery.trim()) {
            setResults([]);
            return;
        }

        const search = async () => {
            setLoading(true);
            try {
                const searchParams = new URLSearchParams({
                    q: debouncedQuery,
                    ...filters.values
                });

                const response = await fetch(`/api/search?${searchParams}`);
                const data = await response.json();
                setResults(data.results);
            } catch (error) {
                console.error('Search failed:', error);
                setResults([]);
            } finally {
                setLoading(false);
            }
        };

        search();
    }, [debouncedQuery, filters.values]);

    return {
        searchQuery,
        setSearchQuery,
        filters,
        results,
        loading
    };
}

function AdvancedSearch() {
    const { searchQuery, setSearchQuery, filters, results, loading } = useSearchForm({
        category: '',
        priceMin: '',
        priceMax: '',
        rating: '',
        inStock: false
    });

    return (
        <div className="advanced-search">
            <div className="search-input">
                <input
                    type="text"
                    value={searchQuery}
                    onChange={(e) => setSearchQuery(e.target.value)}
                    placeholder="Search products..."
                />
            </div>

            <div className="filters">
                <select
                    name="category"
                    value={filters.values.category}
                    onChange={filters.handleChange}
                >
                    <option value="">All Categories</option>
                    <option value="electronics">Electronics</option>
                    <option value="books">Books</option>
                    <option value="clothing">Clothing</option>
                </select>

                <input
                    type="number"
                    name="priceMin"
                    value={filters.values.priceMin}
                    onChange={filters.handleChange}
                    onBlur={filters.handleBlur}
                    placeholder="Min Price"
                />
                {filters.errors.priceMin && <span>{filters.errors.priceMin}</span>}

                <input
                    type="number"
                    name="priceMax"
                    value={filters.values.priceMax}
                    onChange={filters.handleChange}
                    onBlur={filters.handleBlur}
                    placeholder="Max Price"
                />
                {filters.errors.priceMax && <span>{filters.errors.priceMax}</span>}

                <select
                    name="rating"
                    value={filters.values.rating}
                    onChange={filters.handleChange}
                >
                    <option value="">Any Rating</option>
                    <option value="4">4+ Stars</option>
                    <option value="3">3+ Stars</option>
                    <option value="2">2+ Stars</option>
                </select>

                <label>
                    <input
                        type="checkbox"
                        name="inStock"
                        checked={filters.values.inStock}
                        onChange={filters.handleChange}
                    />
                    In Stock Only
                </label>
            </div>

            {loading && <div>Searching...</div>}

            <div className="results">
                {results.map(product => (
                    <div key={product.id} className="product">
                        <h4>{product.name}</h4>
                        <p>${product.price}</p>
                        <p>Rating: {product.rating}/5</p>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

### 3. **Dynamic Form Builder**

```javascript
function useDynamicForm(schema = []) {
    const [formData, setFormData] = useState({});
    const [errors, setErrors] = useState({});
    const [touched, setTouched] = useState({});

    // Initialize form data from schema
    useEffect(() => {
        const initialData = {};
        schema.forEach(field => {
            initialData[field.name] = field.defaultValue || '';
        });
        setFormData(initialData);
    }, [schema]);

    const validateField = (fieldName, value) => {
        const field = schema.find(f => f.name === fieldName);
        if (!field || !field.validation) return '';

        const { validation } = field;

        if (validation.required && !value) {
            return `${field.label} is required`;
        }

        if (validation.minLength && value.length < validation.minLength) {
            return `${field.label} must be at least ${validation.minLength} characters`;
        }

        if (validation.pattern && !validation.pattern.test(value)) {
            return validation.message || `${field.label} format is invalid`;
        }

        return '';
    };

    const handleChange = (event) => {
        const { name, value, type, checked } = event.target;
        const inputValue = type === 'checkbox' ? checked : value;

        setFormData(prev => ({
            ...prev,
            [name]: inputValue
        }));

        // Clear error
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

        const error = validateField(name, formData[name]);
        if (error) {
            setErrors(prev => ({
                ...prev,
                [name]: error
            }));
        }
    };

    const validateForm = () => {
        const newErrors = {};
        let isValid = true;

        schema.forEach(field => {
            const error = validateField(field.name, formData[field.name]);
            if (error) {
                newErrors[field.name] = error;
                isValid = false;
            }
        });

        setErrors(newErrors);
        return isValid;
    };

    const handleSubmit = (onSubmit) => {
        return (event) => {
            event.preventDefault();

            if (!validateForm()) {
                // Mark all fields as touched
                const allTouched = {};
                schema.forEach(field => {
                    allTouched[field.name] = true;
                });
                setTouched(allTouched);
                return;
            }

            onSubmit(formData);
        };
    };

    return {
        formData,
        errors,
        touched,
        handleChange,
        handleBlur,
        handleSubmit,
        validateForm
    };
}

function DynamicFormBuilder({ schema }) {
    const { formData, errors, touched, handleChange, handleBlur, handleSubmit } = useDynamicForm(schema);

    const renderField = (field) => {
        const { name, label, type = 'text', options = [] } = field;
        const hasError = touched[name] && errors[name];

        switch (type) {
            case 'select':
                return (
                    <select
                        name={name}
                        value={formData[name] || ''}
                        onChange={handleChange}
                        onBlur={handleBlur}
                        className={hasError ? 'error' : ''}
                    >
                        <option value="">Select {label}</option>
                        {options.map(option => (
                            <option key={option.value} value={option.value}>
                                {option.label}
                            </option>
                        ))}
                    </select>
                );

            case 'checkbox':
                return (
                    <label>
                        <input
                            type="checkbox"
                            name={name}
                            checked={formData[name] || false}
                            onChange={handleChange}
                        />
                        {label}
                    </label>
                );

            case 'textarea':
                return (
                    <textarea
                        name={name}
                        value={formData[name] || ''}
                        onChange={handleChange}
                        onBlur={handleBlur}
                        placeholder={label}
                        className={hasError ? 'error' : ''}
                    />
                );

            default:
                return (
                    <input
                        type={type}
                        name={name}
                        value={formData[name] || ''}
                        onChange={handleChange}
                        onBlur={handleBlur}
                        placeholder={label}
                        className={hasError ? 'error' : ''}
                    />
                );
        }
    };

    const onSubmit = (data) => {
        console.log('Form submitted:', data);
        alert('Form submitted successfully!');
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            {schema.map(field => (
                <div key={field.name} className="form-field">
                    <label>{field.label}</label>
                    {renderField(field)}
                    {touched[field.name] && errors[field.name] && (
                        <span className="error">{errors[field.name]}</span>
                    )}
                </div>
            ))}

            <button type="submit">Submit</button>
        </form>
    );
}

// Usage
const formSchema = [
    {
        name: 'firstName',
        label: 'First Name',
        type: 'text',
        validation: { required: true, minLength: 2 }
    },
    {
        name: 'email',
        label: 'Email',
        type: 'email',
        validation: {
            required: true,
            pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'Please enter a valid email'
        }
    },
    {
        name: 'country',
        label: 'Country',
        type: 'select',
        options: [
            { value: 'us', label: 'United States' },
            { value: 'ca', label: 'Canada' },
            { value: 'uk', label: 'United Kingdom' }
        ],
        validation: { required: true }
    },
    {
        name: 'newsletter',
        label: 'Subscribe to newsletter',
        type: 'checkbox'
    }
];

<DynamicFormBuilder schema={formSchema} />
```

## Best Practices for Form Hooks

### 1. **State Management**
- Keep form state minimal and focused
- Use controlled components for complex forms
- Consider uncontrolled components for simple forms

### 2. **Validation**
- Validate on blur for better UX
- Provide immediate feedback for critical fields
- Use async validation for server-side checks

### 3. **Performance**
- Use `useCallback` for event handlers
- Debounce expensive validations
- Memoize validation functions

### 4. **User Experience**
- Show loading states during submission
- Clear errors when user starts correcting
- Provide helpful error messages

### 5. **Accessibility**
- Associate labels with inputs
- Use proper ARIA attributes
- Support keyboard navigation

### Interview Tip:
*"Custom form hooks encapsulate form state, validation, and submission logic. They should handle controlled inputs, provide validation with proper error states, and manage form submission with loading states. Always consider user experience with proper error handling and accessibility."*