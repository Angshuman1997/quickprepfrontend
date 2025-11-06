# Explain Webpack Module Federation

## Question
Explain Webpack Module Federation.

## Answer

Webpack Module Federation is a revolutionary feature introduced in Webpack 5 that enables the creation of micro frontend architectures by allowing JavaScript applications to dynamically load code from other applications at runtime. It enables true code sharing and composition across different applications, making it possible to build and deploy micro frontends independently while sharing dependencies and components seamlessly.

## How Module Federation Works

### **Core Concepts**

**Host vs Remote:**
- **Host Application:** The application that consumes federated modules
- **Remote Application:** The application that exposes modules to be consumed

**Key Components:**
- **RemoteEntry:** A manifest file that describes what modules are available
- **Container:** Runtime that manages module loading and sharing
- **Shared Dependencies:** Libraries that can be shared between applications

### **Basic Architecture**

```javascript
// Remote Application (exposes modules)
const remoteConfig = {
  name: 'remote_app',
  filename: 'remoteEntry.js',  // Manifest file
  exposes: {
    './Button': './src/components/Button',
    './Header': './src/components/Header'
  },
  shared: ['react', 'react-dom']  // Shared dependencies
};

// Host Application (consumes modules)
const hostConfig = {
  name: 'host_app',
  remotes: {
    remoteApp: 'remote_app@http://localhost:3001/remoteEntry.js'
  },
  shared: ['react', 'react-dom']
};
```

## Configuration Deep Dive

### 1. **Host Application Configuration**

```javascript
// webpack.config.js for host/shell application
const { ModuleFederationPlugin } = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  // ... other webpack config
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',  // Name of this application

      // Applications this app can consume from
      remotes: {
        // Format: 'alias': 'remoteName@remoteEntryUrl'
        header: 'header_app@http://localhost:3001/remoteEntry.js',
        dashboard: 'dashboard_app@http://localhost:3002/remoteEntry.js',
        userProfile: 'user_app@http://localhost:3003/remoteEntry.js'
      },

      // Dependencies shared with remote applications
      shared: {
        react: {
          singleton: true,  // Only one instance of React
          eager: true      // Load immediately
        },
        'react-dom': {
          singleton: true,
          eager: true
        },
        lodash: {
          singleton: true,
          requiredVersion: '^4.17.0'  // Version constraints
        }
      }
    })
  ]
};
```

### 2. **Remote Application Configuration**

```javascript
// webpack.config.js for remote/header application
const { ModuleFederationPlugin } = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  // ... other webpack config
  plugins: [
    new ModuleFederationPlugin({
      name: 'header_app',  // Name other apps will reference

      // File where manifest will be generated
      filename: 'remoteEntry.js',

      // Modules this app exposes to others
      exposes: {
        './Header': './src/components/Header/Header.jsx',
        './UserMenu': './src/components/UserMenu/UserMenu.jsx',
        './Navigation': './src/components/Navigation/Navigation.jsx'
      },

      // Dependencies shared with host applications
      shared: {
        react: {
          singleton: true,
          eager: true
        },
        'react-dom': {
          singleton: true,
          eager: true
        },
        '@company/design-system': {
          singleton: true,
          requiredVersion: '^1.0.0'
        }
      }
    })
  ]
};
```

### 3. **Bidirectional Federation**

**Applications can be both host and remote:**

```javascript
// App A - both host and remote
const appAConfig = {
  name: 'app_a',
  filename: 'remoteEntry.js',
  exposes: {
    './ComponentA': './src/ComponentA'
  },
  remotes: {
    appB: 'app_b@http://localhost:3002/remoteEntry.js'
  },
  shared: ['react']
};

// App B - both host and remote
const appBConfig = {
  name: 'app_b',
  filename: 'remoteEntry.js',
  exposes: {
    './ComponentB': './src/ComponentB'
  },
  remotes: {
    appA: 'app_a@http://localhost:3001/remoteEntry.js'
  },
  shared: ['react']
};
```

## Usage in Code

### 1. **Consuming Remote Modules**

```javascript
// In host application
import React, { Suspense } from 'react';

// Dynamic import of remote module
const Header = React.lazy(() => import('header/Header'));
const UserMenu = React.lazy(() => import('header/UserMenu'));
const Dashboard = React.lazy(() => import('dashboard/Dashboard'));

function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading header...</div>}>
        <Header />
      </Suspense>

      <Suspense fallback={<div>Loading user menu...</div>}>
        <UserMenu />
      </Suspense>

      <Suspense fallback={<div>Loading dashboard...</div>}>
        <Dashboard />
      </Suspense>
    </div>
  );
}
```

### 2. **Dynamic Remote Loading**

```javascript
// Load remote modules dynamically based on routes
function loadRemoteComponent(remoteName, moduleName) {
  return React.lazy(() =>
    import(`@mf/${remoteName}/${moduleName}`)
      .catch(error => {
        console.error(`Failed to load ${remoteName}/${moduleName}:`, error);
        return import('./ErrorFallback');
      })
  );
}

// Usage in routing
const routes = [
  {
    path: '/dashboard',
    component: loadRemoteComponent('dashboard', 'Dashboard')
  },
  {
    path: '/profile',
    component: loadRemoteComponent('userProfile', 'Profile')
  }
];
```

### 3. **Error Handling and Fallbacks**

```javascript
// Graceful degradation when remote fails
function RemoteWrapper({ children, fallback }) {
  const [hasError, setHasError] = React.useState(false);

  React.useEffect(() => {
    // Check if remote is available
    const checkRemote = async () => {
      try {
        await import('header/remoteEntry.js');
      } catch (error) {
        console.warn('Remote header not available, using fallback');
        setHasError(true);
      }
    };

    checkRemote();
  }, []);

  if (hasError) {
    return fallback;
  }

  return children;
}

// Usage
<RemoteWrapper fallback={<LocalHeader />}>
  <Suspense fallback={<LoadingSpinner />}>
    <RemoteHeader />
  </Suspense>
</RemoteWrapper>
```

## Shared Dependencies Management

### 1. **Singleton Dependencies**

**Ensure only one instance of libraries like React:**

```javascript
shared: {
  react: {
    singleton: true,        // Only one React instance
    eager: true,           // Load immediately, don't lazy load
    requiredVersion: '^18.0.0'  // Version constraints
  },
  'react-dom': {
    singleton: true,
    eager: true,
    requiredVersion: '^18.0.0'
  }
}
```

**Benefits:**
- **Memory efficiency:** Single instance of large libraries
- **Consistency:** Same library version across applications
- **Performance:** No duplicate code in bundles

### 2. **Version Management**

**Handle version conflicts gracefully:**

```javascript
shared: {
  lodash: {
    singleton: true,
    requiredVersion: '^4.17.0',  // Accept patch updates
    version: '4.17.21'          // Preferred version
  },
  'date-fns': {
    // Allow multiple versions if needed
    singleton: false,
    requiredVersion: false
  }
}
```

### 3. **Eager vs Lazy Loading**

```javascript
shared: {
  // Load immediately with main bundle
  react: {
    singleton: true,
    eager: true  // Loads with host app
  },

  // Load when first remote needs it
  lodash: {
    singleton: true,
    eager: false  // Lazy loaded
  }
}
```

## Advanced Patterns

### 1. **Dynamic Remote Configuration**

**Load remotes based on environment or user permissions:**

```javascript
// Dynamic remote configuration
function getRemoteConfig() {
  const remotes = {};

  // Load different remotes based on environment
  if (process.env.NODE_ENV === 'development') {
    remotes.header = 'header@http://localhost:3001/remoteEntry.js';
    remotes.dashboard = 'dashboard@http://localhost:3002/remoteEntry.js';
  } else {
    // Production URLs
    remotes.header = 'header@https://cdn.example.com/header/remoteEntry.js';
    remotes.dashboard = 'dashboard@https://cdn.example.com/dashboard/remoteEntry.js';
  }

  // Load remotes based on user permissions
  if (userHasPermission('admin')) {
    remotes.adminPanel = 'admin@https://cdn.example.com/admin/remoteEntry.js';
  }

  return remotes;
}

const webpackConfig = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: getRemoteConfig(),
      // ... other config
    })
  ]
};
```

### 2. **Federated Types**

**Share TypeScript types across applications:**

```typescript
// types/federated-types.ts
export interface User {
  id: string;
  name: string;
  email: string;
}

export interface Product {
  id: string;
  name: string;
  price: number;
}

// Expose types from remote
const remoteConfig = {
  exposes: {
    './types': './src/types/federated-types.ts',
    './UserComponent': './src/components/UserComponent.tsx'
  }
};

// Consume types in host
import type { User } from 'remote/types';

function HostComponent() {
  const [user, setUser] = useState<User | null>(null);
  // TypeScript knows the User shape
}
```

### 3. **Runtime Remote Discovery**

**Discover and load remotes dynamically:**

```javascript
// Remote registry service
class RemoteRegistry {
  private remotes = new Map();

  register(name: string, url: string) {
    this.remotes.set(name, url);
  }

  async loadRemote(name: string) {
    const url = this.remotes.get(name);
    if (!url) throw new Error(`Remote ${name} not found`);

    // Dynamically add to webpack remotes
    await __webpack_init_sharing__('default');
    await __webpack_share_scopes__['default'];

    const container = window[name] || await loadScript(url);
    await container.init(__webpack_share_scopes__['default']);

    return container;
  }
}

// Usage
const registry = new RemoteRegistry();
registry.register('dynamicFeature', 'https://cdn.example.com/feature/remoteEntry.js');

// Load on demand
const DynamicFeature = React.lazy(() =>
  registry.loadRemote('dynamicFeature')
    .then(container => container.get('./Feature'))
);
```

## Performance Optimization

### 1. **Bundle Splitting**

**Optimize bundle sizes with federation:**

```javascript
// Large remote application
const remoteConfig = {
  exposes: {
    // Expose different parts separately
    './SmallComponent': './src/SmallComponent.tsx',    // 10KB
    './LargeDashboard': './src/LargeDashboard.tsx',    // 500KB
    './AdminPanel': './src/AdminPanel.tsx'             // 300KB
  },

  // Split chunks for better caching
  shared: {
    // Large libraries as shared
    'react-big-calendar': { singleton: true },
    'react-data-grid': { singleton: true }
  }
};
```

### 2. **Lazy Loading Strategy**

**Load remotes only when needed:**

```javascript
// Route-based lazy loading
const routes = [
  {
    path: '/dashboard',
    component: React.lazy(() =>
      import('dashboard/Dashboard').catch(() => import('./OfflineDashboard'))
    )
  },
  {
    path: '/admin',
    component: React.lazy(() =>
      // Load admin remote only for admin routes
      import('admin/AdminPanel')
    ),
    // Guard route for admin users
    guard: (user) => user.role === 'admin'
  }
];
```

### 3. **Caching Strategy**

**Leverage browser caching for remote entries:**

```javascript
// Cache remote entries with versioning
const remoteConfig = {
  remotes: {
    // Include version in URL for cache busting
    header: `header@${process.env.HEADER_VERSION}@https://cdn.example.com/header/remoteEntry.js`,
    dashboard: `dashboard@${process.env.DASHBOARD_VERSION}@https://cdn.example.com/dashboard/remoteEntry.js`
  }
};

// Service worker caching
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('remoteEntry.js')) {
    event.respondWith(
      caches.match(event.request)
        .then(response => response || fetch(event.request))
    );
  }
});
```

## Development and Debugging

### 1. **Development Setup**

**Hot reloading with federation:**

```javascript
// webpack.dev.js
const devConfig = {
  devServer: {
    port: 3000,
    historyApiFallback: true,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, PATCH, OPTIONS',
      'Access-Control-Allow-Headers': 'X-Requested-With, content-type, Authorization'
    }
  },

  plugins: [
    new ModuleFederationPlugin({
      // Development remotes
      remotes: {
        header: 'header@http://localhost:3001/remoteEntry.js',
        dashboard: 'dashboard@http://localhost:3002/remoteEntry.js'
      }
    })
  ]
};
```

### 2. **Debugging Tools**

**Debug federation issues:**

```javascript
// Debug remote loading
if (process.env.NODE_ENV === 'development') {
  window.__webpack_remotes__ = {}; // Track loaded remotes

  // Monkey patch to track loading
  const originalImport = window.import;
  window.import = function(...args) {
    console.log('Loading remote:', args[0]);
    return originalImport.apply(this, args);
  };
}

// Inspect shared scope
console.log('Shared scope:', __webpack_share_scopes__);

// Check loaded containers
console.log('Loaded containers:', Object.keys(window).filter(key =>
  key.includes('_container_')
));
```

### 3. **Error Boundaries**

**Handle federation errors gracefully:**

```typescript
class FederationErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log federation errors
    console.error('Federation error:', error, errorInfo);

    // Report to monitoring service
    reportError({
      type: 'federation_error',
      error: error.message,
      stack: error.stack,
      remote: this.props.remoteName
    });
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="federation-error">
          <h3>Failed to load {this.props.remoteName}</h3>
          <p>{this.state.error?.message}</p>
          <button onClick={() => window.location.reload()}>
            Reload Application
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
<FederationErrorBoundary remoteName="dashboard">
  <Suspense fallback={<LoadingSpinner />}>
    <RemoteDashboard />
  </Suspense>
</FederationErrorBoundary>
```

## Common Issues and Solutions

### 1. **Version Conflicts**

**Problem:** Different versions of shared dependencies.

**Solution:**
```javascript
shared: {
  react: {
    singleton: true,
    requiredVersion: '^18.2.0',  // Allow compatible versions
    version: '18.2.0'           // Preferred version
  }
}
```

### 2. **Loading Order Issues**

**Problem:** Remotes load in wrong order.

**Solution:**
```javascript
// Explicit loading order
async function loadRemotes() {
  // Load shared dependencies first
  await __webpack_init_sharing__('default');

  // Load remotes in dependency order
  const [shared, header, dashboard] = await Promise.all([
    import('shared/remoteEntry.js'),
    import('header/remoteEntry.js'),
    import('dashboard/remoteEntry.js')
  ]);

  // Initialize containers
  await shared.init(__webpack_share_scopes__.default);
  await header.init(__webpack_share_scopes__.default);
  await dashboard.init(__webpack_share_scopes__.default);
}
```

### 3. **CORS Issues**

**Problem:** Cross-origin remote loading fails.

**Solution:**
```javascript
// Development webpack dev server
devServer: {
  headers: {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, PATCH, OPTIONS',
    'Access-Control-Allow-Headers': 'X-Requested-With, content-type, Authorization'
  }
}

// Production CDN headers
// Ensure CDN serves remoteEntry.js with proper CORS headers
```

## Testing Module Federation

### 1. **Unit Testing**

```typescript
// Mock remote modules for testing
jest.mock('header/Header', () => ({
  __esModule: true,
  default: () => <div>Mock Header</div>
}));

describe('Host Application', () => {
  it('should render with remote components', () => {
    render(
      <Suspense fallback={<div>Loading...</div>}>
        <App />
      </Suspense>
    );

    expect(screen.getByText('Mock Header')).toBeInTheDocument();
  });
});
```

### 2. **Integration Testing**

```typescript
// Test federation loading
describe('Module Federation', () => {
  it('should load remote modules', async () => {
    // Mock webpack container
    window.header_app = {
      init: jest.fn().mockResolvedValue(),
      get: jest.fn().mockResolvedValue({ default: () => <div>Header</div> })
    };

    render(<FederatedApp />);

    await waitFor(() => {
      expect(screen.getByText('Header')).toBeInTheDocument();
    });
  });
});
```

### 3. **E2E Testing**

```typescript
// Cypress test for federation
describe('Federated Application', () => {
  it('should load all micro frontends', () => {
    cy.visit('/');

    // Wait for all remotes to load
    cy.get('[data-testid="header-loaded"]', { timeout: 10000 }).should('be.visible');
    cy.get('[data-testid="dashboard-loaded"]', { timeout: 10000 }).should('be.visible');

    // Test interaction between MFEs
    cy.get('[data-testid="header-user-menu"]').click();
    cy.get('[data-testid="dashboard-user-info"]').should('be.visible');
  });
});
```

## Common Interview Questions

### Q: What is Webpack Module Federation?

**A:** "Module Federation is a Webpack 5 feature that allows JavaScript applications to consume code from other applications at runtime. It enables micro frontend architectures by letting applications expose and consume modules dynamically, while sharing dependencies to avoid duplication."

### Q: How does Module Federation differ from traditional bundling?

**A:** "Traditional bundling creates self-contained bundles, while Module Federation creates interconnected applications that can share code at runtime. Applications can expose modules to be consumed by others, and shared dependencies are loaded only once, reducing bundle sizes and enabling independent deployments."

### Q: What are the key components of Module Federation?

**A:** "The key components are: 1) Remote applications that expose modules, 2) Host applications that consume remote modules, 3) RemoteEntry files that serve as manifests, 4) Shared dependencies that are loaded once and reused, and 5) Containers that manage the runtime loading and sharing."

### Q: How do you handle shared dependencies in Module Federation?

**A:** "I configure shared dependencies in both host and remote webpack configs, specifying singleton mode to ensure only one instance exists, eager loading for critical dependencies, and version constraints to prevent conflicts. This ensures libraries like React are shared efficiently across applications."

### Q: What are the challenges with Module Federation?

**A:** "Challenges include managing version conflicts between shared dependencies, handling CORS issues in development, coordinating deployment of interconnected applications, and debugging issues across different codebases. Proper error handling and fallback strategies are crucial for production use."

## Summary

**Module Federation enables:**
- **Dynamic code loading** across applications
- **Dependency sharing** to reduce bundle sizes
- **Independent deployment** of micro frontends
- **Runtime composition** of applications

**Key configuration:**
- **Host apps** consume remote modules
- **Remote apps** expose modules via remoteEntry.js
- **Shared dependencies** prevent duplication
- **Singleton mode** ensures single instances

**Benefits:**
- **Micro frontend architecture** made simple
- **Reduced bundle sizes** through code sharing
- **Independent deployments** and scaling
- **Technology flexibility** across teams

**Best practices:**
- **Version management** for shared dependencies
- **Error boundaries** for graceful failures
- **Lazy loading** for performance
- **Testing strategies** for federation reliability

**Interview Tip:** "Module Federation allows applications to expose and consume code at runtime, enabling micro frontend architectures. It creates a remoteEntry manifest for exposed modules and manages shared dependencies to prevent duplication, allowing teams to deploy independently while maintaining a cohesive user experience."