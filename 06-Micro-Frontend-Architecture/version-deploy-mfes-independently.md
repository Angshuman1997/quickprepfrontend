# Independent Versioning and Deployment

## Simple Versioning Strategy

### Semantic Versioning
```
MAJOR.MINOR.PATCH
- MAJOR: Breaking changes (2.0.0)
- MINOR: New features (1.1.0) 
- PATCH: Bug fixes (1.0.1)
```

### Version in URLs
```javascript
// Host app loads specific versions
const remotes = {
  products: 'products@//cdn.com/products/2.1.3/remoteEntry.js',
  cart: 'cart@//cdn.com/cart/1.0.5/remoteEntry.js',
  user: 'user@//cdn.com/user/3.2.1/remoteEntry.js'
};
```

## Independent Deployment Process

### Each MFE Has Its Own Pipeline
```yaml
# products-mfe CI/CD
name: Deploy Products MFE

on:
  push:
    branches: [main]

jobs:
  deploy:
    steps:
      - run: npm version patch  # Auto-increment version
      - run: npm run build
      - run: aws s3 cp dist/ s3://cdn/products/$VERSION/
      - run: update-host-config products $VERSION
```

### Host App Updates Automatically
```javascript
// Host app fetches latest versions
const getLatestVersions = async () => {
  const response = await fetch('/api/mfe-versions');
  return response.json();
  // Returns: { products: '2.1.4', cart: '1.0.5' }
};
```

## Rollback Strategy

### Quick Rollback
```javascript
// Version registry API
POST /rollback
{
  "mfe": "products",
  "version": "2.1.3"  // Previous stable version
}
```

### Zero-Downtime Deployment
```javascript
// Load new version, keep old as fallback
const loadMfe = async (name) => {
  try {
    // Try new version first
    return await import(`//cdn.com/${name}/${newVersion}/app.js`);
  } catch (error) {
    // Fallback to old version
    return await import(`//cdn.com/${name}/${oldVersion}/app.js`);
  }
};
```

## Interview Questions & Answers

**Q: How do micro frontends deploy independently?**
**A:** "Each micro frontend has its own CI/CD pipeline that builds, versions, and deploys it separately. The host app loads the latest versions via configuration."

**Q: How do you handle versioning?**
**A:** "Using semantic versioning (MAJOR.MINOR.PATCH) and including version numbers in the remote URLs so the host knows exactly which version to load."

**Q: What about rollbacks?**
**A:** "Each deployment keeps the previous version available. If issues occur, we can quickly rollback by updating the version registry to point to the previous stable version."

**Q: How does the host app know which versions to load?**
**A:** "Through a version registry service that the host app queries at runtime, or through environment variables updated during deployment."

## Summary
- **Independent Pipelines**: Each MFE deploys separately
- **Version Registry**: Tracks which versions are active
- **Semantic Versioning**: Clear upgrade/downgrade paths
- **Rollback Ready**: Previous versions stay available
- **Zero Downtime**: New versions load without breaking old ones