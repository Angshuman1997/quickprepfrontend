# Vite Advanced Configuration and Optimization

## Question
How do you configure Vite for optimal performance and production deployment in large-scale applications?

## Answer

Vite provides excellent developer experience with fast builds and hot reloading. For production applications, proper configuration is crucial for optimal performance, build optimization, and deployment strategies.

## Production-Ready Vite Configuration

### 1. **Comprehensive vite.config.ts Setup**

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';
import { visualizer } from 'rollup-plugin-visualizer';
import { createHtmlPlugin } from 'vite-plugin-html';
import legacy from '@vitejs/plugin-legacy';
import { VitePWA } from 'vite-plugin-pwa';
import { compression } from 'vite-plugin-compression2';

export default defineConfig(({ command, mode }) => {
    // Load environment variables
    const env = loadEnv(mode, process.cwd(), '');
    const isProduction = mode === 'production';
    const isDevelopment = mode === 'development';

    return {
        // Base public path
        base: env.VITE_BASE_PATH || '/',
        
        // Define global constants
        define: {
            __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
            __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
        },
        
        plugins: [
            // React support with Fast Refresh
            react({
                fastRefresh: isDevelopment,
                babel: {
                    plugins: [
                        isDevelopment && 'react-refresh/babel',
                        // Remove PropTypes in production
                        isProduction && 'babel-plugin-transform-react-remove-prop-types',
                    ].filter(Boolean),
                },
            }),
            
            // HTML processing
            createHtmlPlugin({
                minify: isProduction,
                inject: {
                    data: {
                        title: env.VITE_APP_TITLE || 'My App',
                        description: env.VITE_APP_DESCRIPTION || 'My amazing application',
                    },
                },
            }),
            
            // Legacy browser support
            legacy({
                targets: ['defaults', 'not IE 11'],
                additionalLegacyPolyfills: ['regenerator-runtime/runtime'],
                modernPolyfills: true,
            }),
            
            // PWA support
            VitePWA({
                registerType: 'autoUpdate',
                includeAssets: ['favicon.ico', 'apple-touch-icon.png', 'masked-icon.svg'],
                manifest: {
                    name: 'My Progressive Web App',
                    short_name: 'MyPWA',
                    description: 'My awesome Progressive Web App',
                    theme_color: '#ffffff',
                    background_color: '#ffffff',
                    display: 'standalone',
                    icons: [
                        {
                            src: 'pwa-192x192.png',
                            sizes: '192x192',
                            type: 'image/png',
                        },
                        {
                            src: 'pwa-512x512.png',
                            sizes: '512x512',
                            type: 'image/png',
                        },
                    ],
                },
                workbox: {
                    globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
                    runtimeCaching: [
                        {
                            urlPattern: /^https:\/\/api\.example\.com\/.*/i,
                            handler: 'StaleWhileRevalidate',
                            options: {
                                cacheName: 'api-cache',
                                expiration: {
                                    maxEntries: 100,
                                    maxAgeSeconds: 60 * 60 * 24, // 24 hours
                                },
                            },
                        },
                    ],
                },
            }),
            
            // Gzip compression
            compression({
                algorithm: 'gzip',
                exclude: [/\.(br)$ /, /\.(gz)$/],
            }),
            
            // Brotli compression
            compression({
                algorithm: 'brotliCompress',
                exclude: [/\.(br)$ /, /\.(gz)$/],
                compressionOptions: { level: 11 },
            }),
            
            // Bundle analyzer (only when needed)
            process.env.ANALYZE && visualizer({
                filename: 'dist/bundle-analysis.html',
                open: true,
                gzipSize: true,
                brotliSize: true,
            }),
        ].filter(Boolean),
        
        // Resolve configuration
        resolve: {
            alias: {
                '@': resolve(__dirname, 'src'),
                '@components': resolve(__dirname, 'src/components'),
                '@hooks': resolve(__dirname, 'src/hooks'),
                '@utils': resolve(__dirname, 'src/utils'),
                '@types': resolve(__dirname, 'src/types'),
                '@assets': resolve(__dirname, 'src/assets'),
                '@pages': resolve(__dirname, 'src/pages'),
                '@services': resolve(__dirname, 'src/services'),
                '@store': resolve(__dirname, 'src/store'),
            },
        },
        
        // CSS configuration
        css: {
            preprocessorOptions: {
                scss: {
                    additionalData: '@import "@/styles/variables.scss"; @import "@/styles/mixins.scss";',
                },
            },
            modules: {
                localsConvention: 'camelCaseOnly',
                generateScopedName: isProduction 
                    ? '[hash:base64:8]' 
                    : '[name]__[local]___[hash:base64:5]',
            },
            devSourcemap: isDevelopment,
        },
        
        // Build configuration
        build: {
            // Output directory
            outDir: 'dist',
            
            // Generate source maps
            sourcemap: isProduction ? 'hidden' : true,
            
            // Minification
            minify: 'terser',
            terserOptions: {
                compress: {
                    drop_console: isProduction,
                    drop_debugger: isProduction,
                },
            },
            
            // Chunk size warnings
            chunkSizeWarningLimit: 1000,
            
            // Rollup options
            rollupOptions: {
                input: {
                    main: resolve(__dirname, 'index.html'),
                },
                
                output: {
                    // Manual chunk splitting
                    manualChunks: {
                        // Vendor chunks
                        'react-vendor': ['react', 'react-dom'],
                        'router-vendor': ['react-router-dom'],
                        'ui-vendor': ['@mui/material', '@emotion/react', '@emotion/styled'],
                        'chart-vendor': ['chart.js', 'react-chartjs-2'],
                        'utils-vendor': ['lodash', 'date-fns'],
                        
                        // Feature-based chunks
                        'auth': ['./src/features/auth'],
                        'dashboard': ['./src/features/dashboard'],
                        'profile': ['./src/features/profile'],
                    },
                    
                    // Asset naming
                    assetFileNames: (assetInfo) => {
                        const info = assetInfo.name.split('.');
                        const ext = info[info.length - 1];
                        
                        if (/png|jpe?g|svg|gif|tiff|bmp|ico/i.test(ext)) {
                            return `images/[name]-[hash][extname]`;
                        }
                        if (/woff|woff2|eot|ttf|otf/i.test(ext)) {
                            return `fonts/[name]-[hash][extname]`;
                        }
                        return `assets/[name]-[hash][extname]`;
                    },
                    
                    chunkFileNames: 'js/[name]-[hash].js',
                    entryFileNames: 'js/[name]-[hash].js',
                },
                
                // External dependencies
                external: isDevelopment ? [] : [
                    // Externalize heavy libraries for CDN loading
                    // 'react',
                    // 'react-dom',
                ],
            },
        },
        
        // Development server
        server: {
            port: 3000,
            host: true,
            
            // API proxy
            proxy: {
                '/api': {
                    target: env.VITE_API_URL || 'http://localhost:8080',
                    changeOrigin: true,
                    rewrite: (path) => path.replace(/^\/api/, ''),
                },
                '/uploads': {
                    target: env.VITE_UPLOAD_URL || 'http://localhost:8080',
                    changeOrigin: true,
                },
            },
            
            // CORS
            cors: true,
            
            // HTTPS in development
            https: env.VITE_HTTPS === 'true',
        },
        
        // Preview server (for production builds)
        preview: {
            port: 4173,
            host: true,
        },
        
        // Optimizations
        optimizeDeps: {
            include: [
                'react',
                'react-dom',
                'react-router-dom',
                '@mui/material',
                '@emotion/react',
                '@emotion/styled',
            ],
            exclude: [
                // Exclude problematic dependencies
            ],
        },
        
        // Environment variables
        envPrefix: 'VITE_',
    };
});
```

### 2. **Advanced Plugin Configuration**

```typescript
// vite-plugins/custom-plugins.ts

// Custom plugin for environment-based optimization
function environmentOptimization(): Plugin {
    return {
        name: 'environment-optimization',
        configResolved(config) {
            if (config.command === 'build') {
                // Production optimizations
                console.log('🚀 Applying production optimizations...');
            }
        },
        generateBundle(options, bundle) {
            // Analyze bundle and optimize
            Object.keys(bundle).forEach(fileName => {
                const chunk = bundle[fileName];
                if (chunk.type === 'chunk') {
                    console.log(`📦 Generated chunk: ${fileName} (${chunk.code.length} bytes)`);
                }
            });
        },
    };
}

// Plugin for API mocking in development
function apiMocking(): Plugin {
    return {
        name: 'api-mocking',
        configureServer(server) {
            server.middlewares.use('/api/mock', (req, res, next) => {
                // Mock API responses in development
                if (req.url?.includes('/users')) {
                    res.setHeader('Content-Type', 'application/json');
                    res.end(JSON.stringify([
                        { id: 1, name: 'John Doe' },
                        { id: 2, name: 'Jane Smith' },
                    ]));
                } else {
                    next();
                }
            });
        },
    };
}

// Plugin for build-time optimizations
function buildTimeOptimizations(): Plugin {
    return {
        name: 'build-time-optimizations',
        transform(code, id) {
            // Remove development-only code in production
            if (process.env.NODE_ENV === 'production') {
                // Remove console.log statements
                code = code.replace(/console\.log\([^)]*\);?/g, '');
                
                // Remove development assertions
                code = code.replace(/if\s*\(\s*process\.env\.NODE_ENV\s*===\s*['"]development['"]\s*\)\s*{[^}]*}/g, '');
            }
            
            return { code, map: null };
        },
    };
}

// Dynamic import optimization
function dynamicImportOptimization(): Plugin {
    return {
        name: 'dynamic-import-optimization',
        generateBundle(options, bundle) {
            // Optimize dynamic imports
            Object.keys(bundle).forEach(fileName => {
                const chunk = bundle[fileName];
                if (chunk.type === 'chunk' && chunk.isDynamicEntry) {
                    // Add preload hints for dynamic chunks
                    chunk.code = `
                        // Preload hint for faster loading
                        const link = document.createElement('link');
                        link.rel = 'preload';
                        link.href = '${fileName}';
                        link.as = 'script';
                        document.head.appendChild(link);
                        
                        ${chunk.code}
                    `;
                }
            });
        },
    };
}

export {
    environmentOptimization,
    apiMocking,
    buildTimeOptimizations,
    dynamicImportOptimization,
};
```

### 3. **Performance Optimization Strategies**

```typescript
// vite-performance/optimization.ts

// Lazy loading configuration
export const lazyLoadingConfig = {
    // Route-based lazy loading
    routes: {
        home: () => import('../pages/Home'),
        dashboard: () => import('../pages/Dashboard'),
        profile: () => import('../pages/Profile'),
        settings: () => import('../pages/Settings'),
    },
    
    // Component-based lazy loading
    components: {
        DataTable: () => import('../components/DataTable'),
        Chart: () => import('../components/Chart'),
        Calendar: () => import('../components/Calendar'),
        FileUploader: () => import('../components/FileUploader'),
    },
    
    // Library-based lazy loading
    libraries: {
        charts: () => import('chart.js'),
        editor: () => import('@monaco-editor/react'),
        pdf: () => import('react-pdf'),
    },
};

// Preloading strategy
class PreloadManager {
    private preloadedModules = new Set<string>();

    preloadRoute(routeName: string): void {
        if (this.preloadedModules.has(routeName)) return;
        
        const routeLoader = lazyLoadingConfig.routes[routeName];
        if (routeLoader) {
            routeLoader().then(() => {
                this.preloadedModules.add(routeName);
                console.log(`🚀 Preloaded route: ${routeName}`);
            });
        }
    }

    preloadOnHover(routeName: string): void {
        // Preload when user hovers over navigation links
        setTimeout(() => this.preloadRoute(routeName), 100);
    }

    preloadOnIdle(): void {
        // Preload during idle time
        if ('requestIdleCallback' in window) {
            requestIdleCallback(() => {
                Object.keys(lazyLoadingConfig.routes).forEach(route => {
                    if (!this.preloadedModules.has(route)) {
                        this.preloadRoute(route);
                    }
                });
            });
        }
    }
}

// Image optimization
export const imageOptimization = {
    // Generate responsive images
    generateResponsiveImages: (src: string) => ({
        srcSet: [
            `${src}?w=320 320w`,
            `${src}?w=640 640w`,
            `${src}?w=1024 1024w`,
            `${src}?w=1920 1920w`,
        ].join(', '),
        sizes: '(max-width: 320px) 320px, (max-width: 640px) 640px, (max-width: 1024px) 1024px, 1920px',
    }),
    
    // WebP/AVIF format support
    modernFormats: (src: string) => ({
        webp: `${src}?format=webp`,
        avif: `${src}?format=avif`,
        fallback: src,
    }),
};

// Bundle splitting strategies
export const bundleSplitting = {
    // Vendor splitting by usage frequency
    highFrequency: ['react', 'react-dom', 'react-router-dom'],
    mediumFrequency: ['@mui/material', '@emotion/react'],
    lowFrequency: ['chart.js', 'pdf-lib'],
    
    // Feature-based splitting
    features: {
        auth: [/src\/features\/auth/],
        dashboard: [/src\/features\/dashboard/],
        admin: [/src\/features\/admin/],
    },
    
    // Utility splitting
    utilities: {
        lodash: ['lodash'],
        dateFns: ['date-fns'],
        validation: ['yup', 'joi'],
    },
};
```

### 4. **Development Experience Enhancements**

```typescript
// vite-dev/dev-enhancements.ts

// Hot Module Replacement customization
export function setupHMR() {
    if (import.meta.hot) {
        // Accept hot updates for specific modules
        import.meta.hot.accept('./App', (newModule) => {
            console.log('🔥 HMR: App module updated');
            // Custom update logic
        });
        
        // Custom HMR for state management
        import.meta.hot.accept('./store', (newStore) => {
            console.log('🔥 HMR: Store updated');
            // Preserve store state during HMR
            const currentState = store.getState();
            store.replaceReducer(newStore.rootReducer);
            store.dispatch({ type: 'REHYDRATE', payload: currentState });
        });
        
        // CSS HMR
        import.meta.hot.accept('./styles/main.css', () => {
            console.log('🎨 HMR: CSS updated');
        });
    }
}

// Development tools integration
export function setupDevTools() {
    if (import.meta.env.DEV) {
        // Performance monitoring in development
        const observer = new PerformanceObserver((list) => {
            list.getEntries().forEach((entry) => {
                console.log(`⏱️ ${entry.name}: ${entry.duration.toFixed(2)}ms`);
            });
        });
        
        observer.observe({ entryTypes: ['measure', 'navigation'] });
        
        // Memory usage monitoring
        setInterval(() => {
            if ('memory' in performance) {
                const memory = (performance as any).memory;
                console.log(`🧠 Memory: ${(memory.usedJSHeapSize / 1024 / 1024).toFixed(2)}MB`);
            }
        }, 10000);
        
        // Bundle size warnings
        const bundleSizeLimit = 2 * 1024 * 1024; // 2MB
        if (window.performance) {
            window.addEventListener('load', () => {
                const resources = performance.getEntriesByType('resource');
                resources.forEach((resource: any) => {
                    if (resource.name.includes('.js') && resource.transferSize > bundleSizeLimit) {
                        console.warn(`⚠️ Large bundle detected: ${resource.name} (${(resource.transferSize / 1024 / 1024).toFixed(2)}MB)`);
                    }
                });
            });
        }
    }
}

// API mocking with MSW integration
export async function setupMSW() {
    if (import.meta.env.DEV && import.meta.env.VITE_ENABLE_MSW === 'true') {
        const { worker } = await import('./mocks/browser');
        await worker.start({
            onUnhandledRequest: 'bypass',
        });
        console.log('🔧 MSW: API mocking enabled');
    }
}
```

### 5. **Production Deployment Configuration**

```typescript
// vite-deploy/deployment.ts

// Multi-environment configuration
export const deploymentConfig = {
    development: {
        base: '/',
        apiUrl: 'http://localhost:8080',
        enableDevTools: true,
        enableMocking: true,
    },
    
    staging: {
        base: '/staging/',
        apiUrl: 'https://staging-api.example.com',
        enableDevTools: true,
        enableMocking: false,
    },
    
    production: {
        base: '/',
        apiUrl: 'https://api.example.com',
        enableDevTools: false,
        enableMocking: false,
    },
};

// Build optimization for different environments
export function getBuildConfig(environment: string) {
    const config = deploymentConfig[environment];
    
    return {
        build: {
            sourcemap: environment !== 'production',
            minify: environment === 'production' ? 'terser' : false,
            
            rollupOptions: {
                output: {
                    // Different chunking strategies per environment
                    manualChunks: environment === 'production' 
                        ? productionChunks
                        : developmentChunks,
                },
            },
            
            // Environment-specific optimizations
            terserOptions: environment === 'production' ? {
                compress: {
                    drop_console: true,
                    drop_debugger: true,
                    pure_funcs: ['console.log', 'console.info'],
                },
            } : {},
        },
    };
}

// CDN integration
export function setupCDN(environment: string) {
    if (environment === 'production') {
        return {
            base: 'https://cdn.example.com/app/',
            
            build: {
                rollupOptions: {
                    external: ['react', 'react-dom'],
                    output: {
                        globals: {
                            react: 'React',
                            'react-dom': 'ReactDOM',
                        },
                    },
                },
            },
        };
    }
    
    return {};
}

// Performance budgets
export const performanceBudgets = {
    maxBundleSize: 2 * 1024 * 1024, // 2MB
    maxChunkSize: 500 * 1024, // 500KB
    maxAssetSize: 1 * 1024 * 1024, // 1MB
};

// Build analysis
export function analyzeBuild() {
    return {
        plugins: [
            {
                name: 'build-analyzer',
                generateBundle(options, bundle) {
                    const analysis = {
                        totalSize: 0,
                        chunks: [],
                        assets: [],
                    };
                    
                    Object.entries(bundle).forEach(([fileName, chunk]) => {
                        if (chunk.type === 'chunk') {
                            const size = chunk.code.length;
                            analysis.totalSize += size;
                            analysis.chunks.push({ fileName, size });
                            
                            // Check against performance budgets
                            if (size > performanceBudgets.maxChunkSize) {
                                console.warn(`⚠️ Chunk ${fileName} exceeds size budget: ${(size / 1024).toFixed(2)}KB`);
                            }
                        } else if (chunk.type === 'asset') {
                            const size = chunk.source.length;
                            analysis.assets.push({ fileName, size });
                            
                            if (size > performanceBudgets.maxAssetSize) {
                                console.warn(`⚠️ Asset ${fileName} exceeds size budget: ${(size / 1024).toFixed(2)}KB`);
                            }
                        }
                    });
                    
                    // Generate build report
                    console.log('\n📊 Build Analysis Report:');
                    console.log(`Total Bundle Size: ${(analysis.totalSize / 1024 / 1024).toFixed(2)}MB`);
                    console.log(`Chunks: ${analysis.chunks.length}`);
                    console.log(`Assets: ${analysis.assets.length}`);
                    
                    // Save detailed report
                    this.emitFile({
                        type: 'asset',
                        fileName: 'build-report.json',
                        source: JSON.stringify(analysis, null, 2),
                    });
                },
            },
        ],
    };
}
```

### 6. **Testing Integration**

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
    plugins: [react()],
    
    test: {
        globals: true,
        environment: 'jsdom',
        setupFiles: ['./src/test/setup.ts'],
        
        // Coverage configuration
        coverage: {
            provider: 'v8',
            reporter: ['text', 'json', 'html'],
            exclude: [
                'node_modules/',
                'src/test/',
                '**/*.d.ts',
                '**/*.config.*',
            ],
            thresholds: {
                global: {
                    branches: 80,
                    functions: 80,
                    lines: 80,
                    statements: 80,
                },
            },
        },
        
        // Mock configuration
        deps: {
            inline: ['@testing-library/jest-dom'],
        },
        
        // Performance configuration
        pool: 'threads',
        poolOptions: {
            threads: {
                singleThread: false,
            },
        },
    },
    
    resolve: {
        alias: {
            '@': resolve(__dirname, 'src'),
            '@components': resolve(__dirname, 'src/components'),
            '@utils': resolve(__dirname, 'src/utils'),
        },
    },
});
```

## Interview Questions & Answers

### Q: What are the main advantages of Vite over traditional bundlers like Webpack?

**A:** Vite offers faster development builds using esbuild, native ES modules in development, instant hot module replacement, better tree shaking with Rollup, simpler configuration, and built-in TypeScript support. It leverages browser native ES modules during development and uses Rollup for optimized production builds.

### Q: How do you optimize Vite builds for production?

**A:** I configure manual chunk splitting for optimal caching, enable tree shaking and dead code elimination, use Terser for minification, implement asset compression with gzip/brotli, optimize images and fonts, configure proper source maps, and use bundle analysis tools to monitor size and performance.

### Q: Explain your approach to code splitting in Vite.

**A:** I use dynamic imports for route-based splitting, implement lazy loading for heavy components, configure manual chunks for vendor libraries based on update frequency, use Rollup's optimization features, implement preloading strategies for critical resources, and monitor chunk sizes to maintain optimal loading performance.

### Q: How do you handle environment-specific configurations in Vite?

**A:** I use environment variables with VITE_ prefix, create mode-specific config files, implement conditional plugin loading based on build mode, configure different optimization levels per environment, set up environment-specific proxy rules, and use build-time constants for feature flags.

### Interview Tip:
*"Vite's strength lies in its development experience and modern approach to bundling. Focus on leveraging its native ES modules support during development while ensuring proper optimization for production builds with Rollup's advanced features."*