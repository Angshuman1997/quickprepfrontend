# How Micro Frontends Communicate

## Simple Methods

### 1. **Custom Events (Most Common)**
```javascript
// Create a simple event system
const events = {};

const emit = (event, data) => {
  if (events[event]) {
    events[event].forEach(callback => callback(data));
  }
};

const on = (event, callback) => {
  if (!events[event]) events[event] = [];
  events[event].push(callback);
};

// MFE A sends message
const ProductList = () => {
  const selectProduct = (product) => {
    emit('PRODUCT_SELECTED', product);
  };

  return <div onClick={() => selectProduct(product)}>Select Product</div>;
};

// MFE B receives message
const ProductDetail = () => {
  const [product, setProduct] = useState(null);

  useEffect(() => {
    on('PRODUCT_SELECTED', setProduct);
  }, []);

  return <div>{product?.name}</div>;
};
```

### 2. **Shared State (Redux/Zustand)**
```javascript
// Shared store across MFEs
const store = createStore({
  selectedProduct: null
});

// MFE A updates state
const ProductList = () => {
  const dispatch = useDispatch();

  const selectProduct = (product) => {
    dispatch({ type: 'SELECT_PRODUCT', payload: product });
  };

  return <button onClick={() => selectProduct(product)}>Select</button>;
};

// MFE B reads state
const ProductDetail = () => {
  const product = useSelector(state => state.selectedProduct);

  return <div>{product?.name}</div>;
};
```

### 3. **URL Parameters**
```javascript
// MFE A navigates with data
const navigate = useNavigate();

const selectProduct = (product) => {
  navigate(`/product/${product.id}`);
};

// MFE B reads from URL
const ProductDetail = () => {
  const { id } = useParams();
  // Fetch product by ID
};
```

### 4. **Browser Storage**
```javascript
// MFE A saves to localStorage
const saveProduct = (product) => {
  localStorage.setItem('selectedProduct', JSON.stringify(product));
  window.dispatchEvent(new Event('storage')); // Trigger update
};

// MFE B listens for changes
useEffect(() => {
  const handleStorage = () => {
    const product = JSON.parse(localStorage.getItem('selectedProduct'));
    setProduct(product);
  };

  window.addEventListener('storage', handleStorage);
  return () => window.removeEventListener('storage', handleStorage);
}, []);
```

## Interview Questions & Answers

**Q: How do micro frontends communicate?**
**A:** "Micro frontends communicate through custom events, shared state management, URL parameters, or browser storage. The most common approach is custom events using a simple pub/sub pattern."

**Q: What's the best way for MFEs to communicate?**
**A:** "It depends on the use case. For loose coupling, use custom events. For complex state sharing, use a shared state management solution like Redux."

**Q: How do you handle cross-origin communication?**
**A:** "For cross-origin scenarios, use window.postMessage API with proper origin validation for security."

**Q: What are the challenges with MFE communication?**
**A:** "Challenges include maintaining loose coupling, handling different deployment timings, and ensuring communication works across different domains."

## Summary
- **Custom Events**: Simple pub/sub for loose coupling
- **Shared State**: Redux/Zustand for complex state
- **URL Params**: For navigation-based communication
- **LocalStorage**: Simple but synchronous communication
- **PostMessage**: For cross-origin scenarios