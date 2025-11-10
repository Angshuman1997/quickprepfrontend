# Advanced Webpack Configuration Patterns

## Question
How do you configure Webpack for a production-ready application with optimal performance and developer experience?

## Answer

Modern Webpack configuration requires balancing build performance, bundle optimization, and developer experience. Here's a comprehensive guide for production-ready setups.

## Production-Optimized Configuration

### 1. **Base Configuration Structure**

```javascript
// webpack.config.js
const path = require('path');
const webpack = require('webpack');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const CompressionPlugin = require('compression-webpack-plugin');

const isDevelopment = process.env.NODE_ENV === 'development';
const isProduction = process.env.NODE_ENV === 'production';

module.exports = {
    mode: isProduction ? 'production' : 'development',
    
    // Entry points with dynamic imports for code splitting
    entry: {
        main: './src/index.tsx',
        vendor: ['react', 'react-dom'],
    },
    
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: isProduction 
            ? 'js/[name].[contenthash:8].js'
            : 'js/[name].js',
        chunkFilename: isProduction
            ? 'js/[name].[contenthash:8].chunk.js'
            : 'js/[name].chunk.js',
        publicPath: '/',
        clean: true, // Clean dist folder before each build
        assetModuleFilename: 'assets/[name].[contenthash:8][ext]',
    },
    
    // Resolve configuration
    resolve: {
        extensions: ['.tsx', '.ts', '.jsx', '.js', '.json'],
        alias: {
            '@': path.resolve(__dirname, 'src'),
            '@components': path.resolve(__dirname, 'src/components'),
            '@utils': path.resolve(__dirname, 'src/utils'),
            '@hooks': path.resolve(__dirname, 'src/hooks'),
            '@types': path.resolve(__dirname, 'src/types'),
        },
        // Fallbacks for Node.js modules in browser
        fallback: {
            buffer: require.resolve('buffer'),
            process: require.resolve('process/browser'),
        },
    },
    
    module: {
        rules: [
            // TypeScript/JavaScript
            {
                test: /\.(ts|tsx|js|jsx)$/,
                exclude: /node_modules/,
                use: [
                    // Thread loader for parallel processing
                    {
                        loader: 'thread-loader',
                        options: {
                            workers: require('os').cpus().length - 1,
                        },
                    },
                    {
                        loader: 'babel-loader',
                        options: {
                            presets: [
                                ['@babel/preset-env', {
                                    useBuiltIns: 'entry',
                                    corejs: 3,
                                    targets: {
                                        browsers: ['> 1%', 'last 2 versions'],
                                    },
                                }],
                                '@babel/preset-react',
                                '@babel/preset-typescript',
                            ],
                            plugins: [
                                '@babel/plugin-proposal-class-properties',
                                '@babel/plugin-syntax-dynamic-import',
                                isProduction && ['babel-plugin-transform-remove-console', {
                                    exclude: ['error', 'warn'],
                                }],
                            ].filter(Boolean),
                            cacheDirectory: true,
                        },
                    },
                ],
            },
            
            // CSS/SCSS
            {
                test: /\.(css|scss|sass)$/,
                use: [
                    isProduction ? MiniCssExtractPlugin.loader : 'style-loader',
                    {
                        loader: 'css-loader',
                        options: {
                            modules: {
                                auto: true, // Enable CSS modules for .module.css files
                                localIdentName: isProduction
                                    ? '[hash:base64:8]'
                                    : '[name]__[local]__[hash:base64:5]',
                            },
                            importLoaders: 2,
                        },
                    },
                    {
                        loader: 'postcss-loader',
                        options: {
                            postcssOptions: {
                                plugins: [
                                    ['autoprefixer'],
                                    ['cssnano', { preset: 'default' }],
                                ],
                            },
                        },
                    },
                    'sass-loader',
                ],
            },
            
            // Images
            {
                test: /\.(png|jpg|jpeg|gif|webp|avif)$/i,
                type: 'asset',
                generator: {
                    filename: 'images/[name].[contenthash:8][ext]',
                },
                parser: {
                    dataUrlCondition: {
                        maxSize: 8 * 1024, // 8kb
                    },
                },
            },
            
            // SVG with SVGR
            {
                test: /\.svg$/,
                use: [
                    {
                        loader: '@svgr/webpack',
                        options: {
                            typescript: true,
                            ext: 'tsx',
                        },
                    },
                    'url-loader',
                ],
            },
            
            // Fonts
            {
                test: /\.(woff|woff2|eot|ttf|otf)$/i,
                type: 'asset/resource',
                generator: {
                    filename: 'fonts/[name].[contenthash:8][ext]',
                },
            },
        ],
    },
    
    plugins: [
        // HTML template
        new HtmlWebpackPlugin({
            template: './public/index.html',
            minify: isProduction ? {
                removeComments: true,
                collapseWhitespace: true,
                removeRedundantAttributes: true,
                useShortDoctype: true,
                removeEmptyAttributes: true,
                removeStyleLinkTypeAttributes: true,
                keepClosingSlash: true,
                minifyJS: true,
                minifyCSS: true,
                minifyURLs: true,
            } : false,
        }),
        
        // CSS extraction for production
        isProduction && new MiniCssExtractPlugin({
            filename: 'css/[name].[contenthash:8].css',
            chunkFilename: 'css/[name].[contenthash:8].chunk.css',
        }),
        
        // Bundle analysis
        process.env.ANALYZE && new BundleAnalyzerPlugin(),
        
        // Compression
        isProduction && new CompressionPlugin({
            algorithm: 'gzip',
            test: /\.(js|css|html|svg)$/,
            threshold: 8192,
            minRatio: 0.8,
        }),
        
        // Brotli compression
        isProduction && new CompressionPlugin({
            filename: '[path][base].br',
            algorithm: 'brotliCompress',
            test: /\.(js|css|html|svg)$/,
            compressionOptions: { level: 11 },
            threshold: 8192,
            minRatio: 0.8,
        }),
        
        // Environment variables
        new webpack.DefinePlugin({
            'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
            'process.env.REACT_APP_API_URL': JSON.stringify(process.env.REACT_APP_API_URL),
        }),
        
        // Polyfills
        new webpack.ProvidePlugin({
            Buffer: ['buffer', 'Buffer'],
            process: 'process/browser',
        }),
    ].filter(Boolean),
    
    // Optimization
    optimization: {
        minimize: isProduction,
        minimizer: [
            new TerserPlugin({
                terserOptions: {
                    parse: { ecma: 8 },
                    compress: {
                        ecma: 5,
                        warnings: false,
                        comparisons: false,
                        inline: 2,
                        drop_console: true,
                    },
                    mangle: { safari10: true },
                    output: {
                        ecma: 5,
                        comments: false,
                        ascii_only: true,
                    },
                },
                parallel: true,
            }),
            new CssMinimizerPlugin(),
        ],
        
        // Code splitting configuration
        splitChunks: {
            chunks: 'all',
            cacheGroups: {
                // Vendor libraries
                vendor: {
                    test: /[\\/]node_modules[\\/]/,
                    name: 'vendors',
                    priority: 10,
                    enforce: true,
                },
                
                // React and related libraries
                react: {
                    test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
                    name: 'react',
                    priority: 20,
                    enforce: true,
                },
                
                // UI library (e.g., Ant Design, Material-UI)
                ui: {
                    test: /[\\/]node_modules[\\/](@ant-design|@material-ui|antd)[\\/]/,
                    name: 'ui-library',
                    priority: 15,
                    enforce: true,
                },
                
                // Utilities
                utils: {
                    test: /[\\/]node_modules[\\/](lodash|moment|date-fns)[\\/]/,
                    name: 'utils',
                    priority: 15,
                    enforce: true,
                },
                
                // Common chunks
                common: {
                    minChunks: 2,
                    priority: 5,
                    reuseExistingChunk: true,
                },
            },
        },
        
        // Runtime chunk
        runtimeChunk: {
            name: 'runtime',
        },
    },
    
    // Development server
    devServer: isDevelopment ? {
        port: 3000,
        hot: true,
        compress: true,
        historyApiFallback: true,
        static: {
            directory: path.join(__dirname, 'public'),
        },
        client: {
            overlay: {
                errors: true,
                warnings: false,
            },
        },
    } : undefined,
    
    // Source maps
    devtool: isDevelopment 
        ? 'eval-cheap-module-source-map' 
        : 'source-map',
    
    // Performance
    performance: {
        maxEntrypointSize: 250000,
        maxAssetSize: 250000,
        hints: isProduction ? 'warning' : false,
    },
};
```

### 2. **Advanced Code Splitting Strategies**

```javascript
// Dynamic import configurations
// webpack.dynamic-imports.config.js

// Route-based code splitting
const routes = [
    { path: '/', component: () => import('./pages/Home') },
    { path: '/products', component: () => import('./pages/Products') },
    { path: '/checkout', component: () => import('./pages/Checkout') },
];

// Component-based code splitting
const LazyComponents = {
    HeavyChart: React.lazy(() => 
        import(/* webpackChunkName: "heavy-chart" */ './components/HeavyChart')
    ),
    VideoPlayer: React.lazy(() =>
        import(/* webpackChunkName: "video-player" */ './components/VideoPlayer')
    ),
    RichTextEditor: React.lazy(() =>
        import(/* webpackChunkName: "rich-editor" */ './components/RichTextEditor')
    ),
};

// Library code splitting with webpack magic comments
const loadLibrary = async (libraryName: string) => {
    switch (libraryName) {
        case 'charts':
            return import(
                /* webpackChunkName: "charts" */
                /* webpackPrefetch: true */
                'chart.js'
            );
        case 'pdf':
            return import(
                /* webpackChunkName: "pdf-lib" */
                /* webpackPreload: true */
                'pdf-lib'
            );
        default:
            throw new Error(`Unknown library: ${libraryName}`);
    }
};

// Progressive loading with priorities
class ProgressiveLoader {
    private loadQueue: Array<() => Promise<any>> = [];
    private isLoading = false;

    enqueue(loader: () => Promise<any>, priority: 'high' | 'medium' | 'low' = 'medium'): void {
        if (priority === 'high') {
            this.loadQueue.unshift(loader);
        } else {
            this.loadQueue.push(loader);
        }
        
        if (!this.isLoading) {
            this.processQueue();
        }
    }

    private async processQueue(): Promise<void> {
        this.isLoading = true;

        while (this.loadQueue.length > 0) {
            const loader = this.loadQueue.shift()!;
            
            try {
                await loader();
            } catch (error) {
                console.error('Progressive loading failed:', error);
            }
            
            // Yield to browser between loads
            await new Promise(resolve => setTimeout(resolve, 0));
        }

        this.isLoading = false;
    }
}

// Usage in React components
function useProgressiveLoader() {
    const loader = useRef(new ProgressiveLoader());

    const loadComponent = useCallback((componentName: string, priority?: 'high' | 'medium' | 'low') => {
        loader.current.enqueue(
            () => import(`../components/${componentName}`),
            priority
        );
    }, []);

    return { loadComponent };
}
```

### 3. **Performance Optimization Techniques**

```javascript
// webpack.performance.config.js

const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();

const performanceConfig = smp.wrap({
    // Cache configuration for faster builds
    cache: {
        type: 'filesystem',
        buildDependencies: {
            config: [__filename],
        },
        cacheDirectory: path.resolve(__dirname, '.webpack-cache'),
    },
    
    // Resolve performance optimizations
    resolve: {
        // Reduce resolve attempts
        modules: [
            path.resolve(__dirname, 'src'),
            'node_modules',
        ],
        
        // Alias for faster resolution
        alias: {
            'react': path.resolve(__dirname, 'node_modules/react'),
            'react-dom': path.resolve(__dirname, 'node_modules/react-dom'),
        },
        
        // Symlinks can slow down resolution
        symlinks: false,
    },
    
    module: {
        rules: [
            {
                test: /\.(ts|tsx)$/,
                exclude: /node_modules/,
                use: [
                    // Parallel processing with thread-loader
                    {
                        loader: 'thread-loader',
                        options: {
                            poolTimeout: 2000,
                            workers: require('os').cpus().length - 1,
                        },
                    },
                    // Fork TypeScript checker to separate process
                    {
                        loader: 'ts-loader',
                        options: {
                            happyPackMode: true,
                            transpileOnly: true,
                        },
                    },
                ],
            },
        ],
    },
    
    plugins: [
        // TypeScript type checking in separate process
        new ForkTsCheckerWebpackPlugin({
            typescript: {
                diagnosticOptions: {
                    semantic: true,
                    syntactic: true,
                },
            },
        }),
        
        // Progress reporting
        new webpack.ProgressPlugin((percentage, message, ...args) => {
            console.info(`${(percentage * 100).toFixed(2)}%`, message, ...args);
        }),
    ],
    
    // Optimization for faster builds
    optimization: {
        // Remove unused exports
        usedExports: true,
        
        // Enable tree shaking
        sideEffects: false,
        
        // Concatenate modules for better minification
        concatenateModules: true,
        
        // Split chunks efficiently
        splitChunks: {
            chunks: 'all',
            minSize: 20000,
            maxSize: 240000,
            cacheGroups: {
                vendor: {
                    test: /[\\/]node_modules[\\/]/,
                    name: 'vendors',
                    chunks: 'all',
                },
            },
        },
    },
    
    // Ignore parsing for known libraries without dependencies
    module: {
        noParse: /jquery|lodash/,
    },
    
    // External dependencies (loaded via CDN)
    externals: {
        'react': 'React',
        'react-dom': 'ReactDOM',
        'moment': 'moment',
    },
});

module.exports = performanceConfig;
```

### 4. **Module Federation Configuration**

```javascript
// webpack.federation.config.js
const ModuleFederationPlugin = require('@module-federation/webpack');

// Host application configuration
const hostConfig = {
    plugins: [
        new ModuleFederationPlugin({
            name: 'host',
            remotes: {
                'mf-header': 'header@http://localhost:3001/remoteEntry.js',
                'mf-footer': 'footer@http://localhost:3002/remoteEntry.js',
                'mf-dashboard': 'dashboard@http://localhost:3003/remoteEntry.js',
            },
            shared: {
                react: {
                    singleton: true,
                    requiredVersion: '^18.0.0',
                    eager: true,
                },
                'react-dom': {
                    singleton: true,
                    requiredVersion: '^18.0.0',
                    eager: true,
                },
                '@company/design-system': {
                    singleton: true,
                    requiredVersion: '^1.0.0',
                },
            },
        }),
    ],
};

// Remote application configuration
const remoteConfig = {
    plugins: [
        new ModuleFederationPlugin({
            name: 'dashboard',
            filename: 'remoteEntry.js',
            exposes: {
                './Dashboard': './src/Dashboard',
                './Charts': './src/components/Charts',
                './UserProfile': './src/components/UserProfile',
            },
            shared: {
                react: {
                    singleton: true,
                    requiredVersion: '^18.0.0',
                },
                'react-dom': {
                    singleton: true, 
                    requiredVersion: '^18.0.0',
                },
            },
        }),
    ],
};

// Dynamic remote loading
class RemoteComponentLoader {
    private loadedRemotes = new Map<string, any>();

    async loadRemoteComponent(remoteName: string, componentName: string): Promise<React.ComponentType> {
        const remoteKey = `${remoteName}/${componentName}`;
        
        if (this.loadedRemotes.has(remoteKey)) {
            return this.loadedRemotes.get(remoteKey);
        }

        try {
            // @ts-ignore
            const container = window[remoteName];
            await __webpack_init_sharing__('default');
            await container.init(__webpack_share_scopes__.default);
            
            const factory = await container.get(componentName);
            const Component = factory().default;
            
            this.loadedRemotes.set(remoteKey, Component);
            return Component;
        } catch (error) {
            console.error(`Failed to load remote component ${remoteKey}:`, error);
            throw error;
        }
    }
}

// React hook for remote components
function useRemoteComponent(remoteName: string, componentName: string) {
    const [Component, setComponent] = useState<React.ComponentType | null>(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<Error | null>(null);
    
    useEffect(() => {
        const loader = new RemoteComponentLoader();
        
        loader.loadRemoteComponent(remoteName, componentName)
            .then(setComponent)
            .catch(setError)
            .finally(() => setLoading(false));
    }, [remoteName, componentName]);
    
    return { Component, loading, error };
}
```

### 5. **Environment-Specific Configurations**

```javascript
// webpack.config.factory.js
const createWebpackConfig = (env, argv) => {
    const isDevelopment = env.NODE_ENV === 'development';
    const isProduction = env.NODE_ENV === 'production';
    const isAnalyze = env.ANALYZE === 'true';
    
    const baseConfig = {
        // Base configuration
    };
    
    if (isDevelopment) {
        return {
            ...baseConfig,
            mode: 'development',
            devtool: 'eval-cheap-module-source-map',
            
            devServer: {
                port: 3000,
                hot: true,
                historyApiFallback: true,
                
                // API proxy
                proxy: {
                    '/api': {
                        target: 'http://localhost:8080',
                        changeOrigin: true,
                        pathRewrite: {
                            '^/api': '',
                        },
                    },
                },
                
                // HTTPS in development
                https: process.env.HTTPS === 'true',
                
                // Custom headers
                headers: {
                    'Access-Control-Allow-Origin': '*',
                },
            },
            
            plugins: [
                ...baseConfig.plugins,
                new webpack.HotModuleReplacementPlugin(),
            ],
        };
    }
    
    if (isProduction) {
        return {
            ...baseConfig,
            mode: 'production',
            devtool: 'source-map',
            
            plugins: [
                ...baseConfig.plugins,
                
                // Bundle analysis
                isAnalyze && new BundleAnalyzerPlugin({
                    analyzerMode: 'static',
                    openAnalyzer: false,
                }),
                
                // Workbox for PWA
                new WorkboxPlugin.GenerateSW({
                    clientsClaim: true,
                    skipWaiting: true,
                    maximumFileSizeToCacheInBytes: 5000000,
                    runtimeCaching: [{
                        urlPattern: /^https:\/\/api\.example\.com/,
                        handler: 'StaleWhileRevalidate',
                        options: {
                            cacheName: 'api-cache',
                            expiration: {
                                maxEntries: 100,
                                maxAgeSeconds: 60 * 60 * 24, // 24 hours
                            },
                        },
                    }],
                }),
            ].filter(Boolean),
            
            optimization: {
                ...baseConfig.optimization,
                
                // Enable all optimizations
                minimize: true,
                sideEffects: false,
                usedExports: true,
                
                // Advanced minification
                minimizer: [
                    new TerserPlugin({
                        parallel: true,
                        terserOptions: {
                            compress: {
                                drop_console: true,
                                drop_debugger: true,
                            },
                        },
                    }),
                    new CssMinimizerPlugin(),
                ],
            },
        };
    }
    
    return baseConfig;
};

module.exports = createWebpackConfig;
```

### 6. **Build Performance Monitoring**

```javascript
// webpack.monitor.js
class WebpackBuildMonitor {
    constructor() {
        this.startTime = Date.now();
        this.stats = {
            buildTime: 0,
            bundleSize: 0,
            chunkCount: 0,
            assetCount: 0,
        };
    }

    onBuildStart() {
        console.log('🚀 Build started...');
        this.startTime = Date.now();
    }

    onBuildEnd(stats) {
        this.stats.buildTime = Date.now() - this.startTime;
        this.stats.bundleSize = this.calculateBundleSize(stats);
        this.stats.chunkCount = stats.compilation.chunks.size;
        this.stats.assetCount = Object.keys(stats.compilation.assets).length;

        this.reportStats();
        this.checkPerformanceThresholds();
    }

    calculateBundleSize(stats) {
        return Object.values(stats.compilation.assets)
            .reduce((total, asset) => total + asset.size(), 0);
    }

    reportStats() {
        console.log('\n📊 Build Statistics:');
        console.log(`⏱️  Build Time: ${this.stats.buildTime}ms`);
        console.log(`📦 Bundle Size: ${(this.stats.bundleSize / 1024 / 1024).toFixed(2)}MB`);
        console.log(`🧩 Chunks: ${this.stats.chunkCount}`);
        console.log(`📁 Assets: ${this.stats.assetCount}`);
    }

    checkPerformanceThresholds() {
        const thresholds = {
            buildTime: 30000, // 30 seconds
            bundleSize: 5 * 1024 * 1024, // 5MB
        };

        if (this.stats.buildTime > thresholds.buildTime) {
            console.warn(`⚠️  Build time (${this.stats.buildTime}ms) exceeds threshold`);
        }

        if (this.stats.bundleSize > thresholds.bundleSize) {
            console.warn(`⚠️  Bundle size exceeds threshold`);
        }
    }
}

// Plugin integration
class BuildMonitorPlugin {
    apply(compiler) {
        const monitor = new WebpackBuildMonitor();

        compiler.hooks.run.tap('BuildMonitorPlugin', () => {
            monitor.onBuildStart();
        });

        compiler.hooks.done.tap('BuildMonitorPlugin', (stats) => {
            monitor.onBuildEnd(stats);
        });
    }
}

module.exports = { BuildMonitorPlugin };
```

## Interview Questions & Answers

### Q: How do you optimize Webpack build performance for large applications?

**A:** I use several strategies: enable filesystem caching, use thread-loader for parallel processing, implement proper code splitting with splitChunks optimization, use externals for large libraries, enable tree shaking with sideEffects: false, and use tools like ForkTsCheckerWebpackPlugin to run TypeScript checking in parallel.

### Q: Explain your approach to code splitting in Webpack.

**A:** I implement multi-level code splitting: route-based splitting for different pages, vendor splitting for third-party libraries, common chunks for shared code, and dynamic imports for heavy components. I use webpack magic comments for chunk naming and prefetching, and implement progressive loading based on user interactions.

### Q: How do you configure Webpack for a micro-frontend architecture?

**A:** I use Module Federation with shared dependencies configuration, implement dynamic remote loading with fallbacks, set up proper error boundaries for remote failures, configure shared design systems as singletons, and implement health checks for remote applications to ensure reliability.

### Q: What's your strategy for bundle size optimization?

**A:** I analyze bundles with webpack-bundle-analyzer, implement aggressive code splitting, use tree shaking and dead code elimination, compress assets with gzip/brotli, optimize images with appropriate formats, implement lazy loading for non-critical resources, and use externals for large libraries that can be loaded from CDN.

### Interview Tip:
*"Modern Webpack configuration is about finding the right balance between build performance, bundle optimization, and developer experience. Focus on understanding the impact of each optimization and measure the results to make informed decisions."*