# Flexbox vs CSS Grid

## Simple Choice Guide

### Flexbox (1D Layouts)
- **Use for**: Rows or columns only
- **Best for**: Navigation, cards, forms, centering
- **Example**: `display: flex; justify-content: space-between;`

### CSS Grid (2D Layouts)
- **Use for**: Rows AND columns together
- **Best for**: Page layouts, galleries, complex grids
- **Example**: `display: grid; grid-template-columns: 1fr 1fr 1fr;`

## When to Use Flexbox

### Navigation Bars
```css
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.nav-links {
  display: flex;
  gap: 2rem;
}
```

### Card Components
```css
.card {
  display: flex;
  flex-direction: column;
}

.card-content {
  flex-grow: 1; /* Takes available space */
}

.card-actions {
  margin-top: auto; /* Sticks to bottom */
}
```

### Centering Content
```css
.center-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
```

## When to Use CSS Grid

### Page Layouts
```css
.page-layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 200px 1fr;
  min-height: 100vh;
}
```

### Photo Galleries
```css
.photo-gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
}
```

### Dashboard Widgets
```css
.dashboard {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: repeat(3, 200px);
  gap: 1rem;
}

.widget-large {
  grid-column: span 2; /* Spans 2 columns */
  grid-row: span 2;    /* Spans 2 rows */
}
```

## Quick Comparison

| Feature | Flexbox | CSS Grid |
|---------|---------|----------|
| **Dimensions** | 1D (row or column) | 2D (rows + columns) |
| **Content Size** | Adapts to content | Fixed or flexible |
| **Use Case** | Components, alignment | Layouts, positioning |
| **Browser Support** | IE11+ | IE11+ (limited) |

## Interview Questions & Answers

**Q: When do you use Flexbox vs CSS Grid?**
**A:** "Use Flexbox for one-dimensional layouts like navigation bars or centering content. Use CSS Grid for two-dimensional layouts like page structures or photo galleries."

**Q: Can you use both together?**
**A:** "Yes! Often Grid for the overall page layout, and Flexbox inside components for content alignment."

**Q: What's easier to learn?**
**A:** "Flexbox is simpler for basic layouts. CSS Grid has more power but steeper learning curve."

**Q: Which has better browser support?**
**A:** "Both have excellent support in modern browsers. Flexbox works in IE11, Grid has limited IE11 support."

**Q: Performance difference?**
**A:** "Minimal difference. Choose based on layout needs, not performance."

## Summary
- **Flexbox**: Content-first, flexible, great for components
- **CSS Grid**: Layout-first, powerful, great for pages
- **Together**: Best of both worlds for modern layouts
- **Choice**: Depends on whether you need 1D or 2D layout