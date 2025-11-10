# How do you structure large component files to keep them maintainable?

## Question
How do you structure large component files to keep them maintainable?

## Answer

Large React components can become difficult to maintain, test, and understand. Breaking them down into smaller, focused pieces improves code organization, reusability, and maintainability. Here are proven patterns for structuring large component files.

## Component Splitting Strategies

### 1. **Extract Custom Hooks**

Move complex logic into custom hooks:

```javascript
// Before: Large component with mixed concerns
function UserDashboard({ userId }) {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [loading, setLoading] = useState(true);
    const [activeTab, setActiveTab] = useState('profile');

    useEffect(() => {
        const fetchData = async () => {
            setLoading(true);
            try {
                const [userRes, postsRes] = await Promise.all([
                    fetch(`/api/users/${userId}`),
                    fetch(`/api/users/${userId}/posts`)
                ]);
                setUser(await userRes.json());
                setPosts(await postsRes.json());
            } catch (error) {
                console.error('Failed to fetch data:', error);
            } finally {
                setLoading(false);
            }
        };
        fetchData();
    }, [userId]);

    const handleTabChange = (tab) => setActiveTab(tab);

    const handlePostLike = async (postId) => {
        // Complex like logic...
    };

    // Large render method...
    return (
        <div>
            {/* Large JSX structure */}
        </div>
    );
}

// After: Split into component + hooks
function useUserData(userId) {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        const fetchData = async () => {
            setLoading(true);
            try {
                const [userRes, postsRes] = await Promise.all([
                    fetch(`/api/users/${userId}`),
                    fetch(`/api/users/${userId}/posts`)
                ]);
                setUser(await userRes.json());
                setPosts(await postsRes.json());
            } catch (error) {
                console.error('Failed to fetch data:', error);
            } finally {
                setLoading(false);
            }
        };
        fetchData();
    }, [userId]);

    return { user, posts, loading };
}

function usePostInteractions() {
    const handlePostLike = useCallback(async (postId) => {
        try {
            await fetch(`/api/posts/${postId}/like`, { method: 'POST' });
            // Update local state or refetch data
        } catch (error) {
            console.error('Failed to like post:', error);
        }
    }, []);

    const handlePostShare = useCallback(async (postId) => {
        // Share logic...
    }, []);

    return { handlePostLike, handlePostShare };
}

function UserDashboard({ userId }) {
    const { user, posts, loading } = useUserData(userId);
    const { handlePostLike, handlePostShare } = usePostInteractions();
    const [activeTab, setActiveTab] = useState('profile');

    if (loading) return <LoadingSpinner />;

    return (
        <div className="dashboard">
            <UserProfile user={user} />
            <TabNavigation activeTab={activeTab} onTabChange={setActiveTab} />
            <TabContent
                activeTab={activeTab}
                posts={posts}
                onPostLike={handlePostLike}
                onPostShare={handlePostShare}
            />
        </div>
    );
}
```

### 2. **Extract Sub-components**

Break down the render method into smaller components:

```javascript
// Before: One large component
function ProductPage({ product }) {
    return (
        <div className="product-page">
            <div className="product-header">
                <img src={product.image} alt={product.name} />
                <div className="product-info">
                    <h1>{product.name}</h1>
                    <p className="price">${product.price}</p>
                    <div className="rating">
                        {Array.from({ length: 5 }).map((_, i) => (
                            <Star
                                key={i}
                                filled={i < product.rating}
                            />
                        ))}
                    </div>
                    <button className="add-to-cart">Add to Cart</button>
                </div>
            </div>

            <div className="product-details">
                <h2>Description</h2>
                <p>{product.description}</p>

                <h2>Specifications</h2>
                <ul>
                    {product.specifications.map(spec => (
                        <li key={spec.name}>
                            <strong>{spec.name}:</strong> {spec.value}
                        </li>
                    ))}
                </ul>

                <h2>Reviews</h2>
                <div className="reviews">
                    {product.reviews.map(review => (
                        <div key={review.id} className="review">
                            <div className="review-header">
                                <span className="reviewer">{review.author}</span>
                                <div className="review-rating">
                                    {Array.from({ length: 5 }).map((_, i) => (
                                        <Star key={i} filled={i < review.rating} />
                                    ))}
                                </div>
                            </div>
                            <p className="review-text">{review.text}</p>
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
}

// After: Split into focused components
function ProductHeader({ product }) {
    return (
        <div className="product-header">
            <img src={product.image} alt={product.name} />
            <div className="product-info">
                <h1>{product.name}</h1>
                <p className="price">${product.price}</p>
                <StarRating rating={product.rating} />
                <AddToCartButton product={product} />
            </div>
        </div>
    );
}

function StarRating({ rating }) {
    return (
        <div className="rating">
            {Array.from({ length: 5 }).map((_, i) => (
                <Star key={i} filled={i < rating} />
            ))}
        </div>
    );
}

function AddToCartButton({ product }) {
    const handleAddToCart = () => {
        // Add to cart logic
    };

    return (
        <button className="add-to-cart" onClick={handleAddToCart}>
            Add to Cart
        </button>
    );
}

function ProductSpecifications({ specifications }) {
    return (
        <div>
            <h2>Specifications</h2>
            <ul>
                {specifications.map(spec => (
                    <li key={spec.name}>
                        <strong>{spec.name}:</strong> {spec.value}
                    </li>
                ))}
            </ul>
        </div>
    );
}

function ProductReviews({ reviews }) {
    return (
        <div className="reviews">
            <h2>Reviews</h2>
            {reviews.map(review => (
                <ReviewItem key={review.id} review={review} />
            ))}
        </div>
    );
}

function ReviewItem({ review }) {
    return (
        <div className="review">
            <div className="review-header">
                <span className="reviewer">{review.author}</span>
                <StarRating rating={review.rating} />
            </div>
            <p className="review-text">{review.text}</p>
        </div>
    );
}

function ProductPage({ product }) {
    return (
        <div className="product-page">
            <ProductHeader product={product} />
            <div className="product-details">
                <h2>Description</h2>
                <p>{product.description}</p>
                <ProductSpecifications specifications={product.specifications} />
                <ProductReviews reviews={product.reviews} />
            </div>
        </div>
    );
}
```

### 3. **Use Render Props or Compound Components**

For complex UI patterns:

```javascript
// Render Props Pattern
function DataTable({ data, children }) {
    const [sortConfig, setSortConfig] = useState({ key: null, direction: 'asc' });
    const [selectedRows, setSelectedRows] = useState(new Set());

    const sortedData = useMemo(() => {
        if (!sortConfig.key) return data;

        return [...data].sort((a, b) => {
            const aValue = a[sortConfig.key];
            const bValue = b[sortConfig.key];

            if (aValue < bValue) return sortConfig.direction === 'asc' ? -1 : 1;
            if (aValue > bValue) return sortConfig.direction === 'asc' ? 1 : -1;
            return 0;
        });
    }, [data, sortConfig]);

    const handleSort = (key) => {
        setSortConfig(prev => ({
            key,
            direction: prev.key === key && prev.direction === 'asc' ? 'desc' : 'asc'
        }));
    };

    const handleRowSelect = (id) => {
        setSelectedRows(prev => {
            const newSet = new Set(prev);
            if (newSet.has(id)) {
                newSet.delete(id);
            } else {
                newSet.add(id);
            }
            return newSet;
        });
    };

    return children({
        sortedData,
        sortConfig,
        selectedRows,
        onSort: handleSort,
        onRowSelect: handleRowSelect
    });
}

// Usage
function UserTable({ users }) {
    return (
        <DataTable data={users}>
            {({ sortedData, sortConfig, selectedRows, onSort, onRowSelect }) => (
                <table>
                    <thead>
                        <tr>
                            <th>
                                <input type="checkbox" />
                            </th>
                            <SortableHeader
                                label="Name"
                                sortKey="name"
                                sortConfig={sortConfig}
                                onSort={onSort}
                            />
                            <SortableHeader
                                label="Email"
                                sortKey="email"
                                sortConfig={sortConfig}
                                onSort={onSort}
                            />
                        </tr>
                    </thead>
                    <tbody>
                        {sortedData.map(user => (
                            <tr key={user.id}>
                                <td>
                                    <input
                                        type="checkbox"
                                        checked={selectedRows.has(user.id)}
                                        onChange={() => onRowSelect(user.id)}
                                    />
                                </td>
                                <td>{user.name}</td>
                                <td>{user.email}</td>
                            </tr>
                        ))}
                    </tbody>
                </table>
            )}
        </DataTable>
    );
}
```

## File Organization Patterns

### 1. **Component Folder Structure**

```
components/
├── UserDashboard/
│   ├── UserDashboard.js          # Main component
│   ├── UserDashboard.css         # Styles
│   ├── UserDashboard.test.js     # Tests
│   ├── components/               # Sub-components
│   │   ├── UserProfile.js
│   │   ├── PostList.js
│   │   └── PostItem.js
│   ├── hooks/                    # Custom hooks
│   │   ├── useUserData.js
│   │   └── usePostInteractions.js
│   ├── utils/                    # Utilities
│   │   └── formatters.js
│   └── index.js                  # Export file
```

### 2. **Barrel Exports**

```javascript
// components/UserDashboard/index.js
export { default } from './UserDashboard';
export { default as UserProfile } from './components/UserProfile';
export { default as PostList } from './components/PostList';

// Usage
import UserDashboard, { UserProfile, PostList } from './components/UserDashboard';
```

### 3. **Separate Concerns**

```javascript
// UserDashboard.js - Main component logic
import { useUserData, usePostInteractions } from './hooks';
import UserProfile from './components/UserProfile';
import PostList from './components/PostList';
import './UserDashboard.css';

function UserDashboard({ userId }) {
    const { user, posts, loading } = useUserData(userId);
    const { handleLike, handleShare } = usePostInteractions();

    if (loading) return <div>Loading...</div>;

    return (
        <div className="dashboard">
            <UserProfile user={user} />
            <PostList posts={posts} onLike={handleLike} onShare={handleShare} />
        </div>
    );
}

export default UserDashboard;

// hooks/useUserData.js - Data logic
export function useUserData(userId) {
    // Data fetching logic
}

// hooks/usePostInteractions.js - Interaction logic
export function usePostInteractions() {
    // Interaction logic
}

// components/UserProfile.js - UI component
function UserProfile({ user }) {
    return (
        <div className="profile">
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}
```

## Advanced Patterns

### 1. **Container/Presentational Pattern**

```javascript
// Container component - handles data and logic
function UserDashboardContainer({ userId }) {
    const { user, posts, loading, error } = useUserData(userId);
    const { handleLike, handleShare } = usePostInteractions();

    if (loading) return <LoadingSpinner />;
    if (error) return <ErrorMessage error={error} />;

    return (
        <UserDashboardPresentation
            user={user}
            posts={posts}
            onLike={handleLike}
            onShare={handleShare}
        />
    );
}

// Presentational component - handles UI only
function UserDashboardPresentation({ user, posts, onLike, onShare }) {
    return (
        <div className="dashboard">
            <UserProfile user={user} />
            <PostList posts={posts} onLike={onLike} onShare={onShare} />
        </div>
    );
}
```

### 2. **Compound Components Pattern**

```javascript
// Compound component for complex forms
const Form = ({ children, onSubmit }) => {
    const [values, setValues] = useState({});
    const [errors, setErrors] = useState({});

    const handleSubmit = (e) => {
        e.preventDefault();
        onSubmit(values);
    };

    const registerField = (name) => ({
        value: values[name] || '',
        onChange: (value) => setValues(prev => ({ ...prev, [name]: value })),
        error: errors[name]
    });

    return (
        <form onSubmit={handleSubmit}>
            {typeof children === 'function' ? children({ registerField }) : children}
        </form>
    );
};

Form.Field = ({ name, children }) => {
    const { value, onChange, error } = useFormField(name);

    return (
        <div className="form-field">
            {children({ value, onChange })}
            {error && <span className="error">{error}</span>}
        </div>
    );
};

// Usage
function ContactForm() {
    const handleSubmit = (values) => {
        console.log('Form submitted:', values);
    };

    return (
        <Form onSubmit={handleSubmit}>
            {({ registerField }) => (
                <>
                    <Form.Field name="name">
                        {({ value, onChange }) => (
                            <input
                                type="text"
                                value={value}
                                onChange={(e) => onChange(e.target.value)}
                                placeholder="Name"
                            />
                        )}
                    </Form.Field>

                    <Form.Field name="email">
                        {({ value, onChange }) => (
                            <input
                                type="email"
                                value={value}
                                onChange={(e) => onChange(e.target.value)}
                                placeholder="Email"
                            />
                        )}
                    </Form.Field>

                    <button type="submit">Submit</button>
                </>
            )}
        </Form>
    );
}
```

### 3. **Render Props with Custom Hooks**

```javascript
// Custom hook for complex state management
function useWizard(steps) {
    const [currentStep, setCurrentStep] = useState(0);
    const [data, setData] = useState({});

    const next = () => setCurrentStep(prev => Math.min(prev + 1, steps.length - 1));
    const prev = () => setCurrentStep(prev => Math.max(prev - 1, 0));
    const goToStep = (step) => setCurrentStep(step);

    const updateData = (stepData) => {
        setData(prev => ({ ...prev, ...stepData }));
    };

    return {
        currentStep,
        data,
        next,
        prev,
        goToStep,
        updateData,
        isFirst: currentStep === 0,
        isLast: currentStep === steps.length - 1
    };
}

// Wizard component using render props
function Wizard({ steps, onComplete }) {
    const wizard = useWizard(steps);

    const handleStepSubmit = (stepData) => {
        wizard.updateData(stepData);
        if (wizard.isLast) {
            onComplete(wizard.data);
        } else {
            wizard.next();
        }
    };

    const CurrentStepComponent = steps[wizard.currentStep].component;

    return (
        <div className="wizard">
            <div className="wizard-progress">
                {steps.map((step, index) => (
                    <div
                        key={step.id}
                        className={`step ${index === wizard.currentStep ? 'active' : ''} ${index < wizard.currentStep ? 'completed' : ''}`}
                    >
                        {step.title}
                    </div>
                ))}
            </div>

            <div className="wizard-content">
                <CurrentStepComponent
                    data={wizard.data}
                    onSubmit={handleStepSubmit}
                />
            </div>

            <div className="wizard-navigation">
                {!wizard.isFirst && (
                    <button onClick={wizard.prev}>Previous</button>
                )}
                {!wizard.isLast && (
                    <button onClick={wizard.next}>Next</button>
                )}
            </div>
        </div>
    );
}

// Usage
const steps = [
    { id: 'personal', title: 'Personal Info', component: PersonalInfoStep },
    { id: 'contact', title: 'Contact Details', component: ContactDetailsStep },
    { id: 'review', title: 'Review', component: ReviewStep }
];

function OnboardingWizard() {
    const handleComplete = (data) => {
        console.log('Onboarding complete:', data);
    };

    return <Wizard steps={steps} onComplete={handleComplete} />;
}
```

## Real React Examples

### 1. **E-commerce Product Page**

```javascript
// ProductPage.js
import { useProductData } from './hooks/useProductData';
import { useCart } from './hooks/useCart';
import ProductGallery from './components/ProductGallery';
import ProductInfo from './components/ProductInfo';
import ProductReviews from './components/ProductReviews';
import RelatedProducts from './components/RelatedProducts';
import './ProductPage.css';

function ProductPage({ productId }) {
    const { product, loading, error } = useProductData(productId);
    const { addToCart, isInCart } = useCart();

    if (loading) return <ProductPageSkeleton />;
    if (error) return <ProductPageError error={error} />;
    if (!product) return <ProductNotFound />;

    return (
        <div className="product-page">
            <div className="product-main">
                <ProductGallery images={product.images} />
                <ProductInfo
                    product={product}
                    onAddToCart={addToCart}
                    isInCart={isInCart(product.id)}
                />
            </div>

            <div className="product-details">
                <ProductTabs
                    description={product.description}
                    specifications={product.specifications}
                    shipping={product.shipping}
                />
            </div>

            <ProductReviews
                reviews={product.reviews}
                productId={product.id}
            />

            <RelatedProducts
                products={product.related}
                currentProductId={product.id}
            />
        </div>
    );
}

// hooks/useProductData.js
export function useProductData(productId) {
    const [product, setProduct] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        const fetchProduct = async () => {
            try {
                const response = await fetch(`/api/products/${productId}`);
                const data = await response.json();
                setProduct(data);
            } catch (err) {
                setError(err.message);
            } finally {
                setLoading(false);
            }
        };

        fetchProduct();
    }, [productId]);

    return { product, loading, error };
}

// components/ProductInfo.js
function ProductInfo({ product, onAddToCart, isInCart }) {
    const [selectedVariant, setSelectedVariant] = useState(product.variants[0]);

    return (
        <div className="product-info">
            <h1>{product.name}</h1>
            <p className="price">${product.price}</p>

            <VariantSelector
                variants={product.variants}
                selected={selectedVariant}
                onSelect={setSelectedVariant}
            />

            <AddToCartButton
                product={product}
                variant={selectedVariant}
                onAddToCart={onAddToCart}
                isInCart={isInCart}
            />

            <ProductHighlights features={product.features} />
        </div>
    );
}
```

### 2. **Dashboard with Multiple Widgets**

```javascript
// Dashboard.js
import { useDashboardData } from './hooks/useDashboardData';
import { useWidgetVisibility } from './hooks/useWidgetVisibility';
import DashboardHeader from './components/DashboardHeader';
import StatsGrid from './components/StatsGrid';
import ChartsSection from './components/ChartsSection';
import RecentActivity from './components/RecentActivity';
import QuickActions from './components/QuickActions';
import './Dashboard.css';

function Dashboard() {
    const { data, loading, error, refresh } = useDashboardData();
    const { visibleWidgets, toggleWidget } = useWidgetVisibility();

    if (loading) return <DashboardSkeleton />;
    if (error) return <DashboardError error={error} onRetry={refresh} />;

    return (
        <div className="dashboard">
            <DashboardHeader
                user={data.user}
                lastUpdated={data.lastUpdated}
                onRefresh={refresh}
            />

            <div className="dashboard-grid">
                {visibleWidgets.stats && (
                    <StatsGrid stats={data.stats} />
                )}

                {visibleWidgets.charts && (
                    <ChartsSection charts={data.charts} />
                )}

                <div className="dashboard-sidebar">
                    {visibleWidgets.activity && (
                        <RecentActivity activities={data.activities} />
                    )}

                    {visibleWidgets.actions && (
                        <QuickActions actions={data.actions} />
                    )}
                </div>
            </div>

            <WidgetCustomizer
                visibleWidgets={visibleWidgets}
                onToggleWidget={toggleWidget}
            />
        </div>
    );
}

// hooks/useDashboardData.js
export function useDashboardData() {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    const fetchData = useCallback(async () => {
        setLoading(true);
        setError(null);

        try {
            const response = await fetch('/api/dashboard');
            const dashboardData = await response.json();
            setData(dashboardData);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, []);

    useEffect(() => {
        fetchData();
    }, [fetchData]);

    return { data, loading, error, refresh: fetchData };
}

// hooks/useWidgetVisibility.js
export function useWidgetVisibility() {
    const [visibleWidgets, setVisibleWidgets] = useState({
        stats: true,
        charts: true,
        activity: true,
        actions: true
    });

    const toggleWidget = useCallback((widgetName) => {
        setVisibleWidgets(prev => ({
            ...prev,
            [widgetName]: !prev[widgetName]
        }));
    }, []);

    return { visibleWidgets, toggleWidget };
}
```

### 3. **Complex Form with Validation**

```javascript
// ComplexForm.js
import { useFormState } from './hooks/useFormState';
import { useFormValidation } from './hooks/useFormValidation';
import PersonalInfoSection from './components/PersonalInfoSection';
import ContactInfoSection from './components/ContactInfoSection';
import PreferencesSection from './components/PreferencesSection';
import FormNavigation from './components/FormNavigation';
import FormProgress from './components/FormProgress';
import './ComplexForm.css';

const FORM_STEPS = ['personal', 'contact', 'preferences', 'review'];

function ComplexForm({ onSubmit }) {
    const { currentStep, goToStep, canGoNext, canGoPrev } = useFormState(FORM_STEPS);
    const {
        formData,
        updateField,
        errors,
        isValid,
        validateStep
    } = useFormValidation(FORM_STEPS);

    const handleNext = () => {
        if (validateStep(currentStep)) {
            goToStep(currentStep + 1);
        }
    };

    const handlePrev = () => {
        goToStep(currentStep - 1);
    };

    const handleSubmit = () => {
        if (isValid) {
            onSubmit(formData);
        }
    };

    const renderCurrentStep = () => {
        switch (FORM_STEPS[currentStep]) {
            case 'personal':
                return (
                    <PersonalInfoSection
                        data={formData.personal}
                        onChange={(field, value) => updateField('personal', field, value)}
                        errors={errors.personal}
                    />
                );
            case 'contact':
                return (
                    <ContactInfoSection
                        data={formData.contact}
                        onChange={(field, value) => updateField('contact', field, value)}
                        errors={errors.contact}
                    />
                );
            case 'preferences':
                return (
                    <PreferencesSection
                        data={formData.preferences}
                        onChange={(field, value) => updateField('preferences', field, value)}
                        errors={errors.preferences}
                    />
                );
            case 'review':
                return <ReviewSection data={formData} />;
            default:
                return null;
        }
    };

    return (
        <div className="complex-form">
            <FormProgress
                steps={FORM_STEPS}
                currentStep={currentStep}
            />

            <form className="form-content">
                {renderCurrentStep()}
            </form>

            <FormNavigation
                currentStep={currentStep}
                totalSteps={FORM_STEPS.length}
                canGoNext={canGoNext}
                canGoPrev={canGoPrev}
                onNext={handleNext}
                onPrev={handlePrev}
                onSubmit={handleSubmit}
            />
        </div>
    );
}

// hooks/useFormState.js
export function useFormState(steps) {
    const [currentStep, setCurrentStep] = useState(0);

    const goToStep = useCallback((step) => {
        setCurrentStep(Math.max(0, Math.min(step, steps.length - 1)));
    }, [steps.length]);

    const canGoNext = currentStep < steps.length - 1;
    const canGoPrev = currentStep > 0;

    return { currentStep, goToStep, canGoNext, canGoPrev };
}

// hooks/useFormValidation.js
export function useFormValidation(steps) {
    const [formData, setFormData] = useState(() => {
        const initial = {};
        steps.forEach(step => {
            initial[step] = {};
        });
        return initial;
    });

    const [errors, setErrors] = useState({});

    const updateField = useCallback((step, field, value) => {
        setFormData(prev => ({
            ...prev,
            [step]: {
                ...prev[step],
                [field]: value
            }
        }));

        // Clear error when user starts typing
        if (errors[step]?.[field]) {
            setErrors(prev => ({
                ...prev,
                [step]: {
                    ...prev[step],
                    [field]: ''
                }
            }));
        }
    }, [errors]);

    const validateStep = useCallback((stepIndex) => {
        const step = steps[stepIndex];
        const stepData = formData[step];
        const stepErrors = {};

        // Validation logic for each step
        switch (step) {
            case 'personal':
                if (!stepData.firstName) stepErrors.firstName = 'First name is required';
                if (!stepData.lastName) stepErrors.lastName = 'Last name is required';
                break;
            case 'contact':
                if (!stepData.email) stepErrors.email = 'Email is required';
                if (stepData.email && !/\S+@\S+\.\S+/.test(stepData.email)) {
                    stepErrors.email = 'Email is invalid';
                }
                break;
            // Add more validation...
        }

        setErrors(prev => ({
            ...prev,
            [step]: stepErrors
        }));

        return Object.keys(stepErrors).length === 0;
    }, [steps, formData]);

    const isValid = useMemo(() => {
        return steps.every((_, index) => validateStep(index));
    }, [steps, validateStep]);

    return { formData, updateField, errors, isValid, validateStep };
}
```

## Best Practices

### 1. **Single Responsibility Principle**
- Each component/hook should have one clear purpose
- Extract logic when a function exceeds 30-50 lines
- Split components when JSX becomes too complex

### 2. **File Organization**
- Group related files in folders
- Use index.js files for clean imports
- Separate concerns (logic, UI, styles, tests)

### 3. **Naming Conventions**
- Use descriptive names for hooks and components
- Prefix custom hooks with `use`
- Use PascalCase for components, camelCase for hooks

### 4. **Performance Considerations**
- Use React.memo for expensive components
- Memoize callbacks and values appropriately
- Avoid over-splitting (each split has a cost)

### 5. **Testing Strategy**
- Test components, hooks, and utilities separately
- Use integration tests for component interactions
- Mock external dependencies

### Interview Tip:
*"Break large components by extracting custom hooks for logic, creating sub-components for UI sections, and organizing files by feature. Use container/presentational pattern and compound components for complex UIs. Keep components focused and testable."*