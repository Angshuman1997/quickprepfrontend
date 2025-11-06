# How do you version and deploy MFEs independently?

## Question
How do you version and deploy MFEs independently?

# How do you version and deploy MFEs independently?

## Question
How do you version and deploy MFEs independently?

## Answer

Independent versioning and deployment is one of the key benefits of micro frontends. Here's how to implement it effectively:

## 1. Semantic Versioning Strategy

### Version Numbering Convention
```
MAJOR.MINOR.PATCH
- MAJOR: Breaking changes (API changes, removed features)
- MINOR: New features (backward compatible)
- PATCH: Bug fixes (backward compatible)
```

### Package.json Versioning
```json
// products-mfe/package.json
{
  "name": "products-mfe",
  "version": "2.1.3",
  "scripts": {
    "version:patch": "npm version patch",
    "version:minor": "npm version minor",
    "version:major": "npm version major"
  }
}
```

## 2. Module Federation Versioning

### Version in Remote URLs
```javascript
// host-app/webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      remotes: {
        // Version-specific remote URLs
        products: `products@//cdn.example.com/products/2.1.3/remoteEntry.js`,
        cart: `cart@//cdn.example.com/cart/1.0.5/remoteEntry.js`,
        user: `user@//cdn.example.com/user/3.2.1/remoteEntry.js`
      },
      shared: ['react', 'react-dom']
    })
  ]
};
```

### Dynamic Version Loading
```javascript
// src/config/remotes.js
export const REMOTE_CONFIG = {
  products: {
    url: process.env.PRODUCTS_MFE_URL || '//cdn.example.com/products/2.1.3/remoteEntry.js',
    version: '2.1.3'
  },
  cart: {
    url: process.env.CART_MFE_URL || '//cdn.example.com/cart/1.0.5/remoteEntry.js',
    version: '1.0.5'
  }
};
```

```javascript
// webpack.config.js
const { REMOTE_CONFIG } = require('./src/config/remotes');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      remotes: Object.fromEntries(
        Object.entries(REMOTE_CONFIG).map(([name, config]) => [
          name,
          `${name}@${config.url}`
        ])
      ),
      shared: ['react', 'react-dom']
    })
  ]
};
```

## 3. CI/CD Pipeline for Independent Deployment

### GitHub Actions Workflow
```yaml
# .github/workflows/deploy.yml
name: Deploy MFE

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Build
      run: npm run build
    
    - name: Generate version
      run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_ENV
    
    - name: Deploy to CDN
      run: |
        aws s3 cp dist/ s3://my-cdn-bucket/products/$VERSION/ --recursive
        aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_ID --paths "/products/$VERSION/*"
    
    - name: Update host configuration
      run: |
        # Update host app's remote configuration
        curl -X POST $HOST_UPDATE_ENDPOINT \
          -H "Content-Type: application/json" \
          -d "{\"mfe\": \"products\", \"version\": \"$VERSION\"}"
```

### Jenkins Pipeline
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Version') {
            steps {
                script {
                    def version = sh(script: 'npm version patch --no-git-tag-version', returnStdout: true).trim()
                    env.VERSION = version.replace('v', '')
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }
        
        stage('Deploy') {
            steps {
                sh "aws s3 cp dist/ s3://my-cdn/products/${env.VERSION}/ --recursive"
                sh "aws cloudfront create-invalidation --distribution-id ${CLOUDFRONT_ID} --paths '/products/${env.VERSION}/*'"
            }
        }
        
        stage('Update Host') {
            steps {
                sh """
                    curl -X POST \${HOST_UPDATE_URL} \\
                      -H "Content-Type: application/json" \\
                      -d '{"mfe': 'products', 'version': '\${VERSION}'}'
                """
            }
        }
    }
}
```

## 4. Runtime Version Management

### Version Registry Service
```javascript
// version-registry/src/server.js
const express = require('express');
const app = express();

const versions = {
  products: '2.1.3',
  cart: '1.0.5',
  user: '3.2.1'
};

app.get('/versions', (req, res) => {
  res.json(versions);
});

app.post('/update-version', express.json(), (req, res) => {
  const { mfe, version } = req.body;
  versions[mfe] = version;
  res.json({ success: true });
});

app.listen(3000, () => {
  console.log('Version registry running on port 3000');
});
```

### Dynamic Remote Loading
```javascript
// src/utils/remoteLoader.js
export class RemoteLoader {
  constructor(registryUrl) {
    this.registryUrl = registryUrl;
    this.loadedRemotes = new Map();
  }

  async loadRemote(name) {
    if (this.loadedRemotes.has(name)) {
      return this.loadedRemotes.get(name);
    }

    try {
      // Fetch latest version from registry
      const versions = await this.fetchVersions();
      const version = versions[name];
      
      // Load remote dynamically
      const remoteUrl = `//cdn.example.com/${name}/${version}/remoteEntry.js`;
      await this.loadScript(remoteUrl);
      
      const remote = await window[name].get('./App');
      this.loadedRemotes.set(name, remote);
      
      return remote;
    } catch (error) {
      console.error(`Failed to load remote ${name}:`, error);
      throw error;
    }
  }

  async fetchVersions() {
    const response = await fetch(`${this.registryUrl}/versions`);
    return response.json();
  }

  loadScript(url) {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.src = url;
      script.onload = resolve;
      script.onerror = reject;
      document.head.appendChild(script);
    });
  }
}
```

## 5. Rollback and Rollforward Strategies

### Version Rollback
```javascript
// src/utils/versionManager.js
export class VersionManager {
  constructor(registryUrl) {
    this.registryUrl = registryUrl;
    this.versionHistory = new Map();
  }

  async rollback(mfeName, targetVersion) {
    try {
      // Update registry with target version
      await this.updateVersion(mfeName, targetVersion);
      
      // Clear cached remote
      this.clearCache(mfeName);
      
      // Force page reload to load new version
      window.location.reload();
      
    } catch (error) {
      console.error(`Rollback failed for ${mfeName}:`, error);
    }
  }

  async updateVersion(mfeName, version) {
    const response = await fetch(`${this.registryUrl}/update-version`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ mfe: mfeName, version })
    });
    return response.json();
  }

  clearCache(mfeName) {
    // Clear from local storage or service worker cache
    localStorage.removeItem(`mfe_${mfeName}_version`);
  }
}
```

## 6. Feature Flags and Canary Deployments

### Feature Flag Integration
```javascript
// src/config/features.js
export const FEATURES = {
  NEW_PRODUCTS_UI: {
    enabled: true,
    version: '2.2.0',
    rollout: 50 // 50% of users
  },
  ENHANCED_CART: {
    enabled: false,
    version: '1.1.0'
  }
};
```

```javascript
// src/components/ProductList.js
import React from 'react';
import { FEATURES } from '../config/features';

const ProductList = () => {
  const shouldUseNewUI = FEATURES.NEW_PRODUCTS_UI.enabled && 
    Math.random() * 100 < FEATURES.NEW_PRODUCTS_UI.rollout;

  if (shouldUseNewUI) {
    // Load new version
    return <NewProductList />;
  }

  // Load current version
  return <CurrentProductList />;
};
```

## 7. Monitoring and Health Checks

### Health Check Implementation
```javascript
// src/utils/healthCheck.js
export class HealthChecker {
  async checkMfeHealth(mfeName, version) {
    try {
      const response = await fetch(`//cdn.example.com/${mfeName}/${version}/health`);
      const health = await response.json();
      
      return {
        status: health.status === 'ok' ? 'healthy' : 'unhealthy',
        version,
        timestamp: Date.now()
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        version,
        error: error.message,
        timestamp: Date.now()
      };
    }
  }

  async checkAllMfes() {
    const versions = await this.fetchVersions();
    const healthChecks = await Promise.all(
      Object.entries(versions).map(([name, version]) => 
        this.checkMfeHealth(name, version)
      )
    );
    
    return healthChecks.reduce((acc, check) => {
      acc[check.name] = check;
      return acc;
    }, {});
  }
}
```

## Best Practices

1. **Semantic Versioning**: Follow MAJOR.MINOR.PATCH convention
2. **Automated Pipelines**: CI/CD for each MFE
3. **Version Registry**: Centralized version management
4. **Rollback Strategy**: Quick rollback capabilities
5. **Health Monitoring**: Monitor MFE health post-deployment
6. **Feature Flags**: Gradual rollout of new versions
7. **Caching Strategy**: Proper cache invalidation for updates

## Interview Tips
- **Independent deployment**: Each MFE can be deployed without affecting others
- **Version registry**: Service that tracks which versions are active
- **Semantic versioning**: Breaking changes increment major version
- **Rollback**: Ability to quickly revert to previous version
- **Feature flags**: Gradual rollout and A/B testing
- **Health checks**: Monitor MFE status after deployment
- **CDN**: Fast delivery of MFE bundles to users