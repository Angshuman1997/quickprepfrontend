# Authentication and Authorization Patterns

## Question
How do you implement secure authentication and authorization in modern frontend applications?

## Answer

Frontend authentication and authorization require careful balance between security, user experience, and system complexity. Here's a comprehensive approach covering JWT tokens, OAuth flows, session management, and role-based access control with security best practices.

## Authentication Strategies and Implementation

### 1. **JWT Token Management**

```typescript
// Comprehensive JWT token management system
class TokenManager {
    private accessToken: string | null = null;
    private refreshToken: string | null = null;
    private tokenRefreshPromise: Promise<string> | null = null;
    private refreshTimer: NodeJS.Timeout | null = null;

    constructor(private config: TokenManagerConfig) {
        this.loadTokensFromStorage();
        this.setupTokenRefreshScheduler();
    }

    // Get valid access token with automatic refresh
    async getValidAccessToken(): Promise<string | null> {
        if (!this.accessToken) {
            return null;
        }

        // Check if token is still valid
        if (this.isTokenValid(this.accessToken)) {
            return this.accessToken;
        }

        // Token is expired, attempt refresh
        try {
            return await this.refreshAccessToken();
        } catch (error) {
            console.error('Token refresh failed:', error);
            this.clearTokens();
            return null;
        }
    }

    // Refresh access token using refresh token
    private async refreshAccessToken(): Promise<string> {
        // Prevent multiple concurrent refresh attempts
        if (this.tokenRefreshPromise) {
            return this.tokenRefreshPromise;
        }

        this.tokenRefreshPromise = this.performTokenRefresh();
        
        try {
            const newAccessToken = await this.tokenRefreshPromise;
            this.tokenRefreshPromise = null;
            return newAccessToken;
        } catch (error) {
            this.tokenRefreshPromise = null;
            throw error;
        }
    }

    private async performTokenRefresh(): Promise<string> {
        if (!this.refreshToken) {
            throw new Error('No refresh token available');
        }

        const response = await fetch('/api/auth/refresh', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                refreshToken: this.refreshToken,
            }),
        });

        if (!response.ok) {
            throw new Error('Token refresh failed');
        }

        const tokenData = await response.json();
        this.setTokens(tokenData.accessToken, tokenData.refreshToken);
        
        return tokenData.accessToken;
    }

    // Set new tokens and persist
    setTokens(accessToken: string, refreshToken: string): void {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
        
        // Store tokens securely
        this.storeTokensSecurely(accessToken, refreshToken);
        
        // Schedule token refresh
        this.scheduleTokenRefresh();
    }

    // Clear all tokens
    clearTokens(): void {
        this.accessToken = null;
        this.refreshToken = null;
        
        this.clearTokenStorage();
        this.clearRefreshTimer();
    }

    // Check if token is valid (not expired)
    private isTokenValid(token: string): boolean {
        try {
            const payload = this.decodeJWTPayload(token);
            const currentTime = Math.floor(Date.now() / 1000);
            
            // Check expiration with 5-minute buffer
            return payload.exp > currentTime + 300;
        } catch (error) {
            return false;
        }
    }

    private decodeJWTPayload(token: string): any {
        const base64Url = token.split('.')[1];
        const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
        const jsonPayload = decodeURIComponent(
            window.atob(base64)
                .split('')
                .map(c => '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2))
                .join('')
        );
        return JSON.parse(jsonPayload);
    }

    // Secure token storage
    private storeTokensSecurely(accessToken: string, refreshToken: string): void {
        try {
            // Store access token in memory or sessionStorage (short-lived)
            sessionStorage.setItem('accessToken', accessToken);
            
            // Store refresh token in httpOnly cookie or secure localStorage
            if (this.config.useHttpOnlyCookies) {
                // Backend should set httpOnly cookie
                document.cookie = `refreshToken=${refreshToken}; Secure; SameSite=Strict; Path=/`;
            } else {
                // Encrypt before storing in localStorage
                const encryptedRefreshToken = this.encryptToken(refreshToken);
                localStorage.setItem('refreshToken', encryptedRefreshToken);
            }
        } catch (error) {
            console.error('Failed to store tokens:', error);
        }
    }

    private loadTokensFromStorage(): void {
        try {
            this.accessToken = sessionStorage.getItem('accessToken');
            
            if (this.config.useHttpOnlyCookies) {
                // Refresh token will be sent automatically with requests
                this.refreshToken = this.getCookieValue('refreshToken');
            } else {
                const encryptedRefreshToken = localStorage.getItem('refreshToken');
                if (encryptedRefreshToken) {
                    this.refreshToken = this.decryptToken(encryptedRefreshToken);
                }
            }
        } catch (error) {
            console.error('Failed to load tokens:', error);
        }
    }

    private scheduleTokenRefresh(): void {
        this.clearRefreshTimer();
        
        if (!this.accessToken) return;

        try {
            const payload = this.decodeJWTPayload(this.accessToken);
            const currentTime = Math.floor(Date.now() / 1000);
            const timeUntilRefresh = (payload.exp - currentTime - 600) * 1000; // 10 minutes before expiry

            if (timeUntilRefresh > 0) {
                this.refreshTimer = setTimeout(() => {
                    this.refreshAccessToken().catch(error => {
                        console.error('Scheduled token refresh failed:', error);
                    });
                }, timeUntilRefresh);
            }
        } catch (error) {
            console.error('Failed to schedule token refresh:', error);
        }
    }

    private clearRefreshTimer(): void {
        if (this.refreshTimer) {
            clearTimeout(this.refreshTimer);
            this.refreshTimer = null;
        }
    }

    private encryptToken(token: string): string {
        // Simple encryption - in production, use proper encryption
        return btoa(token);
    }

    private decryptToken(encryptedToken: string): string {
        // Simple decryption - in production, use proper decryption
        return atob(encryptedToken);
    }
}

// Authentication service with multiple providers
class AuthenticationService {
    private tokenManager: TokenManager;
    private currentUser: User | null = null;
    private authStateListeners: Set<(user: User | null) => void> = new Set();

    constructor(tokenManagerConfig: TokenManagerConfig) {
        this.tokenManager = new TokenManager(tokenManagerConfig);
        this.initializeAuthState();
    }

    // Login with email/password
    async login(credentials: LoginCredentials): Promise<AuthResult> {
        try {
            const response = await fetch('/api/auth/login', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(credentials),
            });

            if (!response.ok) {
                const error = await response.json();
                throw new AuthError(error.message, error.code);
            }

            const authData = await response.json();
            await this.handleSuccessfulAuth(authData);

            return {
                success: true,
                user: this.currentUser!,
            };
        } catch (error) {
            return {
                success: false,
                error: error.message,
            };
        }
    }

    // OAuth login (Google, GitHub, etc.)
    async loginWithOAuth(provider: OAuthProvider): Promise<void> {
        const { authUrl, codeVerifier } = await this.initiateOAuthFlow(provider);
        
        // Store code verifier for PKCE
        sessionStorage.setItem('oauth_code_verifier', codeVerifier);
        sessionStorage.setItem('oauth_provider', provider);
        
        // Redirect to OAuth provider
        window.location.href = authUrl;
    }

    // Handle OAuth callback
    async handleOAuthCallback(code: string): Promise<AuthResult> {
        const codeVerifier = sessionStorage.getItem('oauth_code_verifier');
        const provider = sessionStorage.getItem('oauth_provider');

        if (!codeVerifier || !provider) {
            throw new AuthError('OAuth state not found');
        }

        try {
            const response = await fetch('/api/auth/oauth/callback', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    code,
                    codeVerifier,
                    provider,
                }),
            });

            if (!response.ok) {
                const error = await response.json();
                throw new AuthError(error.message, error.code);
            }

            const authData = await response.json();
            await this.handleSuccessfulAuth(authData);

            // Clean up OAuth state
            sessionStorage.removeItem('oauth_code_verifier');
            sessionStorage.removeItem('oauth_provider');

            return {
                success: true,
                user: this.currentUser!,
            };
        } catch (error) {
            return {
                success: false,
                error: error.message,
            };
        }
    }

    // Logout
    async logout(): Promise<void> {
        try {
            // Call logout endpoint to invalidate tokens on server
            await fetch('/api/auth/logout', {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${await this.tokenManager.getValidAccessToken()}`,
                },
            });
        } catch (error) {
            console.error('Logout API call failed:', error);
        }

        // Clear local auth state
        this.tokenManager.clearTokens();
        this.currentUser = null;
        this.notifyAuthStateChange();
    }

    // Get current user
    getCurrentUser(): User | null {
        return this.currentUser;
    }

    // Check if user is authenticated
    isAuthenticated(): boolean {
        return this.currentUser !== null;
    }

    // Subscribe to auth state changes
    onAuthStateChange(callback: (user: User | null) => void): () => void {
        this.authStateListeners.add(callback);
        
        // Return unsubscribe function
        return () => {
            this.authStateListeners.delete(callback);
        };
    }

    private async handleSuccessfulAuth(authData: AuthData): Promise<void> {
        // Set tokens
        this.tokenManager.setTokens(authData.accessToken, authData.refreshToken);
        
        // Set current user
        this.currentUser = authData.user;
        
        // Notify listeners
        this.notifyAuthStateChange();
    }

    private async initializeAuthState(): Promise<void> {
        const accessToken = await this.tokenManager.getValidAccessToken();
        
        if (accessToken) {
            try {
                // Fetch current user data
                const response = await fetch('/api/auth/me', {
                    headers: {
                        'Authorization': `Bearer ${accessToken}`,
                    },
                });

                if (response.ok) {
                    this.currentUser = await response.json();
                    this.notifyAuthStateChange();
                } else {
                    // Invalid token, clear auth state
                    this.tokenManager.clearTokens();
                }
            } catch (error) {
                console.error('Failed to initialize auth state:', error);
                this.tokenManager.clearTokens();
            }
        }
    }

    private notifyAuthStateChange(): void {
        this.authStateListeners.forEach(callback => {
            try {
                callback(this.currentUser);
            } catch (error) {
                console.error('Auth state listener error:', error);
            }
        });
    }

    private async initiateOAuthFlow(provider: OAuthProvider): Promise<OAuthInitiation> {
        // Generate PKCE code verifier and challenge
        const codeVerifier = this.generateCodeVerifier();
        const codeChallenge = await this.generateCodeChallenge(codeVerifier);

        const params = new URLSearchParams({
            response_type: 'code',
            client_id: this.getClientId(provider),
            redirect_uri: this.getRedirectUri(),
            scope: this.getScope(provider),
            code_challenge: codeChallenge,
            code_challenge_method: 'S256',
            state: this.generateState(),
        });

        return {
            authUrl: `${this.getAuthUrl(provider)}?${params.toString()}`,
            codeVerifier,
        };
    }
}
```

### 2. **Role-Based Access Control (RBAC)**

```typescript
// Comprehensive RBAC implementation
class RoleBasedAccessControl {
    private permissions: Map<string, Permission[]> = new Map();
    private roleHierarchy: Map<string, string[]> = new Map();
    private userRoles: Set<string> = new Set();

    constructor(private user: User) {
        this.initializeUserRoles();
        this.setupPermissions();
        this.setupRoleHierarchy();
    }

    // Check if user has specific permission
    hasPermission(permission: string, resource?: string, context?: AccessContext): boolean {
        // Check direct permissions
        const userPermissions = this.getUserPermissions();
        
        for (const perm of userPermissions) {
            if (this.permissionMatches(perm, permission, resource, context)) {
                return true;
            }
        }

        return false;
    }

    // Check if user has role
    hasRole(role: string): boolean {
        // Check direct roles
        if (this.userRoles.has(role)) {
            return true;
        }

        // Check inherited roles through hierarchy
        for (const userRole of this.userRoles) {
            if (this.inheritsRole(userRole, role)) {
                return true;
            }
        }

        return false;
    }

    // Get effective permissions for user
    getUserPermissions(): Permission[] {
        const allPermissions: Permission[] = [];

        // Get permissions from direct roles
        for (const role of this.userRoles) {
            const rolePermissions = this.permissions.get(role) || [];
            allPermissions.push(...rolePermissions);
        }

        // Get permissions from inherited roles
        for (const role of this.userRoles) {
            const inheritedRoles = this.getInheritedRoles(role);
            for (const inheritedRole of inheritedRoles) {
                const rolePermissions = this.permissions.get(inheritedRole) || [];
                allPermissions.push(...rolePermissions);
            }
        }

        // Remove duplicates and return
        return this.deduplicatePermissions(allPermissions);
    }

    // Dynamic permission checking with context
    canPerformAction(action: string, resourceType: string, resourceId?: string, context?: ActionContext): boolean {
        const permission = `${action}:${resourceType}`;
        
        // Check basic permission
        if (!this.hasPermission(permission)) {
            return false;
        }

        // Check resource-specific constraints
        if (resourceId && !this.checkResourceConstraints(action, resourceType, resourceId, context)) {
            return false;
        }

        // Check time-based constraints
        if (!this.checkTimeConstraints(action, context)) {
            return false;
        }

        // Check conditional permissions
        return this.checkConditionalPermissions(action, resourceType, context);
    }

    private setupPermissions(): void {
        // Admin permissions
        this.permissions.set('admin', [
            { action: '*', resource: '*', conditions: [] },
        ]);

        // Manager permissions
        this.permissions.set('manager', [
            { action: 'read', resource: 'user', conditions: [] },
            { action: 'update', resource: 'user', conditions: ['same_department'] },
            { action: 'create', resource: 'project', conditions: [] },
            { action: 'update', resource: 'project', conditions: ['project_owner'] },
            { action: 'delete', resource: 'project', conditions: ['project_owner'] },
        ]);

        // Developer permissions
        this.permissions.set('developer', [
            { action: 'read', resource: 'project', conditions: ['team_member'] },
            { action: 'update', resource: 'project', conditions: ['team_member'] },
            { action: 'create', resource: 'task', conditions: ['assigned_project'] },
            { action: 'update', resource: 'task', conditions: ['task_assignee'] },
        ]);

        // User permissions
        this.permissions.set('user', [
            { action: 'read', resource: 'profile', conditions: ['self'] },
            { action: 'update', resource: 'profile', conditions: ['self'] },
        ]);
    }

    private setupRoleHierarchy(): void {
        // Define role inheritance
        this.roleHierarchy.set('admin', ['manager', 'developer', 'user']);
        this.roleHierarchy.set('manager', ['developer', 'user']);
        this.roleHierarchy.set('developer', ['user']);
    }

    private permissionMatches(
        permission: Permission, 
        requiredAction: string, 
        requiredResource?: string,
        context?: AccessContext
    ): boolean {
        // Check action match
        if (permission.action !== '*' && permission.action !== requiredAction) {
            return false;
        }

        // Check resource match
        if (permission.resource !== '*' && permission.resource !== requiredResource) {
            return false;
        }

        // Check conditions
        return this.checkConditions(permission.conditions, context);
    }

    private checkConditions(conditions: string[], context?: AccessContext): boolean {
        if (!conditions.length) return true;
        if (!context) return false;

        return conditions.every(condition => {
            switch (condition) {
                case 'self':
                    return context.userId === this.user.id;
                case 'same_department':
                    return context.userDepartment === this.user.department;
                case 'team_member':
                    return context.projectTeam?.includes(this.user.id);
                case 'project_owner':
                    return context.projectOwner === this.user.id;
                case 'task_assignee':
                    return context.taskAssignee === this.user.id;
                default:
                    return false;
            }
        });
    }

    private checkResourceConstraints(
        action: string, 
        resourceType: string, 
        resourceId: string, 
        context?: ActionContext
    ): boolean {
        // Example: Users can only edit their own profiles
        if (resourceType === 'profile' && action === 'update') {
            return resourceId === this.user.id;
        }

        // Example: Developers can only view projects they're assigned to
        if (resourceType === 'project' && action === 'read') {
            return context?.projectTeam?.includes(this.user.id) || false;
        }

        return true;
    }
}

// Permission-based component wrapper
const withPermission = (
    requiredPermission: string,
    fallbackComponent?: React.ComponentType
) => {
    return function PermissionWrapper<P extends object>(
        WrappedComponent: React.ComponentType<P>
    ): React.ComponentType<P> {
        return function WithPermissionComponent(props: P) {
            const { user, rbac } = useAuth();
            
            if (!user || !rbac.hasPermission(requiredPermission)) {
                if (fallbackComponent) {
                    const FallbackComponent = fallbackComponent;
                    return <FallbackComponent />;
                }
                return null;
            }

            return <WrappedComponent {...props} />;
        };
    };
};

// React hook for permission checking
function usePermissions() {
    const { user, rbac } = useAuth();

    const hasPermission = useCallback((
        permission: string,
        resource?: string,
        context?: AccessContext
    ) => {
        if (!user || !rbac) return false;
        return rbac.hasPermission(permission, resource, context);
    }, [user, rbac]);

    const canPerformAction = useCallback((
        action: string,
        resourceType: string,
        resourceId?: string,
        context?: ActionContext
    ) => {
        if (!user || !rbac) return false;
        return rbac.canPerformAction(action, resourceType, resourceId, context);
    }, [user, rbac]);

    const hasRole = useCallback((role: string) => {
        if (!user || !rbac) return false;
        return rbac.hasRole(role);
    }, [user, rbac]);

    return {
        hasPermission,
        canPerformAction,
        hasRole,
        userPermissions: rbac?.getUserPermissions() || [],
        userRoles: user?.roles || [],
    };
}
```

### 3. **Security Best Practices and Implementation**

```typescript
// Comprehensive security framework
class SecurityManager {
    private csrfToken: string | null = null;
    private securityHeaders: Map<string, string> = new Map();

    constructor() {
        this.initializeCSRFProtection();
        this.setupSecurityHeaders();
        this.setupContentSecurityPolicy();
    }

    // CSRF protection
    private initializeCSRFProtection(): void {
        // Get CSRF token from meta tag or API
        const metaToken = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');
        if (metaToken) {
            this.csrfToken = metaToken;
        } else {
            this.fetchCSRFToken();
        }
    }

    private async fetchCSRFToken(): Promise<void> {
        try {
            const response = await fetch('/api/csrf-token', {
                credentials: 'same-origin',
            });
            const data = await response.json();
            this.csrfToken = data.token;
        } catch (error) {
            console.error('Failed to fetch CSRF token:', error);
        }
    }

    // Add security headers to requests
    addSecurityHeaders(headers: Record<string, string> = {}): Record<string, string> {
        const securityHeaders = {
            ...headers,
            'X-Requested-With': 'XMLHttpRequest',
        };

        if (this.csrfToken) {
            securityHeaders['X-CSRF-Token'] = this.csrfToken;
        }

        return securityHeaders;
    }

    // Content Security Policy implementation
    private setupContentSecurityPolicy(): void {
        const csp = [
            "default-src 'self'",
            "script-src 'self' 'unsafe-inline' https://apis.google.com",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
            "font-src 'self' https://fonts.gstatic.com",
            "img-src 'self' data: https:",
            "connect-src 'self' https://api.example.com",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'",
        ].join('; ');

        // Set CSP via meta tag if not set by server
        if (!document.querySelector('meta[http-equiv="Content-Security-Policy"]')) {
            const meta = document.createElement('meta');
            meta.httpEquiv = 'Content-Security-Policy';
            meta.content = csp;
            document.head.appendChild(meta);
        }
    }

    // Input validation and sanitization
    validateInput(input: string, type: ValidationType): ValidationResult {
        const validators = {
            email: (value: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
            password: (value: string) => value.length >= 8 && /(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(value),
            username: (value: string) => /^[a-zA-Z0-9_]{3,20}$/.test(value),
            url: (value: string) => {
                try {
                    const url = new URL(value);
                    return ['http:', 'https:'].includes(url.protocol);
                } catch {
                    return false;
                }
            },
        };

        const isValid = validators[type]?.(input) || false;
        
        return {
            isValid,
            sanitized: this.sanitizeInput(input, type),
            errors: isValid ? [] : [`Invalid ${type} format`],
        };
    }

    private sanitizeInput(input: string, type: ValidationType): string {
        // Basic HTML escaping
        const htmlEscaped = input
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#x27;');

        // Type-specific sanitization
        switch (type) {
            case 'email':
                return htmlEscaped.toLowerCase().trim();
            case 'username':
                return htmlEscaped.toLowerCase().replace(/[^a-z0-9_]/g, '');
            default:
                return htmlEscaped.trim();
        }
    }

    // XSS protection
    sanitizeHTML(html: string): string {
        const div = document.createElement('div');
        div.textContent = html;
        return div.innerHTML;
    }

    // Secure storage utilities
    secureStorage = {
        setItem: (key: string, value: string, encrypt: boolean = true): void => {
            try {
                const finalValue = encrypt ? this.encrypt(value) : value;
                localStorage.setItem(key, finalValue);
            } catch (error) {
                console.error('Secure storage setItem failed:', error);
            }
        },

        getItem: (key: string, decrypt: boolean = true): string | null => {
            try {
                const value = localStorage.getItem(key);
                if (!value) return null;
                return decrypt ? this.decrypt(value) : value;
            } catch (error) {
                console.error('Secure storage getItem failed:', error);
                return null;
            }
        },

        removeItem: (key: string): void => {
            localStorage.removeItem(key);
        },
    };

    private encrypt(data: string): string {
        // Simple encryption for demo - use proper encryption in production
        return btoa(data);
    }

    private decrypt(encryptedData: string): string {
        // Simple decryption for demo - use proper decryption in production
        return atob(encryptedData);
    }

    // Security event logging
    logSecurityEvent(event: SecurityEvent): void {
        const logEntry = {
            timestamp: Date.now(),
            type: event.type,
            details: event.details,
            userAgent: navigator.userAgent,
            url: window.location.href,
            userId: event.userId,
        };

        // Send to security monitoring service
        fetch('/api/security/log', {
            method: 'POST',
            headers: this.addSecurityHeaders({
                'Content-Type': 'application/json',
            }),
            body: JSON.stringify(logEntry),
        }).catch(error => {
            console.error('Security event logging failed:', error);
        });
    }
}

// Secure API request wrapper
class SecureAPIClient {
    private securityManager: SecurityManager;
    private tokenManager: TokenManager;

    constructor(securityManager: SecurityManager, tokenManager: TokenManager) {
        this.securityManager = securityManager;
        this.tokenManager = tokenManager;
    }

    async secureRequest(url: string, options: RequestInit = {}): Promise<Response> {
        // Add authentication
        const accessToken = await this.tokenManager.getValidAccessToken();
        
        // Prepare secure headers
        const headers = this.securityManager.addSecurityHeaders({
            'Content-Type': 'application/json',
            ...(options.headers as Record<string, string>),
        });

        if (accessToken) {
            headers['Authorization'] = `Bearer ${accessToken}`;
        }

        // Add security options
        const secureOptions: RequestInit = {
            ...options,
            headers,
            credentials: 'same-origin', // Include cookies for CSRF protection
            mode: 'cors',
            cache: 'no-store', // Prevent sensitive data caching
        };

        try {
            const response = await fetch(url, secureOptions);
            
            // Log security events
            if (response.status === 401) {
                this.securityManager.logSecurityEvent({
                    type: 'unauthorized_access',
                    details: `Unauthorized request to ${url}`,
                    userId: this.getCurrentUserId(),
                });
            }

            return response;
        } catch (error) {
            // Log security events for failed requests
            this.securityManager.logSecurityEvent({
                type: 'request_error',
                details: `Request failed: ${error.message}`,
                userId: this.getCurrentUserId(),
            });
            
            throw error;
        }
    }

    private getCurrentUserId(): string | undefined {
        // Get current user ID from auth context
        return undefined; // Implementation depends on auth setup
    }
}
```

## Interview Questions & Answers

### Q: How do you handle token refresh without interrupting user experience?

**A:** I implement silent token refresh using refresh tokens before the access token expires, queue API requests during refresh to prevent failures, use interceptors to automatically retry failed requests after refresh, and maintain a single refresh promise to prevent concurrent refresh attempts. I also schedule proactive refresh based on token expiration time.

### Q: What's your approach to implementing role-based access control in a React application?

**A:** I create a centralized RBAC system with permission checking hooks, implement HOCs and components for conditional rendering based on permissions, use context providers for auth state management, and implement both frontend and backend permission validation. I also support dynamic permissions with resource-specific contexts and role hierarchies for scalable permission management.

### Q: How do you secure sensitive data in frontend applications?

**A:** I minimize sensitive data exposure by keeping secrets on the backend, encrypt data before storing in localStorage, use httpOnly cookies for sensitive tokens, implement proper CSRF protection, validate and sanitize all inputs, and use CSP headers to prevent XSS. I also implement secure communication protocols and log security events for monitoring.

### Q: How do you handle authentication in a micro-frontend architecture?

**A:** I implement a shared authentication service that all micro-frontends can use, use a centralized identity provider with SSO, implement token sharing through secure mechanisms like postMessage or shared state, and ensure consistent auth state across all micro-frontends. I also handle token refresh coordination and implement proper logout across all applications.

### Interview Tip:
*"Frontend authentication requires balancing security with user experience. Focus on secure token management, comprehensive error handling, and seamless user flows. Always implement both client-side and server-side validation, and never trust the frontend alone for security decisions."*