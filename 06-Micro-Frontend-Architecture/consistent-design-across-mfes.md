# Consistent Design Across Micro Frontends

## Simple Solution: Shared Design System

### Create a Shared Component Library
```javascript
// design-system/src/Button.js
const Button = ({ children, variant = 'primary', onClick }) => (
  <button 
    className={`btn btn-${variant}`}
    onClick={onClick}
  >
    {children}
  </button>
);

export default Button;
```

### Share via Module Federation
```javascript
// design-system/vite.config.js
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'designSystem',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/Button',
        './Card': './src/Card',
        './Input': './src/Input'
      }
    })
  ]
})
```

### Use in Any MFE
```javascript
// products-mfe/src/ProductCard.js
import Button from 'designSystem/Button';
import Card from 'designSystem/Card';

const ProductCard = ({ product }) => (
  <Card>
    <h3>{product.name}</h3>
    <Button onClick={() => addToCart(product)}>
      Add to Cart
    </Button>
  </Card>
);
```

## CSS Variables for Consistency

### Global Design Tokens
```css
/* design-system/styles.css */
:root {
  --primary-color: #007bff;
  --secondary-color: #6c757d;
  --font-family: 'Inter', sans-serif;
  --border-radius: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
}
```

### Consistent Styling
```javascript
// All MFEs use the same CSS variables
.btn-primary {
  background: var(--primary-color);
  border-radius: var(--border-radius);
  font-family: var(--font-family);
}
```

## Interview Questions & Answers

**Q: How do you ensure consistent design across micro frontends?**
**A:** "I create a shared design system with components and CSS variables that all micro frontends can consume via Module Federation."

**Q: What's the main challenge with design consistency?**
**A:** "Different teams might create similar components with slight variations. A shared design system prevents this by providing approved components."

**Q: How do you share CSS styles across MFEs?**
**A:** "Using CSS custom properties (variables) defined in a shared stylesheet that all micro frontends import."

**Q: What tools help maintain design consistency?**
**A:** "Storybook for component documentation, visual regression testing, and a centralized design system library."

## Summary
- **Shared Components**: One Button, Card, etc. for all MFEs
- **CSS Variables**: Consistent colors, spacing, fonts
- **Design System**: Centralized rules and components
- **Benefit**: Unified user experience across all MFEs