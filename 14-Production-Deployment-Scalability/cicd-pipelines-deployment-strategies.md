# CI/CD Pipelines and Deployment Strategies

## Question
How do you design and implement robust CI/CD pipelines for modern frontend applications?

## Answer

Modern CI/CD pipelines require comprehensive automation, testing, security, and deployment strategies that ensure reliable, fast, and safe delivery to production. Here's a complete framework for building enterprise-grade CI/CD systems for frontend applications.

## Comprehensive CI/CD Pipeline Architecture

### 1. **Multi-Environment Pipeline Design**

```yaml
# Complete GitHub Actions CI/CD Pipeline
name: Frontend CI/CD Pipeline

on:
  push:
    branches: [main, develop, 'feature/*', 'hotfix/*']
  pull_request:
    branches: [main, develop]
  release:
    types: [published]

env:
  NODE_VERSION: '18.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/frontend

jobs:
  # Environment Setup and Validation
  setup:
    runs-on: ubuntu-latest
    outputs:
      deployment-env: ${{ steps.env-detection.outputs.environment }}
      version: ${{ steps.version.outputs.version }}
      should-deploy: ${{ steps.deploy-check.outputs.should-deploy }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Detect deployment environment
        id: env-detection
        run: |
          if [[ $GITHUB_REF == refs/heads/main ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [[ $GITHUB_REF == refs/heads/develop ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          elif [[ $GITHUB_REF == refs/pull/* ]]; then
            echo "environment=preview" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi

      - name: Generate version
        id: version
        run: |
          if [[ ${{ github.event_name }} == 'release' ]]; then
            echo "version=${{ github.event.release.tag_name }}" >> $GITHUB_OUTPUT
          else
            echo "version=${{ github.run_number }}-${{ github.sha }}" >> $GITHUB_OUTPUT
          fi

      - name: Check deployment conditions
        id: deploy-check
        run: |
          if [[ "${{ steps.env-detection.outputs.environment }}" == "production" ]] || 
             [[ "${{ steps.env-detection.outputs.environment }}" == "staging" ]] ||
             [[ "${{ github.event_name }}" == "pull_request" ]]; then
            echo "should-deploy=true" >> $GITHUB_OUTPUT
          else
            echo "should-deploy=false" >> $GITHUB_OUTPUT
          fi

  # Code Quality and Security Checks
  quality-gates:
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: |
          npm ci --prefer-offline --no-audit
          npm ls # Verify dependency tree

      - name: Code quality checks
        run: |
          # Linting with error reporting
          npm run lint -- --format=json --output-file=eslint-report.json
          npm run lint:css -- --formatter=json --output-file=stylelint-report.json
          
          # Type checking
          npm run type-check
          
          # Code formatting check
          npm run format:check

      - name: Security scanning
        run: |
          # Dependency vulnerability scan
          npm audit --audit-level=moderate --json > npm-audit.json || true
          
          # License compliance check
          npx license-checker --onlyAllow 'MIT;Apache-2.0;BSD-3-Clause;BSD-2-Clause;ISC' --json > license-check.json
          
          # Security linting
          npm run lint:security

      - name: Code complexity analysis
        run: |
          # Complexity analysis
          npx plato -r -d complexity-report src/
          
          # Bundle analysis preparation
          npm run build:analyze

      - name: Upload quality reports
        uses: actions/upload-artifact@v4
        with:
          name: quality-reports
          path: |
            eslint-report.json
            stylelint-report.json
            npm-audit.json
            license-check.json
            complexity-report/
          retention-days: 30

  # Comprehensive Testing Suite
  test-suite:
    runs-on: ubuntu-latest
    needs: setup
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci --prefer-offline --no-audit

      - name: Run unit tests
        if: matrix.test-type == 'unit'
        run: |
          npm run test:unit -- --coverage --watchAll=false --passWithNoTests
          npm run test:coverage-threshold

      - name: Run integration tests
        if: matrix.test-type == 'integration'
        run: |
          # Start test services
          docker-compose -f docker-compose.test.yml up -d
          
          # Wait for services
          npm run wait-for-services
          
          # Run integration tests
          npm run test:integration
          
          # Cleanup
          docker-compose -f docker-compose.test.yml down

      - name: Run E2E tests
        if: matrix.test-type == 'e2e'
        run: |
          # Build application for E2E testing
          npm run build:test
          
          # Start application server
          npm run serve:test &
          
          # Wait for server
          npm run wait-for-server
          
          # Run E2E tests
          npm run test:e2e -- --browser=chromium,firefox,webkit

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.test-type }}
          path: |
            coverage/
            test-results/
            playwright-report/
          retention-days: 30

  # Performance and Accessibility Testing
  performance-audit:
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci --prefer-offline --no-audit

      - name: Build application
        run: |
          npm run build
          npm run build:stats

      - name: Bundle size analysis
        run: |
          # Analyze bundle size
          npm run analyze:bundle -- --json > bundle-analysis.json
          
          # Check bundle size limits
          npm run check:bundle-size

      - name: Lighthouse CI
        run: |
          # Install Lighthouse CI
          npm install -g @lhci/cli
          
          # Start server
          npm run serve &
          
          # Wait for server
          npm run wait-for-server
          
          # Run Lighthouse CI
          lhci autorun --config=lighthouserc.js

      - name: Accessibility testing
        run: |
          # Automated accessibility testing
          npm run test:a11y
          
          # Pa11y testing
          npx pa11y-ci --sitemap http://localhost:3000/sitemap.xml

      - name: Upload performance reports
        uses: actions/upload-artifact@v4
        with:
          name: performance-reports
          path: |
            bundle-analysis.json
            .lighthouseci/
            accessibility-report/
          retention-days: 30

  # Container Build and Security Scan
  build-container:
    runs-on: ubuntu-latest
    needs: [setup, quality-gates, test-suite]
    if: needs.setup.outputs.should-deploy == 'true'
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=raw,value=${{ needs.setup.outputs.version }}
          labels: |
            org.opencontainers.image.title=Frontend Application
            org.opencontainers.image.description=Production-ready frontend application
            org.opencontainers.image.vendor=Your Company

      - name: Build and push container
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.production
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_VERSION=${{ needs.setup.outputs.version }}
            NODE_ENV=production

      - name: Container security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.setup.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload security scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # Multi-Environment Deployment
  deploy:
    runs-on: ubuntu-latest
    needs: [setup, build-container, performance-audit]
    if: needs.setup.outputs.should-deploy == 'true'
    environment: ${{ needs.setup.outputs.deployment-env }}
    strategy:
      matrix:
        include:
          - environment: preview
            deploy-script: deploy-preview.sh
          - environment: staging
            deploy-script: deploy-staging.sh
          - environment: production
            deploy-script: deploy-production.sh
    steps:
      - name: Checkout deployment scripts
        uses: actions/checkout@v4
        with:
          sparse-checkout: |
            .github/deployment/
            k8s/
            terraform/

      - name: Setup deployment tools
        run: |
          # Install kubectl
          curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl && sudo mv kubectl /usr/local/bin/
          
          # Install Helm
          curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
          
          # Install Terraform
          wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
          unzip terraform_1.6.0_linux_amd64.zip && sudo mv terraform /usr/local/bin/

      - name: Configure deployment environment
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          # Setup Kubernetes config
          echo "$KUBECONFIG_DATA" | base64 -d > kubeconfig
          export KUBECONFIG=./kubeconfig
          
          # Verify cluster access
          kubectl cluster-info

      - name: Deploy application
        env:
          IMAGE_TAG: ${{ needs.setup.outputs.version }}
          ENVIRONMENT: ${{ needs.setup.outputs.deployment-env }}
        run: |
          # Deploy using Helm
          helm upgrade --install frontend-${{ matrix.environment }} \
            ./k8s/helm/frontend \
            --namespace ${{ matrix.environment }} \
            --create-namespace \
            --set image.tag=${{ needs.setup.outputs.version }} \
            --set environment=${{ matrix.environment }} \
            --values ./k8s/values/${{ matrix.environment }}.yaml \
            --wait --timeout=10m

      - name: Run deployment tests
        run: |
          # Health check
          kubectl wait --for=condition=available deployment/frontend \
            --namespace=${{ matrix.environment }} --timeout=300s
          
          # Smoke tests
          npm run test:smoke -- --env=${{ matrix.environment }}

      - name: Update deployment status
        run: |
          # Update deployment status in monitoring system
          curl -X POST "${{ secrets.DEPLOYMENT_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d '{
              "environment": "${{ matrix.environment }}",
              "version": "${{ needs.setup.outputs.version }}",
              "status": "deployed",
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
            }'

  # Post-deployment monitoring and notifications
  post-deployment:
    runs-on: ubuntu-latest
    needs: [setup, deploy]
    if: always() && needs.deploy.result == 'success'
    steps:
      - name: Deployment verification
        run: |
          # Wait for deployment to be fully ready
          sleep 30
          
          # Run comprehensive health checks
          curl -f "${{ secrets.APP_URL }}/health" || exit 1
          
          # Run critical user journey tests
          npm run test:critical-path -- --env=${{ needs.setup.outputs.deployment-env }}

      - name: Performance baseline update
        if: needs.setup.outputs.deployment-env == 'production'
        run: |
          # Update performance baselines
          npm run update:performance-baseline -- --version=${{ needs.setup.outputs.version }}

      - name: Notify deployment success
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            const environment = '${{ needs.setup.outputs.deployment-env }}';
            const version = '${{ needs.setup.outputs.version }}';
            
            // Create deployment status
            await github.rest.repos.createDeploymentStatus({
              owner,
              repo,
              deployment_id: context.payload.deployment?.id,
              state: 'success',
              environment_url: '${{ secrets.APP_URL }}',
              description: `Deployment ${version} successful to ${environment}`
            });

      - name: Slack notification
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: `🚀 Deployment Complete`,
              blocks: [
                {
                  type: "section",
                  text: {
                    type: "mrkdwn",
                    text: `*Frontend Application Deployed*\n*Environment:* ${{ needs.setup.outputs.deployment-env }}\n*Version:* ${{ needs.setup.outputs.version }}\n*Status:* ✅ Success`
                  }
                },
                {
                  type: "actions",
                  elements: [
                    {
                      type: "button",
                      text: {
                        type: "plain_text",
                        text: "View Application"
                      },
                      url: "${{ secrets.APP_URL }}"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 2. **Advanced Deployment Strategies**

```typescript
// Deployment strategy configuration
interface DeploymentStrategy {
    type: 'blue-green' | 'canary' | 'rolling' | 'feature-flag';
    configuration: DeploymentConfig;
    rollback: RollbackStrategy;
    monitoring: MonitoringConfig;
}

// Blue-Green deployment implementation
class BlueGreenDeployment implements DeploymentStrategy {
    type = 'blue-green' as const;
    
    async deploy(config: BlueGreenConfig): Promise<DeploymentResult> {
        // Phase 1: Prepare green environment
        await this.prepareGreenEnvironment(config);
        
        // Phase 2: Deploy to green environment
        const deploymentResult = await this.deployToGreen(config);
        
        // Phase 3: Run verification tests
        const verificationResult = await this.runVerificationTests(config);
        
        if (!verificationResult.success) {
            await this.cleanupGreenEnvironment(config);
            throw new Error(`Verification failed: ${verificationResult.errors.join(', ')}`);
        }
        
        // Phase 4: Switch traffic
        await this.switchTraffic(config);
        
        // Phase 5: Monitor and validate
        const monitoringResult = await this.monitorPostDeployment(config);
        
        if (!monitoringResult.success) {
            await this.rollback(config);
            throw new Error(`Post-deployment monitoring failed: ${monitoringResult.errors.join(', ')}`);
        }
        
        // Phase 6: Cleanup old environment
        await this.cleanupBlueEnvironment(config);
        
        return {
            success: true,
            environment: 'green',
            version: config.version,
            deploymentTime: Date.now(),
        };
    }

    private async prepareGreenEnvironment(config: BlueGreenConfig): Promise<void> {
        // Create green environment infrastructure
        await this.executeScript(`
            # Create green environment
            kubectl create namespace ${config.greenNamespace}
            
            # Copy blue environment configuration
            kubectl get configmap app-config -n ${config.blueNamespace} -o yaml | 
            sed 's/${config.blueNamespace}/${config.greenNamespace}/g' | 
            kubectl apply -f -
            
            # Update configuration for green environment
            kubectl patch configmap app-config -n ${config.greenNamespace} --patch '
            data:
              ENVIRONMENT: "green"
              VERSION: "${config.version}"
              FEATURE_FLAGS: "${config.featureFlags}"
            '
        `);
    }

    private async deployToGreen(config: BlueGreenConfig): Promise<DeploymentResult> {
        // Deploy application to green environment
        const helmCommand = `
            helm upgrade --install ${config.appName}-green ./helm/chart \\
                --namespace ${config.greenNamespace} \\
                --set image.tag=${config.version} \\
                --set environment=green \\
                --set replicas=${config.replicas} \\
                --set resources.requests.cpu=${config.resources.cpu} \\
                --set resources.requests.memory=${config.resources.memory} \\
                --values ./helm/values/production.yaml \\
                --wait --timeout=10m
        `;
        
        return await this.executeScript(helmCommand);
    }

    private async runVerificationTests(config: BlueGreenConfig): Promise<VerificationResult> {
        const tests = [
            this.runHealthChecks(config),
            this.runSmokeTests(config),
            this.runIntegrationTests(config),
            this.runPerformanceTests(config),
        ];
        
        const results = await Promise.allSettled(tests);
        
        const failures = results
            .filter(result => result.status === 'rejected')
            .map(result => (result as PromiseRejectedResult).reason);
        
        return {
            success: failures.length === 0,
            errors: failures,
            results: results.filter(r => r.status === 'fulfilled').map(r => (r as PromiseFulfilledResult<any>).value),
        };
    }

    private async switchTraffic(config: BlueGreenConfig): Promise<void> {
        // Gradually switch traffic from blue to green
        const trafficSteps = [10, 25, 50, 75, 100];
        
        for (const percentage of trafficSteps) {
            await this.updateTrafficSplit(config, percentage);
            await this.monitorTrafficSplit(config, 2 * 60 * 1000); // Monitor for 2 minutes
            
            const healthCheck = await this.checkApplicationHealth(config);
            if (!healthCheck.healthy) {
                throw new Error(`Health check failed at ${percentage}% traffic split`);
            }
        }
    }

    private async updateTrafficSplit(config: BlueGreenConfig, greenPercentage: number): Promise<void> {
        const bluePercentage = 100 - greenPercentage;
        
        // Update ingress configuration for traffic splitting
        await this.executeScript(`
            kubectl patch ingress ${config.ingressName} -n ${config.namespace} --patch '
            metadata:
              annotations:
                nginx.ingress.kubernetes.io/canary: "true"
                nginx.ingress.kubernetes.io/canary-weight: "${greenPercentage}"
            spec:
              rules:
              - host: ${config.domain}
                http:
                  paths:
                  - path: /
                    pathType: Prefix
                    backend:
                      service:
                        name: ${config.appName}-green
                        port:
                          number: 80
            '
        `);
    }

    private async rollback(config: BlueGreenConfig): Promise<void> {
        console.log('Starting rollback to blue environment...');
        
        // Switch traffic back to blue
        await this.updateTrafficSplit(config, 0);
        
        // Cleanup green environment
        await this.cleanupGreenEnvironment(config);
        
        // Notify about rollback
        await this.notifyRollback(config);
    }
}

// Canary deployment strategy
class CanaryDeployment implements DeploymentStrategy {
    type = 'canary' as const;
    
    async deploy(config: CanaryConfig): Promise<DeploymentResult> {
        // Deploy canary version with small traffic percentage
        await this.deployCanaryVersion(config);
        
        // Gradually increase traffic based on metrics
        const canaryResult = await this.executeCanaryStrategy(config);
        
        if (canaryResult.success) {
            // Promote canary to production
            await this.promoteCanary(config);
        } else {
            // Rollback canary
            await this.rollbackCanary(config);
            throw new Error(`Canary deployment failed: ${canaryResult.errors.join(', ')}`);
        }
        
        return canaryResult;
    }

    private async executeCanaryStrategy(config: CanaryConfig): Promise<DeploymentResult> {
        const stages = config.canaryStages || [
            { percentage: 5, duration: 5 * 60 * 1000, successCriteria: { errorRate: 0.01, latency: 500 } },
            { percentage: 10, duration: 10 * 60 * 1000, successCriteria: { errorRate: 0.01, latency: 500 } },
            { percentage: 25, duration: 15 * 60 * 1000, successCriteria: { errorRate: 0.01, latency: 500 } },
            { percentage: 50, duration: 20 * 60 * 1000, successCriteria: { errorRate: 0.01, latency: 500 } },
            { percentage: 100, duration: 30 * 60 * 1000, successCriteria: { errorRate: 0.01, latency: 500 } },
        ];

        for (const stage of stages) {
            console.log(`Canary stage: ${stage.percentage}% traffic for ${stage.duration}ms`);
            
            // Update traffic split
            await this.updateCanaryTraffic(config, stage.percentage);
            
            // Monitor stage
            const stageResult = await this.monitorCanaryStage(config, stage);
            
            if (!stageResult.success) {
                return {
                    success: false,
                    errors: [`Stage ${stage.percentage}% failed: ${stageResult.errors.join(', ')}`],
                    stage: stage.percentage,
                };
            }
            
            // Wait for stage duration
            await new Promise(resolve => setTimeout(resolve, stage.duration));
        }

        return { success: true, errors: [], stage: 100 };
    }

    private async monitorCanaryStage(
        config: CanaryConfig, 
        stage: CanaryStage
    ): Promise<{ success: boolean; errors: string[] }> {
        const monitoringDuration = Math.min(stage.duration, 5 * 60 * 1000); // Max 5 minutes
        const checkInterval = 30 * 1000; // Check every 30 seconds
        
        const startTime = Date.now();
        const errors: string[] = [];

        while (Date.now() - startTime < monitoringDuration) {
            // Get metrics for canary version
            const metrics = await this.getCanaryMetrics(config);
            
            // Check error rate
            if (metrics.errorRate > stage.successCriteria.errorRate) {
                errors.push(`Error rate ${metrics.errorRate} exceeds threshold ${stage.successCriteria.errorRate}`);
            }
            
            // Check latency
            if (metrics.p95Latency > stage.successCriteria.latency) {
                errors.push(`P95 latency ${metrics.p95Latency}ms exceeds threshold ${stage.successCriteria.latency}ms`);
            }
            
            // Check business metrics
            if (config.businessMetrics) {
                const businessResult = await this.checkBusinessMetrics(config, metrics);
                if (!businessResult.success) {
                    errors.push(...businessResult.errors);
                }
            }
            
            // Early exit on errors
            if (errors.length > 0) {
                return { success: false, errors };
            }
            
            await new Promise(resolve => setTimeout(resolve, checkInterval));
        }

        return { success: true, errors: [] };
    }
}

// Feature flag based deployment
class FeatureFlagDeployment implements DeploymentStrategy {
    type = 'feature-flag' as const;
    
    async deploy(config: FeatureFlagConfig): Promise<DeploymentResult> {
        // Deploy with features disabled
        await this.deployWithFlagsDisabled(config);
        
        // Gradually enable features
        const enablementResult = await this.gradualFeatureEnablement(config);
        
        return enablementResult;
    }

    private async gradualFeatureEnablement(config: FeatureFlagConfig): Promise<DeploymentResult> {
        const features = config.features;
        
        for (const feature of features) {
            console.log(`Enabling feature: ${feature.name}`);
            
            // Enable for internal users first
            await this.enableFeatureForGroup(feature, 'internal');
            await this.monitorFeature(feature, 5 * 60 * 1000); // 5 minutes
            
            // Enable for beta users
            await this.enableFeatureForGroup(feature, 'beta');
            await this.monitorFeature(feature, 10 * 60 * 1000); // 10 minutes
            
            // Gradually roll out to all users
            const rolloutPercentages = [1, 5, 10, 25, 50, 100];
            
            for (const percentage of rolloutPercentages) {
                await this.enableFeatureForPercentage(feature, percentage);
                
                const result = await this.monitorFeature(feature, 5 * 60 * 1000);
                
                if (!result.success) {
                    await this.disableFeature(feature);
                    return {
                        success: false,
                        errors: [`Feature ${feature.name} failed at ${percentage}%: ${result.errors.join(', ')}`],
                    };
                }
            }
        }

        return { success: true, errors: [] };
    }
}
```

### 3. **Infrastructure as Code and Environment Management**

```typescript
// Terraform configuration for AWS infrastructure
const terraformInfrastructure = `
# Provider configuration
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  
  backend "s3" {
    bucket = "your-terraform-state"
    key    = "frontend/infrastructure.tfstate"
    region = "us-west-2"
  }
}

# Variables
variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "app_name" {
  description = "Application name"
  type        = string
  default     = "frontend-app"
}

# VPC Configuration
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "\${var.app_name}-\${var.environment}"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Environment = var.environment
    Project     = var.app_name
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "\${var.app_name}-\${var.environment}"
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # Node groups
  eks_managed_node_groups = {
    general = {
      desired_size = var.environment == "production" ? 3 : 2
      max_size     = var.environment == "production" ? 10 : 5
      min_size     = 1
      
      instance_types = var.environment == "production" ? ["t3.large"] : ["t3.medium"]
      
      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "general"
      }
    }
  }
  
  tags = {
    Environment = var.environment
    Project     = var.app_name
  }
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "frontend" {
  origin {
    domain_name = aws_s3_bucket.frontend_assets.bucket_regional_domain_name
    origin_id   = "S3-\${aws_s3_bucket.frontend_assets.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.frontend.cloudfront_access_identity_path
    }
  }
  
  # API origin for dynamic content
  origin {
    domain_name = "\${var.api_domain}"
    origin_id   = "API"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  
  # Cache behavior for static assets
  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-\${aws_s3_bucket.frontend_assets.id}"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    
    compress = true
  }
  
  # Cache behavior for API calls
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = "API"
    
    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Content-Type"]
      cookies {
        forward = "all"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 0
  }
  
  # Error pages
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }
  
  custom_error_response {
    error_code         = 403
    response_code      = 200
    response_page_path = "/index.html"
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.frontend.arn
    ssl_support_method  = "sni-only"
  }
  
  tags = {
    Environment = var.environment
    Project     = var.app_name
  }
}

# S3 bucket for static assets
resource "aws_s3_bucket" "frontend_assets" {
  bucket = "\${var.app_name}-\${var.environment}-assets"
  
  tags = {
    Environment = var.environment
    Project     = var.app_name
  }
}

resource "aws_s3_bucket_versioning" "frontend_assets" {
  bucket = aws_s3_bucket.frontend_assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "frontend_assets" {
  bucket = aws_s3_bucket.frontend_assets.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
`;

// Kubernetes deployment manifests
const kubernetesManifests = `
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-production
  labels:
    environment: production
    app: frontend

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: frontend-production
data:
  NODE_ENV: "production"
  API_URL: "https://api.example.com"
  CDN_URL: "https://cdn.example.com"
  SENTRY_DSN: "https://your-sentry-dsn"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: frontend-secrets
  namespace: frontend-production
type: Opaque
data:
  # Base64 encoded secrets
  jwt-secret: <base64-encoded-jwt-secret>
  api-key: <base64-encoded-api-key>

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend-production
  labels:
    app: frontend
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        version: v1.0.0
    spec:
      containers:
      - name: frontend
        image: your-registry/frontend:v1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: NODE_ENV
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: API_URL
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: frontend-production
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: frontend-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: frontend-production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - example.com
    secretName: frontend-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
`;
```

## Interview Questions & Answers

### Q: How do you handle zero-downtime deployments in production?

**A:** I use blue-green or canary deployment strategies with proper health checks, implement rolling updates with readiness probes, use feature flags for gradual rollouts, and maintain proper monitoring during deployments. I also ensure database migrations are backward compatible and have automated rollback procedures ready.

### Q: What's your approach to managing different environments (dev, staging, production)?

**A:** I use infrastructure as code with Terraform for consistent environment provisioning, implement environment-specific configuration management with proper secret handling, use GitOps workflows for deployment consistency, and maintain environment parity while allowing for appropriate scaling differences. I also implement proper promotion workflows between environments.

### Q: How do you ensure security in your CI/CD pipeline?

**A:** I implement security scanning at multiple stages, use signed container images with vulnerability scanning, implement proper secret management with rotation, use least-privilege access principles, and implement audit logging for all pipeline activities. I also use policy as code for compliance validation and implement proper access controls for deployment environments.

### Q: How do you handle rollbacks and disaster recovery?

**A:** I maintain automated rollback procedures with health check validation, implement database backup and restore procedures, use infrastructure snapshots for quick recovery, and maintain runbooks for manual intervention when needed. I also implement proper monitoring and alerting to detect issues early and test disaster recovery procedures regularly.

### Interview Tip:
*"Modern CI/CD requires balancing speed with safety and reliability. Focus on automation, testing at every stage, and proper monitoring. Always implement rollback procedures and test them regularly. Remember that the goal is reliable, repeatable deployments that give you confidence to deploy frequently."*