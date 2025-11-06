# What is BEM naming convention?

## Question
What is BEM naming convention?

# What is BEM naming convention?

## Question
What is BEM naming convention?

## Answer

BEM (Block Element Modifier) is a popular CSS naming convention that helps create maintainable and scalable CSS code. It provides a clear structure for naming CSS classes.

## BEM Structure

### Block
A block is a standalone component that is meaningful on its own.

```html
<!-- Block: header -->
<header class="header">
  <div class="logo">...</div>
  <nav class="nav">...</nav>
</header>

<!-- Block: button -->
<button class="button">Click me</button>

<!-- Block: card -->
<div class="card">
  <h3 class="card__title">Card Title</h3>
  <p class="card__content">Card content</p>
</div>
```

```css
/* Block styles */
.header {
  background: #f5f5f5;
  padding: 1rem;
}

.button {
  background: #007acc;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
}

.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
```

### Element
An element is a part of a block that has no standalone meaning.

```html
<!-- Elements within header block -->
<header class="header">
  <div class="header__logo">
    <img src="logo.png" alt="Company Logo">
  </div>
  <nav class="header__nav">
    <ul class="header__nav-list">
      <li class="header__nav-item">
        <a href="/" class="header__nav-link">Home</a>
      </li>
      <li class="header__nav-item">
        <a href="/about" class="header__nav-link">About</a>
      </li>
    </ul>
  </nav>
</header>

<!-- Elements within card block -->
<div class="card">
  <img src="image.jpg" alt="Card image" class="card__image">
  <div class="card__content">
    <h3 class="card__title">Card Title</h3>
    <p class="card__description">Card description</p>
    <button class="card__button">Read More</button>
  </div>
</div>
```

```css
/* Element styles */
.header__logo {
  font-size: 1.5rem;
  font-weight: bold;
}

.header__nav {
  display: flex;
  justify-content: space-between;
}

.header__nav-list {
  display: flex;
  list-style: none;
  gap: 1rem;
}

.header__nav-link {
  text-decoration: none;
  color: #333;
}

.card__image {
  width: 100%;
  height: 200px;
  object-fit: cover;
  border-radius: 4px 4px 0 0;
}

.card__content {
  padding: 1rem;
}

.card__title {
  margin: 0 0 0.5rem 0;
  font-size: 1.25rem;
}

.card__description {
  color: #666;
  margin-bottom: 1rem;
}

.card__button {
  background: #007acc;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}
```

### Modifier
A modifier changes the appearance or behavior of a block or element.

```html
<!-- Block modifiers -->
<button class="button">Default Button</button>
<button class="button button--primary">Primary Button</button>
<button class="button button--secondary">Secondary Button</button>
<button class="button button--large">Large Button</button>
<button class="button button--disabled">Disabled Button</button>

<!-- Element modifiers -->
<div class="card card--featured">
  <img src="image.jpg" alt="Card image" class="card__image">
  <div class="card__content">
    <h3 class="card__title card__title--highlighted">Featured Card</h3>
    <p class="card__description">This is a featured card</p>
  </div>
</div>

<!-- Size modifiers -->
<div class="card card--small">Small card</div>
<div class="card card--medium">Medium card</div>
<div class="card card--large">Large card</div>
```

```css
/* Block modifiers */
.button--primary {
  background: #007acc;
  color: white;
}

.button--secondary {
  background: #28a745;
  color: white;
}

.button--large {
  padding: 0.75rem 1.5rem;
  font-size: 1.1rem;
}

.button--disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Element modifiers */
.card--featured {
  border: 2px solid #007acc;
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.card__title--highlighted {
  color: #007acc;
  font-weight: bold;
}

/* Size modifiers */
.card--small {
  width: 200px;
}

.card--medium {
  width: 300px;
}

.card--large {
  width: 400px;
}
```

## BEM Naming Rules

### Block Names
- Use lowercase letters
- Use hyphens for multi-word names
- Should be descriptive and meaningful

```css
/* Good */
.user-profile
.search-form
.article-card

/* Bad */
.UserProfile  /* Wrong case */
user_profile  /* Wrong separator */
u-p           /* Not descriptive */
```

### Element Names
- Use double underscores `__` to separate block from element
- Use lowercase letters
- Use hyphens for multi-word names

```css
/* Good */
.card__title
.card__content
.user-profile__avatar
.search-form__input

/* Bad */
.card__Title      /* Wrong case */
card__content     /* Wrong separator */
card__contentText /* Wrong separator */
```

### Modifier Names
- Use double hyphens `--` to separate block/element from modifier
- Use lowercase letters
- Use hyphens for multi-word names

```css
/* Good */
.button--primary
.button--large
.card--featured
.card__title--highlighted

/* Bad */
.button--Primary    /* Wrong case */
button--primary     /* Wrong separator */
button--primaryBtn  /* Wrong separator */
```

## Advanced BEM Patterns

### Nested Blocks
```html
<div class="page">
  <header class="header">
    <div class="logo">...</div>
    <nav class="nav">
      <ul class="nav__list">
        <li class="nav__item">
          <a href="/" class="nav__link">Home</a>
        </li>
      </ul>
    </nav>
  </header>
  
  <main class="main">
    <div class="card">
      <h3 class="card__title">Card Title</h3>
      <div class="card__content">
        <button class="button button--primary">Read More</button>
      </div>
    </div>
  </main>
</div>
```

### Mixes (Multiple Blocks on One Element)
```html
<!-- Mix: combining multiple blocks -->
<div class="card search-result">
  <h3 class="card__title search-result__title">Search Result</h3>
  <p class="card__content search-result__snippet">Result snippet...</p>
</div>

<!-- Mix with modifiers -->
<button class="button button--primary nav__button nav__button--active">
  Active Nav Button
</button>
```

### Complex Modifiers
```html
<!-- Theme modifiers -->
<div class="card card--theme-dark">
  <h3 class="card__title card__title--theme-dark">Dark Theme Card</h3>
</div>

<!-- State modifiers -->
<button class="button button--loading" disabled>
  <span class="button__text">Loading...</span>
  <span class="button__spinner" aria-hidden="true"></span>
</button>

<!-- Size modifiers -->
<div class="modal modal--size-large modal--centered">
  <div class="modal__content">
    <h2 class="modal__title">Large Centered Modal</h2>
  </div>
</div>
```

## BEM in CSS Preprocessors

### SCSS with BEM
```scss
// Variables
$color-primary: #007acc;
$color-secondary: #28a745;

// Block
.button {
  display: inline-block;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;

  // Element
  &__text {
    font-weight: 500;
  }

  &__spinner {
    display: none;
    margin-left: 0.5rem;
  }

  // Modifiers
  &--primary {
    background: $color-primary;
    color: white;

    &:hover {
      background: darken($color-primary, 10%);
    }
  }

  &--secondary {
    background: $color-secondary;
    color: white;

    &:hover {
      background: darken($color-secondary, 10%);
    }
  }

  &--loading {
    position: relative;
    color: transparent;

    .button__spinner {
      display: inline-block;
    }
  }

  &--disabled {
    opacity: 0.6;
    cursor: not-allowed;

    &:hover {
      background: inherit;
    }
  }
}
```

### CSS Modules with BEM
```css
/* Button.module.css */
.button {
  /* base styles */
}

.button__text {
  /* text styles */
}

.button--primary {
  /* primary variant */
}

.button--secondary {
  /* secondary variant */
}
```

```jsx
// React component
import styles from './Button.module.css';

const Button = ({ variant, children, loading }) => {
  const className = [
    styles.button,
    variant && styles[`button--${variant}`],
    loading && styles['button--loading']
  ].filter(Boolean).join(' ');

  return (
    <button className={className}>
      <span className={styles.button__text}>{children}</span>
      {loading && <span className={styles.button__spinner}>...</span>}
    </button>
  );
};
```

## BEM vs Other Methodologies

### BEM vs OOCSS
```css
/* OOCSS */
.skin-blue {
  background: blue;
}

.size-large {
  width: 200px;
  height: 100px;
}

/* Usage */
<div class="skin-blue size-large">...</div>

/* BEM */
.button--primary {
  background: blue;
}

.button--large {
  width: 200px;
  height: 100px;
}

/* Usage */
<button class="button button--primary button--large">...</button>
```

### BEM vs SMACSS
```css
/* SMACSS */
.button {
  /* base */
}

.button-primary {
  /* theme */
}

.is-active {
  /* state */
}

/* BEM */
.button {
  /* block */
}

.button--primary {
  /* modifier */
}

.button--active {
  /* modifier for state */
}
```

## Common BEM Mistakes

### 1. Element of Element (Wrong)
```css
/* Wrong: element of element */
.card__header__title {
  /* This creates tight coupling */
}

/* Correct: nested elements */
.card__header {
  /* block element */
}

.card__title {
  /* block element */
}
```

### 2. Modifier on Element Only (Wrong)
```css
/* Wrong: modifier without base class */
.card__title--large {
  /* Missing .card__title base styles */
}

/* Correct: modifier with base class */
.card__title {
  font-size: 1rem;
}

.card__title--large {
  font-size: 1.25rem;
}
```

### 3. Over-nesting (Wrong)
```css
/* Wrong: too specific */
.header .nav .nav__list .nav__item .nav__link {
  /* Tightly coupled to structure */
}

/* Correct: flat BEM */
.nav__link {
  /* Independent of structure */
}
```

## BEM Tools and Automation

### BEM Tools
```bash
# Install BEM tools
npm install -g bem

# Create BEM project structure
bem create my-block
bem create my-block__my-element
bem create my-block--my-modifier
```

### PostCSS BEM Plugin
```javascript
// postcss.config.js
module.exports = {
  plugins: [
    require('postcss-bem')({
      style: 'bem',
      separators: {
        namespace: '-',
        descendent: '__',
        modifier: '--'
      }
    })
  ]
};
```

## Testing BEM Classes

```javascript
// Test BEM class generation
describe('BEM Utility', () => {
  it('should generate block class', () => {
    expect(bem('button')).toBe('button');
  });

  it('should generate element class', () => {
    expect(bem('button', 'text')).toBe('button__text');
  });

  it('should generate modifier class', () => {
    expect(bem('button', null, 'primary')).toBe('button--primary');
  });

  it('should generate element with modifier', () => {
    expect(bem('button', 'text', 'large')).toBe('button__text--large');
  });

  it('should handle multiple modifiers', () => {
    expect(bem('button', null, ['primary', 'large']))
      .toBe('button button--primary button--large');
  });
});
```

## Best Practices

1. **Flat structure**: Avoid deep nesting of elements
2. **Single responsibility**: Each block should have one responsibility
3. **Consistent naming**: Follow naming conventions strictly
4. **Documentation**: Document your blocks and their modifiers
5. **Automation**: Use tools to generate BEM classes
6. **Testing**: Test your CSS classes and their combinations
7. **Performance**: BEM can help with CSS performance by avoiding deep selectors

## Interview Tips
- **Block**: Standalone component (`.block`)
- **Element**: Part of block (`.block__element`)
- **Modifier**: Variant of block/element (`.block--modifier`, `.block__element--modifier`)
- **Naming rules**: Lowercase, hyphens for words, double underscores/dashes for separators
- **Benefits**: Maintainable, scalable, avoids specificity wars
- **Common mistakes**: Element of element, modifier without base class
- **Alternatives**: OOCSS, SMACSS, CSS Modules
- **Tools**: PostCSS plugins, BEM tools, CSS preprocessors