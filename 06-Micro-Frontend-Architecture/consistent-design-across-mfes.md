# How do you ensure consistent design across MFEs?

## Question
How do you ensure consistent design across MFEs?

# How do you ensure consistent design across MFEs?

## Question
How do you ensure consistent design across MFEs?

## Answer

Ensuring consistent design across micro frontends is crucial for maintaining a cohesive user experience. Here are the main approaches:

## 1. Shared Design System/Components

### Create a Shared UI Library
```javascript
// design-system/src/components/Button.js
import React from 'react';
import styled from 'styled-components';

const StyledButton = styled.button`
  background: ${props => props.variant === 'primary' ? '#007bff' : '#6c757d'};
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  
  &:hover {
    opacity: 0.9;
  }
`;

export const Button = ({ children, variant = 'primary', ...props }) => {
  return <StyledButton variant={variant} {...props}>{children}</StyledButton>;
};
```

```javascript
// design-system/src/index.js
export { Button } from './components/Button';
export { Input } from './components/Input';
export { Card } from './components/Card';
export { ThemeProvider, useTheme } from './theme';
```

### Module Federation Configuration
```javascript
// design-system/webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'design_system',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/components/Button',
        './Input': './src/components/Input',
        './Card': './src/components/Card',
        './ThemeProvider': './src/theme/ThemeProvider'
      },
      shared: ['react', 'react-dom', 'styled-components']
    })
  ]
};
```

### Using Shared Components in MFEs
```javascript
// products-mfe/src/components/ProductCard.js
import React from 'react';
import { Card, Button } from 'design_system';

const ProductCard = ({ product }) => {
  return (
    <Card>
      <h3>{product.name}</h3>
      <p>{product.description}</p>
      <Button variant="primary">Add to Cart</Button>
    </Card>
  );
};
```

## 2. Shared Theme and CSS Variables

### Global CSS Variables
```css
/* design-system/src/theme/variables.css */
:root {
  /* Colors */
  --primary-color: #007bff;
  --secondary-color: #6c757d;
  --success-color: #28a745;
  --danger-color: #dc3545;
  
  /* Typography */
  --font-family: 'Inter', sans-serif;
  --font-size-base: 16px;
  --font-size-lg: 20px;
  
  /* Spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  
  /* Border radius */
  --border-radius: 4px;
  --border-radius-lg: 8px;
}
```

### Theme Provider
```javascript
// design-system/src/theme/ThemeProvider.js
import React from 'react';
import './variables.css';

const theme = {
  colors: {
    primary: 'var(--primary-color)',
    secondary: 'var(--secondary-color)',
    success: 'var(--success-color)',
    danger: 'var(--danger-color)'
  },
  typography: {
    fontFamily: 'var(--font-family)',
    fontSize: {
      base: 'var(--font-size-base)',
      lg: 'var(--font-size-lg)'
    }
  },
  spacing: {
    xs: 'var(--spacing-xs)',
    sm: 'var(--spacing-sm)',
    md: 'var(--spacing-md)',
    lg: 'var(--spacing-lg)'
  }
};

export const ThemeProvider = ({ children }) => {
  return (
    <div style={{ fontFamily: theme.typography.fontFamily }}>
      {children}
    </div>
  );
};

export const useTheme = () => theme;
```

## 3. CSS-in-JS with Shared Theme

### Styled Components Theme
```javascript
// design-system/src/theme/styledTheme.js
export const theme = {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff',
    text: '#333333'
  },
  fonts: {
    body: 'Inter, sans-serif',
    heading: 'Inter, sans-serif'
  },
  fontSizes: [12, 14, 16, 20, 24, 32],
  space: [0, 4, 8, 16, 32, 64]
};
```

```javascript
// design-system/src/components/StyledButton.js
import styled from 'styled-components';

export const StyledButton = styled.button`
  background-color: ${props => props.theme.colors.primary};
  color: white;
  border: none;
  padding: ${props => props.theme.space[2]}px ${props => props.theme.space[3]}px;
  border-radius: 4px;
  font-family: ${props => props.theme.fonts.body};
  cursor: pointer;
  
  &:hover {
    opacity: 0.9;
  }
`;
```

## 4. Design Tokens and CSS Custom Properties

### Design Tokens
```javascript
// design-system/src/tokens/index.js
export const tokens = {
  color: {
    primary: {
      50: '#eff6ff',
      500: '#007bff',
      900: '#1e3a8a'
    },
    gray: {
      50: '#f9fafb',
      500: '#6b7280',
      900: '#111827'
    }
  },
  spacing: {
    1: '4px',
    2: '8px',
    3: '12px',
    4: '16px',
    5: '20px'
  },
  typography: {
    fontSize: {
      xs: '12px',
      sm: '14px',
      base: '16px',
      lg: '18px',
      xl: '20px'
    },
    fontWeight: {
      normal: 400,
      medium: 500,
      bold: 700
    }
  }
};
```

### CSS Custom Properties Generation
```javascript
// design-system/scripts/generate-css.js
const tokens = require('./tokens');

const generateCSS = () => {
  let css = ':root {\n';
  
  // Colors
  Object.entries(tokens.color).forEach(([colorName, shades]) => {
    Object.entries(shades).forEach(([shade, value]) => {
      css += `  --color-${colorName}-${shade}: ${value};\n`;
    });
  });
  
  // Spacing
  Object.entries(tokens.spacing).forEach(([key, value]) => {
    css += `  --spacing-${key}: ${value};\n`;
  });
  
  css += '}\n';
  return css;
};

console.log(generateCSS());
```

## 5. Component Library with Storybook

### Storybook Configuration
```javascript
// design-system/.storybook/main.js
module.exports = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: ['@storybook/addon-essentials'],
  framework: '@storybook/react'
};
```

### Component Stories
```javascript
// design-system/src/components/Button.stories.js
import React from 'react';
import { Button } from './Button';

export default {
  title: 'Design System/Button',
  component: Button,
  argTypes: {
    variant: {
      control: { type: 'select' },
      options: ['primary', 'secondary', 'danger']
    }
  }
};

const Template = (args) => <Button {...args} />;

export const Primary = Template.bind({});
Primary.args = {
  children: 'Primary Button',
  variant: 'primary'
};

export const Secondary = Template.bind({});
Secondary.args = {
  children: 'Secondary Button',
  variant: 'secondary'
};
```

## 6. Centralized Design Documentation

### Design Guidelines
```markdown
# Design System Guidelines

## Colors
- Primary: Use for main actions (#007bff)
- Secondary: Use for secondary actions (#6c757d)
- Success: Use for positive feedback (#28a745)

## Typography
- Body: Inter, 16px
- Headings: Inter, 20px+, bold
- Spacing: Use 4px grid system

## Components
- Buttons: Minimum 44px touch target
- Inputs: Consistent padding and border radius
- Cards: Standard shadow and border radius
```

## 7. Automated Testing for Consistency

### Visual Regression Testing
```javascript
// design-system/src/components/__tests__/Button.visual.js
import { render } from '@testing-library/react';
import { Button } from '../Button';
import { ThemeProvider } from '../../theme';

describe('Button Visual Tests', () => {
  it('matches primary button snapshot', () => {
    const { container } = render(
      <ThemeProvider>
        <Button variant="primary">Test Button</Button>
      </ThemeProvider>
    );
    expect(container.firstChild).toMatchSnapshot();
  });
});
```

## Best Practices

1. **Single Source of Truth**: Maintain design tokens in one place
2. **Version Control**: Use semantic versioning for design system releases
3. **Documentation**: Keep design guidelines updated and accessible
4. **Testing**: Implement visual regression tests for components
5. **Communication**: Regular sync between design and development teams
6. **Accessibility**: Ensure all components meet WCAG guidelines
7. **Performance**: Optimize shared components for bundle size

## Interview Tips
- **Design system**: Centralized component library shared across MFEs
- **CSS variables**: For theming and consistent values
- **Storybook**: For component documentation and testing
- **Design tokens**: Abstract design decisions into reusable values
- **Theme provider**: Context-based theming for React applications
- **Visual testing**: Ensures UI consistency across deployments
- **Versioning**: Critical for coordinated MFE updates