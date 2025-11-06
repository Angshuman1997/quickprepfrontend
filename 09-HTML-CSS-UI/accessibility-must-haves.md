# Accessibility must-haves (aria, labels, focus control)

## Question
Accessibility must-haves (aria, labels, focus control).

# Accessibility must-haves (aria, labels, focus control)

## Question
Accessibility must-haves (aria, labels, focus control).

## Answer

Accessibility (a11y) ensures that web applications are usable by people with disabilities. Here are the essential accessibility practices every developer should know.

## Semantic HTML

### Proper Heading Hierarchy
```html
<!-- Good: Proper heading hierarchy -->
<h1>Main Page Title</h1>
<section>
  <h2>Section Title</h2>
  <p>Content...</p>
  <h3>Subsection Title</h3>
  <p>More content...</p>
</section>

<!-- Bad: Skipping heading levels -->
<h1>Main Title</h1>
<h3>Section Title</h3> <!-- Skips h2 -->
```

### Semantic Elements
```html
<!-- Use semantic elements instead of divs -->
<header>Site header</header>
<nav>Navigation menu</nav>
<main>Main content</main>
<section>Content section</section>
<article>Article content</article>
<aside>Sidebar content</aside>
<footer>Site footer</footer>
```

## ARIA Attributes

### ARIA Labels and Descriptions
```html
<!-- aria-label for custom labels -->
<button aria-label="Close dialog">×</button>

<!-- aria-labelledby for referencing labels -->
<div id="email-label">Email Address</div>
<input type="email" aria-labelledby="email-label">

<!-- aria-describedby for additional descriptions -->
<input type="password" 
       aria-describedby="password-help"
       aria-label="Password">
<div id="password-help">
  Password must be at least 8 characters long
</div>
```

### ARIA States and Properties
```html
<!-- ARIA states for dynamic content -->
<button aria-expanded="false" aria-controls="menu">
  Menu
</button>
<ul id="menu" aria-hidden="true">
  <li><a href="#home">Home</a></li>
  <li><a href="#about">About</a></li>
</ul>

<!-- ARIA properties for relationships -->
<div role="tablist">
  <button role="tab" 
          aria-selected="true" 
          aria-controls="panel1"
          id="tab1">
    Tab 1
  </button>
  <button role="tab" 
          aria-selected="false" 
          aria-controls="panel2"
          id="tab2">
    Tab 2
  </button>
</div>

<div role="tabpanel" 
     aria-labelledby="tab1" 
     id="panel1">
  Content for tab 1
</div>
```

### ARIA Live Regions
```html
<!-- Announce dynamic content changes -->
<div aria-live="polite" aria-atomic="true" id="status">
  <!-- Status messages appear here -->
</div>

<!-- For errors, use assertive -->
<div aria-live="assertive" role="alert" id="error-message">
  <!-- Error messages appear here -->
</div>
```

## Form Accessibility

### Proper Labels
```html
<!-- Explicit label association -->
<label for="username">Username:</label>
<input type="text" id="username" name="username">

<!-- Implicit label association -->
<label>
  Email:
  <input type="email" name="email">
</label>

<!-- aria-label for icon buttons -->
<button aria-label="Search">
  <svg>...</svg>
</button>
```

### Form Validation
```html
<!-- Required fields -->
<input type="email" 
       required 
       aria-required="true"
       aria-describedby="email-error">

<!-- Error messages -->
<div id="email-error" 
     role="alert" 
     aria-live="polite">
  Please enter a valid email address
</div>

<!-- Fieldsets for grouping -->
<fieldset>
  <legend>Shipping Address</legend>
  <label for="street">Street:</label>
  <input type="text" id="street" name="street">
  
  <label for="city">City:</label>
  <input type="text" id="city" name="city">
</fieldset>
```

## Focus Management

### Focus Indicators
```css
/* Visible focus indicators */
button:focus,
input:focus,
select:focus,
textarea:focus {
  outline: 2px solid #007acc;
  outline-offset: 2px;
}

/* High contrast focus for better visibility */
button:focus-visible {
  outline: 3px solid #000;
  outline-offset: 2px;
}
```

### Focus Trapping
```javascript
// Focus trap for modals
class FocusTrap {
  constructor(element) {
    this.element = element;
    this.focusableElements = this.getFocusableElements();
    this.firstElement = this.focusableElements[0];
    this.lastElement = this.focusableElements[this.focusableElements.length - 1];
  }

  getFocusableElements() {
    const selectors = [
      'a[href]',
      'button:not([disabled])',
      'input:not([disabled])',
      'select:not([disabled])',
      'textarea:not([disabled])',
      '[tabindex]:not([tabindex="-1"])'
    ];
    
    return this.element.querySelectorAll(selectors.join(', '));
  }

  trapFocus() {
    this.element.addEventListener('keydown', (e) => {
      if (e.key === 'Tab') {
        if (e.shiftKey) {
          // Shift + Tab
          if (document.activeElement === this.firstElement) {
            e.preventDefault();
            this.lastElement.focus();
          }
        } else {
          // Tab
          if (document.activeElement === this.lastElement) {
            e.preventDefault();
            this.firstElement.focus();
          }
        }
      }
    });
  }
}

// Usage
const modal = document.getElementById('modal');
const focusTrap = new FocusTrap(modal);
focusTrap.trapFocus();
```

### Managing Focus Programmatically
```javascript
// Focus management for dynamic content
function openModal() {
  const modal = document.getElementById('modal');
  modal.style.display = 'block';
  
  // Store current focus
  const previouslyFocused = document.activeElement;
  
  // Focus first focusable element in modal
  const firstInput = modal.querySelector('input');
  if (firstInput) {
    firstInput.focus();
  }
  
  // Store for later restoration
  modal.previouslyFocused = previouslyFocused;
}

function closeModal() {
  const modal = document.getElementById('modal');
  modal.style.display = 'none';
  
  // Restore previous focus
  if (modal.previouslyFocused) {
    modal.previouslyFocused.focus();
  }
}
```

## Keyboard Navigation

### Keyboard Event Handling
```javascript
// Custom dropdown with keyboard support
class AccessibleDropdown {
  constructor(button, menu) {
    this.button = button;
    this.menu = menu;
    this.isOpen = false;
    this.init();
  }

  init() {
    this.button.addEventListener('click', () => this.toggle());
    this.button.addEventListener('keydown', (e) => this.handleKeydown(e));
    this.menu.addEventListener('keydown', (e) => this.handleMenuKeydown(e));
  }

  toggle() {
    this.isOpen = !this.isOpen;
    this.button.setAttribute('aria-expanded', this.isOpen);
    this.menu.setAttribute('aria-hidden', !this.isOpen);
  }

  handleKeydown(e) {
    switch(e.key) {
      case 'Enter':
      case ' ':
      case 'ArrowDown':
        e.preventDefault();
        this.open();
        this.focusFirstItem();
        break;
      case 'ArrowUp':
        e.preventDefault();
        this.open();
        this.focusLastItem();
        break;
      case 'Escape':
        this.close();
        break;
    }
  }

  handleMenuKeydown(e) {
    const items = this.menu.querySelectorAll('[role="menuitem"]');
    const currentIndex = Array.from(items).indexOf(document.activeElement);
    
    switch(e.key) {
      case 'ArrowDown':
        e.preventDefault();
        const nextIndex = (currentIndex + 1) % items.length;
        items[nextIndex].focus();
        break;
      case 'ArrowUp':
        e.preventDefault();
        const prevIndex = currentIndex - 1 < 0 ? items.length - 1 : currentIndex - 1;
        items[prevIndex].focus();
        break;
      case 'Escape':
        this.close();
        this.button.focus();
        break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        this.selectItem(document.activeElement);
        break;
    }
  }
}
```

## Color and Contrast

### Color Contrast Requirements
```css
/* WCAG AA compliant colors */
/* Normal text: 4.5:1 contrast ratio */
/* Large text: 3:1 contrast ratio */

.button-primary {
  background-color: #007acc; /* Blue */
  color: #ffffff; /* White text */
  /* Contrast ratio: 8.6:1 ✓ */
}

.button-secondary {
  background-color: #f8f9fa; /* Light gray */
  color: #212529; /* Dark text */
  border: 1px solid #dee2e6;
  /* Contrast ratio: 12.6:1 ✓ */
}

/* Focus indicators with high contrast */
.focus-indicator {
  outline: 3px solid #000000;
  outline-offset: 2px;
}
```

### Don't Rely on Color Alone
```html
<!-- Bad: Color-only indication -->
<div class="status" style="color: red;">Error</div>
<div class="status" style="color: green;">Success</div>

<!-- Good: Color + text/symbol -->
<div class="status error">
  <span aria-hidden="true">❌</span>
  Error: Please check your input
</div>

<div class="status success">
  <span aria-hidden="true">✅</span>
  Success: Data saved successfully
</div>
```

## Screen Reader Support

### Screen Reader Only Content
```css
/* Hide content visually but keep for screen readers */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* Show on focus for skip links */
.sr-only:focus {
  position: static;
  width: auto;
  height: auto;
  padding: 0.5rem;
  margin: 0;
  overflow: visible;
  clip: auto;
  white-space: normal;
  background: #000;
  color: #fff;
}
```

### Skip Links
```html
<!-- Skip to main content -->
<a href="#main-content" class="sr-only sr-only-focusable">
  Skip to main content
</a>

<!-- Skip navigation -->
<a href="#navigation" class="sr-only sr-only-focusable">
  Skip to navigation
</a>

<main id="main-content">
  <!-- Main content -->
</main>
```

## Testing Accessibility

### Automated Testing
```javascript
// Basic accessibility checks
function checkAccessibility() {
  const issues = [];

  // Check for missing alt text
  const images = document.querySelectorAll('img');
  images.forEach(img => {
    if (!img.alt && !img.hasAttribute('aria-hidden')) {
      issues.push(`Image missing alt text: ${img.src}`);
    }
  });

  // Check for missing labels
  const inputs = document.querySelectorAll('input, select, textarea');
  inputs.forEach(input => {
    if (!input.labels.length && !input.getAttribute('aria-label')) {
      issues.push(`Form control missing label: ${input.name}`);
    }
  });

  // Check heading hierarchy
  const headings = document.querySelectorAll('h1, h2, h3, h4, h5, h6');
  let lastLevel = 0;
  headings.forEach(heading => {
    const level = parseInt(heading.tagName.charAt(1));
    if (level - lastLevel > 1) {
      issues.push(`Skipped heading level: ${heading.textContent}`);
    }
    lastLevel = level;
  });

  return issues;
}

// Usage
const accessibilityIssues = checkAccessibility();
console.log('Accessibility Issues:', accessibilityIssues);
```

### Manual Testing Checklist
- [ ] Tab through all interactive elements
- [ ] Test with screen reader (NVDA, JAWS, VoiceOver)
- [ ] Check color contrast with tools
- [ ] Test keyboard-only navigation
- [ ] Verify focus indicators are visible
- [ ] Test form validation messages
- [ ] Check ARIA labels and descriptions

## React Accessibility

### Accessible React Components
```jsx
// Accessible Button
const Button = ({ children, onClick, disabled, ...props }) => (
  <button
    onClick={onClick}
    disabled={disabled}
    aria-disabled={disabled}
    {...props}
  >
    {children}
  </button>
);

// Accessible Modal
const Modal = ({ isOpen, onClose, title, children }) => {
  const modalRef = useRef();
  
  useEffect(() => {
    if (isOpen) {
      // Focus trap
      modalRef.current.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div role="dialog" 
         aria-modal="true" 
         aria-labelledby="modal-title"
         ref={modalRef}
         tabIndex={-1}>
      <div className="modal-header">
        <h2 id="modal-title">{title}</h2>
        <button 
          onClick={onClose}
          aria-label="Close modal"
        >
          ×
        </button>
      </div>
      <div className="modal-body">
        {children}
      </div>
    </div>
  );
};
```

## Best Practices

1. **Semantic HTML**: Use proper HTML elements for their intended purpose
2. **ARIA sparingly**: Only add ARIA when native HTML doesn't provide the semantics
3. **Keyboard navigation**: Ensure all functionality is available via keyboard
4. **Focus management**: Provide visible focus indicators and manage focus programmatically
5. **Color contrast**: Meet WCAG guidelines for color contrast
6. **Screen readers**: Test with actual screen readers
7. **Progressive enhancement**: Build accessible foundations first
8. **Regular testing**: Include accessibility in your testing routine

## Interview Tips
- **WCAG Guidelines**: Web Content Accessibility Guidelines (A, AA, AAA levels)
- **Screen readers**: NVDA, JAWS, VoiceOver, TalkBack
- **Semantic HTML**: Proper use of headings, landmarks, form elements
- **ARIA attributes**: When and how to use aria-label, aria-describedby, etc.
- **Focus management**: Focus trapping, focus restoration, visible indicators
- **Keyboard navigation**: Tab order, keyboard event handling
- **Color contrast**: 4.5:1 for normal text, 3:1 for large text
- **Testing tools**: axe, Lighthouse, WAVE, screen readers