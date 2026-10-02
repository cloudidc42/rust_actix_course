# Part 074: CI/CD with GitHub Actions

## บทนำ

GitHub Actions ช่วยให้เราสามารถ automate กระบวนการ build, test, และ deploy แอปพลิเคชัน Rust บทนี้จะครอบคลุมการสร้าง CI/CD pipeline ที่สมบูรณ์

## 1. GitHub Actions Workflow พื้นฐาน

### 1.1 โครงสร้าง Workflow

```
.github/
└── workflows/
    ├── ci.yml          # Continuous Integration
    ├── cd.yml          # Continuous Deployment  
    ├── security.yml    # Security checks
    └── release.yml     # Release workflow
```

### 1.2 Basic CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  RUST_VERSION: "1.74"

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 3
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: ${{ env.RUST_VERSION }}
      
      - name: Cache dependencies
        uses: Swatinem/rust-cache@v2
        with:
          cache-on-failure: true
      
      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret-for-ci
        run: |
          cargo test --all-features --workspace
```

## 2. Cargo Test in CI

### 2.1 Comprehensive Test Job

```yaml
# .github/workflows/ci.yml (test section)
  test-all:
    name: Test All
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
      
      - uses: Swatinem/rust-cache@v2
      
      # Install sqlx-cli สำหรับ migrations
      - name: Install sqlx-cli
        run: |
          cargo install sqlx-cli \
            --no-default-features \
            --features native-tls,postgres
      
      # Run migrations
      - name: Run database migrations
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
        run: sqlx migrate run
      
      # Run unit tests
      - name: Run unit tests
        run: cargo test --lib
      
      # Run integration tests
      - name: Run integration tests
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
          TEST_JWT_SECRET: test-secret
        run: cargo test --test '*'
      
      # Run doc tests
      - name: Run doc tests
        run: cargo test --doc
      
      # Test with all features
      - name: Test with all features
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
        run: cargo test --all-features
      
      # Test with no default features
      - name: Test with no features
        run: cargo test --no-default-features
```

### 2.2 Matrix Testing

```yaml
# Test บน multiple Rust versions และ OS
  test-matrix:
    name: Test (${{ matrix.os }}, Rust ${{ matrix.rust }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        rust: [stable, beta]
        exclude:
          # ลด CI time - skip Windows/macOS on beta
          - os: windows-latest
            rust: beta
          - os: macos-latest
            rust: beta
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust ${{ matrix.rust }}
        uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}
      
      - uses: Swatinem/rust-cache@v2
      
      - name: Run tests
        run: cargo test
```

## 3. Clippy Lint Check

```yaml
# .github/workflows/ci.yml (clippy section)
  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      
      - uses: Swatinem/rust-cache@v2
      
      - name: Run Clippy
        run: |
          cargo clippy \
            --all-targets \
            --all-features \
            -- \
            -D warnings \
            -W clippy::pedantic \
            -A clippy::module_name_repetitions \
            -A clippy::must_use_candidate
      
      # Annotate PR กับ Clippy results
      - name: Annotate with Clippy
        if: github.event_name == 'pull_request'
        uses: giraffate/clippy-action@v1
        with:
          reporter: 'github-pr-check'
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

```toml
# .cargo/config.toml - Clippy configuration
[target.'cfg(all())']
rustflags = [
    "-D", "warnings",
]
```

## 4. Rustfmt Format Check

```yaml
# .github/workflows/ci.yml (format section)
  fmt:
    name: Rustfmt
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      
      - name: Check formatting
        run: cargo fmt --all -- --check
      
      # ถ้า format ไม่ตรง แสดง diff
      - name: Show diff on failure
        if: failure()
        run: cargo fmt --all -- 2>&1 | head -100
```

```toml
# rustfmt.toml
edition = "2021"
max_width = 100
tab_spaces = 4
use_small_heuristics = "Default"
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
```

## 5. cargo-audit Security Check

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 8 * * 1'  # ทุกวันจันทร์ 8am

jobs:
  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
      
      - name: Install cargo-audit
        run: cargo install cargo-audit --locked
      
      - name: Run security audit
        run: |
          cargo audit \
            --ignore RUSTSEC-2020-0071 \
            --deny warnings
      
      # Generate audit report
      - name: Generate audit report
        if: always()
        run: |
          cargo audit --json > audit-report.json || true
          cat audit-report.json
      
      - name: Upload audit report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: security-audit-report
          path: audit-report.json
  
  # Check for outdated dependencies
  deny:
    name: Cargo Deny
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install cargo-deny
        run: cargo install cargo-deny --locked
      
      - name: Check licenses and bans
        run: cargo deny check
```

```toml
# deny.toml - cargo-deny configuration
[advisories]
ignore = [
    # Add RUSTSEC IDs to ignore here
]

[licenses]
allow = [
    "MIT",
    "Apache-2.0",
    "Apache-2.0 WITH LLVM-exception",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-DFS-2016",
]

[bans]
multiple-versions = "warn"
wildcards = "allow"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

## 6. Docker Build and Push

```yaml
# .github/workflows/cd.yml - Docker build and push
name: CD

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  docker:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      # Extract metadata
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ghcr.io/${{ github.repository }}
            ${{ secrets.DOCKERHUB_USERNAME }}/my-api
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=git-
      
      # Login ไปยัง registries
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Login to DockerHub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      # Setup QEMU สำหรับ multi-platform builds
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      # Setup BuildKit
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      # Build and push
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.ref_name }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

## 7. Deployment to Cloud

### 7.1 Deploy to Fly.io

```yaml
# .github/workflows/deploy-fly.yml
  deploy-fly:
    name: Deploy to Fly.io
    runs-on: ubuntu-latest
    needs: [test, clippy, fmt]
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Fly CLI
        uses: superfly/flyctl-actions/setup-flyctl@master
      
      - name: Deploy to Fly.io
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
        run: |
          flyctl deploy \
            --remote-only \
            --strategy rolling
```

### 7.2 Deploy to Railway

```yaml
# .github/workflows/deploy-railway.yml
  deploy-railway:
    name: Deploy to Railway
    runs-on: ubuntu-latest
    needs: [test, docker]
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Railway
        uses: bervProject/railway-deploy@main
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: my-api
```

### 7.3 Deploy via SSH

```yaml
  deploy-ssh:
    name: Deploy via SSH
    runs-on: ubuntu-latest
    needs: [test, docker]
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/my-app
            
            # Pull latest image
            docker pull ghcr.io/${{ github.repository }}:main
            
            # Zero-downtime deployment
            docker-compose pull
            docker-compose up -d --no-deps --scale api=2 api
            sleep 15
            
            # Health check
            if curl -f http://localhost:8080/health; then
              echo "Deployment successful"
              docker-compose up -d --no-deps api
            else
              echo "Health check failed, rolling back"
              docker-compose rollback
              exit 1
            fi
```

## 8. Cache Cargo Dependencies

### 8.1 สวัสดี Swatinem/rust-cache

```yaml
# .github/workflows/ci.yml - cache setup
  - name: Cache Rust dependencies
    uses: Swatinem/rust-cache@v2
    with:
      # Cache key prefix
      prefix-key: "v1-rust"
      # Save cache even on failure
      cache-on-failure: true
      # Cache cargo registry
      cache-directories: |
        ~/.cargo/registry/index
        ~/.cargo/registry/cache
        ~/.cargo/git/db
```

### 8.2 Manual Cache Configuration

```yaml
  - name: Cache cargo registry
    uses: actions/cache@v3
    with:
      path: |
        ~/.cargo/registry/index
        ~/.cargo/registry/cache
        ~/.cargo/git/db
        target/
      key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
      restore-keys: |
        ${{ runner.os }}-cargo-
  
  - name: Cache cargo-audit binary
    uses: actions/cache@v3
    id: cache-cargo-audit
    with:
      path: ~/.cargo/bin/cargo-audit
      key: cargo-audit-${{ runner.os }}
  
  - name: Install cargo-audit
    if: steps.cache-cargo-audit.outputs.cache-hit != 'true'
    run: cargo install cargo-audit
```

### 8.3 sccache สำหรับ Faster Builds

```yaml
  - name: Setup sccache
    uses: mozilla-actions/sccache-action@v0.0.3
  
  - name: Configure sccache
    run: |
      echo "SCCACHE_GHA_ENABLED=true" >> $GITHUB_ENV
      echo "RUSTC_WRAPPER=sccache" >> $GITHUB_ENV
  
  - name: Build with sccache
    run: cargo build --release
  
  - name: Show sccache stats
    run: sccache --show-stats
```

## 9. Practical: Complete CI/CD Pipeline

### 9.1 Complete Workflow File

```yaml
# .github/workflows/pipeline.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    branches: [main, develop]
  release:
    types: [published]

env:
  CARGO_TERM_COLOR: always
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ========== QUALITY CHECKS ==========
  
  fmt:
    name: Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all -- --check
  
  clippy:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --all-targets --all-features -- -D warnings
  
  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install cargo-audit --locked
      - run: cargo audit --deny warnings
  
  # ========== TESTS ==========
  
  test:
    name: Test Suite
    runs-on: ubuntu-latest
    needs: [fmt, clippy]
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: llvm-tools-preview  # สำหรับ coverage
      
      - uses: Swatinem/rust-cache@v2
      
      - name: Install test tools
        run: |
          cargo install sqlx-cli --no-default-features --features postgres
          cargo install cargo-tarpaulin
      
      - name: Run migrations
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
        run: sqlx migrate run
      
      - name: Run tests with coverage
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-ci-secret
        run: |
          cargo tarpaulin \
            --verbose \
            --all-features \
            --workspace \
            --timeout 120 \
            --out xml \
            --output-dir coverage/
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: coverage/cobertura.xml
          fail_ci_if_error: false
  
  # ========== BUILD ==========
  
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      
      - name: Build release binary
        run: cargo build --release
      
      - name: Upload binary artifact
        uses: actions/upload-artifact@v3
        with:
          name: my-api-linux
          path: target/release/my-api
  
  # ========== DOCKER ==========
  
  docker:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # ========== DEPLOY STAGING ==========
  
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            export IMAGE_TAG=${{ needs.docker.outputs.image-tag }}
            cd /opt/myapp-staging
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$IMAGE_TAG
            sed -i "s|IMAGE_TAG=.*|IMAGE_TAG=$IMAGE_TAG|" .env
            docker-compose up -d --no-deps api
            
            # Health check
            sleep 10
            curl -f http://localhost:8080/health || exit 1
  
  # ========== DEPLOY PRODUCTION ==========
  
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: docker
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Fly CLI
        uses: superfly/flyctl-actions/setup-flyctl@master
      
      - name: Deploy to production
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
        run: |
          flyctl deploy \
            --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.docker.outputs.image-digest }} \
            --strategy rolling \
            --wait-timeout 300
      
      - name: Verify deployment
        run: |
          sleep 30
          STATUS=$(curl -sf https://myapp.com/health | jq -r '.status')
          if [ "$STATUS" != "healthy" ]; then
            echo "Health check failed: $STATUS"
            flyctl releases list
            exit 1
          fi
          echo "Deployment successful!"
      
      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          channel-id: 'deployments'
          payload: |
            {
              "text": "✅ Production deployment successful!",
              "attachments": [{
                "color": "good",
                "fields": [
                  {"title": "Version", "value": "${{ github.event.release.tag_name }}", "short": true},
                  {"title": "Deployed by", "value": "${{ github.actor }}", "short": true}
                ]
              }]
            }
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          channel-id: 'deployments'
          payload: |
            {
              "text": "❌ Production deployment failed!",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Version", "value": "${{ github.event.release.tag_name }}", "short": true},
                  {"title": "See logs", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### 9.2 Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build-release:
    name: Build Release (${{ matrix.target }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: x86_64-unknown-linux-gnu
          - os: ubuntu-latest
            target: x86_64-unknown-linux-musl
          - os: windows-latest
            target: x86_64-pc-windows-msvc
          - os: macos-latest
            target: x86_64-apple-darwin
          - os: macos-latest
            target: aarch64-apple-darwin
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
      
      - uses: Swatinem/rust-cache@v2
      
      - name: Build release binary
        run: cargo build --release --target ${{ matrix.target }}
      
      - name: Package binary (Unix)
        if: runner.os != 'Windows'
        run: |
          cd target/${{ matrix.target }}/release
          tar czf ../../../my-api-${{ matrix.target }}.tar.gz my-api
      
      - name: Package binary (Windows)
        if: runner.os == 'Windows'
        run: |
          cd target/${{ matrix.target }}/release
          Compress-Archive -Path my-api.exe -DestinationPath ../../../my-api-${{ matrix.target }}.zip
      
      - name: Upload to release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            my-api-*.tar.gz
            my-api-*.zip
          token: ${{ secrets.GITHUB_TOKEN }}
```

### 9.3 PR Checks Workflow

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches: [main, develop]

jobs:
  # ตรวจสอบ commit message format
  commit-lint:
    name: Commit Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Check commit messages
        uses: wagoid/commitlint-github-action@v5
        with:
          configFile: .commitlintrc.yml
  
  # ตรวจสอบ CHANGELOG
  changelog:
    name: Changelog
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check CHANGELOG updated
        run: |
          if ! git diff --name-only origin/main...HEAD | grep -q CHANGELOG.md; then
            echo "::warning::CHANGELOG.md was not updated"
          fi
  
  # Size check
  binary-size:
    name: Binary Size
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      
      - name: Build and check size
        run: |
          cargo build --release
          SIZE=$(stat -c%s target/release/my-api)
          echo "Binary size: $(numfmt --to=iec $SIZE)"
          
          # Warn if > 50MB
          if [ $SIZE -gt 52428800 ]; then
            echo "::warning::Binary size exceeds 50MB!"
          fi
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **GitHub Actions** - structure และ concepts หลัก
2. **Test automation** - unit tests, integration tests กับ services
3. **Code quality** - clippy, rustfmt
4. **Security** - cargo-audit, cargo-deny
5. **Docker** - build และ push ไปยัง registries
6. **Deployment** - staging และ production deployments
7. **Cache** - ลด CI time ด้วย dependency caching
8. **Release** - multi-platform binary releases

---

[⬅️ Part 073: Docker and Containerization](../part_073/README.md) | [➡️ Part 075: Deployment to Cloud](../part_075/README.md)
