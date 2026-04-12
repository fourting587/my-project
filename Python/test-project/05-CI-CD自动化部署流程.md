# CI/CD自动化部署流程

## 文档信息

- **版本**: v1.0
- **日期**: 2026-04-12
- **CI工具**: GitHub Actions
- **部署方式**: Docker Compose + SSH

---

## 目录

1. [架构设计](#1-架构设计)
2. [GitHub Actions配置](#2-github-actions配置)
3. [自动化脚本](#3-自动化脚本)
4. [部署流程](#4-部署流程)
5. [环境管理](#5-环境管理)
6. [监控告警](#6-监控告警)

---

## 1. 架构设计

### 1.1 CI/CD流程图

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Developer │───▶│   Git Push  │───▶│   GitHub    │───▶│   Build &   │
│   Commit    │    │   to main   │    │   Actions   │    │   Test      │
└─────────────┘    └─────────────┘    └─────────────┘    └──────┬──────┘
                                                                 │
                    ┌─────────────────────────────────────────────┘
                    │
                    ▼
         ┌─────────────────┐
         │   Test Results  │
         │   Pass?         │
         └────────┬────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
       Yes                  No
        │                   │
        ▼                   ▼
┌──────────────┐   ┌────────────────┐
│   Deploy to  │   │   Send Alert   │
│   Production │   │   & Rollback   │
└──────┬───────┘   └────────────────┘
       │
       ▼
┌──────────────┐
│   Health     │
│   Check      │
└──────┬───────┘
       │
      Yes
       │
       ▼
┌──────────────┐
│   Notify     │
│   Success    │
└──────────────┘
```

### 1.2 部署策略

| 环境 | 触发条件 | 部署策略 | 回滚时间 |
|------|----------|----------|----------|
| 开发(dev) | push到dev分支 | 自动部署 | 即时 |
| 测试(test) | push到test分支 | 自动部署 | 即时 |
| 预发(staging) | 创建Release | 自动部署 | 5分钟内 |
| 生产(prod) | 手动触发 | 蓝绿部署 | 2分钟内 |

---

## 2. GitHub Actions配置

### 2.1 主工作流配置

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, dev, test]
    tags: ['v*']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: '部署环境'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      version:
        description: '版本号'
        required: false
        type: string

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================
  # 代码检查与测试
  # ============================================
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov flake8 black isort

      - name: Lint with flake8
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Check code formatting
        run: |
          black --check .
          isort --check-only .

      - name: Run tests with pytest
        run: |
          pytest --cov=app --cov-report=xml --cov-report=term

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          fail_ci_if_error: false

  # ============================================
  # 构建Docker镜像
  # ============================================
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      version: ${{ steps.version.outputs.value }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
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
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=,suffix=,format=short

      - name: Generate version
        id: version
        run: |
          if [ "${{ github.ref_type }}" = "tag" ]; then
            echo "value=${{ github.ref_name }}" >> $GITHUB_OUTPUT
          else
            echo "value=sha-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          fi

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}
            VERSION=${{ steps.version.outputs.value }}

  # ============================================
  # 部署到开发环境
  # ============================================
  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/dev'
    runs-on: ubuntu-latest
    environment:
      name: development
      url: http://dev.example.com
    steps:
      - name: Deploy to development
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USER }}
          key: ${{ secrets.DEV_SSH_KEY }}
          port: ${{ secrets.DEV_PORT }}
          script: |
            cd /data/app
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker pull ${{ needs.build.outputs.image_tag }}
            
            # 更新docker-compose配置
            export IMAGE_TAG=${{ needs.build.outputs.image_tag }}
            docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d app
            
            # 执行数据库迁移
            docker-compose exec -T app alembic upgrade head
            
            # 清理旧镜像
            docker image prune -f

  # ============================================
  # 部署到测试环境
  # ============================================
  deploy-test:
    needs: build
    if: github.ref == 'refs/heads/test'
    runs-on: ubuntu-latest
    environment:
      name: testing
      url: http://test.example.com
    steps:
      - name: Deploy to testing
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.TEST_HOST }}
          username: ${{ secrets.TEST_USER }}
          key: ${{ secrets.TEST_SSH_KEY }}
          script: |
            cd /data/app
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker pull ${{ needs.build.outputs.image_tag }}
            
            export IMAGE_TAG=${{ needs.build.outputs.image_tag }}
            docker-compose -f docker-compose.yml -f docker-compose.test.yml up -d
            
            # 运行集成测试
            docker-compose exec -T app pytest tests/integration/ -v

  # ============================================
  # 部署到预发环境
  # ============================================
  deploy-staging:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v') || github.event.inputs.environment == 'staging'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: http://staging.example.com
    steps:
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /data/app
            
            # 备份当前版本
            docker tag $(docker-compose images -q app) app:backup-$(date +%Y%m%d-%H%M%S)
            
            # 拉取新镜像
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker pull ${{ needs.build.outputs.image_tag }}
            
            # 滚动更新
            export IMAGE_TAG=${{ needs.build.outputs.image_tag }}
            docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d --no-deps app
            
            # 健康检查
            sleep 10
            curl -f http://localhost:8000/health || exit 1
            
            # 数据库迁移
            docker-compose exec -T app alembic upgrade head

      - name: Run smoke tests
        run: |
          sleep 30
          curl -f http://${{ secrets.STAGING_HOST }}:8000/health
          curl -f http://${{ secrets.STAGING_HOST }}:8000/api/docs

  # ============================================
  # 部署到生产环境
  # ============================================
  deploy-production:
    needs: [build, deploy-staging]
    if: (startsWith(github.ref, 'refs/tags/v') && !contains(github.ref, '-rc')) || github.event.inputs.environment == 'production'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: http://example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PROD_SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          cat >> ~/.ssh/config << EOF
          Host prod
            HostName ${{ secrets.PROD_HOST }}
            User ${{ secrets.PROD_USER }}
            Port ${{ secrets.PROD_PORT }}
            IdentityFile ~/.ssh/deploy_key
            StrictHostKeyChecking no
          EOF

      - name: Blue-Green Deployment
        run: |
          # 执行蓝绿部署脚本
          ssh prod 'bash -s' < scripts/deploy-blue-green.sh ${{ needs.build.outputs.image_tag }}

      - name: Health check
        run: |
          sleep 30
          for i in {1..10}; do
            if curl -f http://${{ secrets.PROD_HOST }}/health; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i failed, retrying..."
            sleep 10
          done
          echo "Health check failed"
          exit 1

      - name: Notify success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "✅ 生产环境部署成功",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*生产环境部署成功*\n版本: ${{ needs.build.outputs.version }}\n部署人: ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Rollback on failure
        if: failure()
        run: |
          ssh prod 'bash -s' < scripts/rollback.sh
          
      - name: Notify failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "❌ 生产环境部署失败，已自动回滚",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*生产环境部署失败*\n版本: ${{ needs.build.outputs.version }}\n已自动回滚到上一版本"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 2.2 Secrets配置

```yaml
# GitHub Repository Secrets配置

# Docker Registry
CR_PAT                    # Container Registry Personal Access Token

# 开发环境
DEV_HOST                  # 开发服务器IP/域名
DEV_USER                  # SSH用户名
DEV_SSH_KEY               # SSH私钥
DEV_PORT                  # SSH端口(默认22)

# 测试环境
TEST_HOST
TEST_USER
TEST_SSH_KEY

# 预发环境
STAGING_HOST
STAGING_USER
STAGING_SSH_KEY

# 生产环境
PROD_HOST
PROD_USER
PROD_SSH_KEY
PROD_PORT

# 通知
SLACK_WEBHOOK_URL         # Slack通知地址

# 其他
CODECOV_TOKEN             # Codecov Token
```

---

## 3. 自动化脚本

### 3.1 蓝绿部署脚本

```bash
#!/bin/bash
# scripts/deploy-blue-green.sh

set -e

IMAGE_TAG=$1
ENV_FILE="/data/app/.env"
COMPOSE_FILE="/data/app/docker-compose.yml"

# 颜色定义
GREEN='\033[0;32m'
BLUE='\033[0;34m'
RED='\033[0;31m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1"
    exit 1
}

cd /data/app

# 确定当前运行的颜色
CURRENT_COLOR=$(docker-compose ps app | grep -oP 'app-\K(blue|green)' || echo "blue")
NEW_COLOR=$([ "$CURRENT_COLOR" = "blue" ] && echo "green" || echo "blue")

log "当前运行: $CURRENT_COLOR, 部署到: $NEW_COLOR"

# 备份当前配置
cp $COMPOSE_FILE ${COMPOSE_FILE}.backup.$(date +%Y%m%d-%H%M%S)

# 更新镜像标签
log "更新镜像标签为: $IMAGE_TAG"
sed -i "s|image: .*app:.*|image: $IMAGE_TAG|g" docker-compose.${NEW_COLOR}.yml

# 启动新环境
log "启动 $NEW_COLOR 环境..."
docker-compose -f docker-compose.yml -f docker-compose.${NEW_COLOR}.yml up -d app-${NEW_COLOR}

# 等待服务就绪
log "等待服务就绪..."
sleep 10

# 健康检查
log "执行健康检查..."
for i in {1..30}; do
    if curl -sf http://localhost:8001/health >/devdev/null 2>&1; then
        log "健康检查通过"
        break
    fi
    if [ $i -eq 30 ]; then
        error "健康检查失败"
    fi
    sleep 2
done

# 执行数据库迁移
log "执行数据库迁移..."
docker-compose exec -T app-${NEW_COLOR} alembic upgrade head

# 切换流量(Nginx配置)
log "切换流量到 $NEW_COLOR..."
sed -i "s/upstream_app_color: $CURRENT_COLOR/upstream_app_color: $NEW_COLOR/g" /data/nginx/conf.d/app.conf
nginx -s reload

# 停止旧环境
log "停止 $CURRENT_COLOR 环境..."
docker-compose stop app-${CURRENT_COLOR}

# 保留旧版本24小时便于回滚
(
    sleep 86400
    docker-compose rm -f app-${CURRENT_COLOR}
) &

log "部署完成! 当前运行: $NEW_COLOR"
```

### 3.2 回滚脚本

```bash
#!/bin/bash
# scripts/rollback.sh

set -e

ENV_FILE="/data/app/.env"
COMPOSE_FILE="/data/app/docker-compose.yml"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

cd /data/app

# 确定当前运行的颜色
CURRENT_COLOR=$(docker-compose ps app | grep -oP 'app-\K(blue|green)' || echo "blue")
OLD_COLOR=$([ "$CURRENT_COLOR" = "blue" ] && echo "green" || echo "blue")

log "开始回滚: 从 $CURRENT_COLOR 回滚到 $OLD_COLOR"

# 检查旧容器是否存在
if ! docker-compose ps | grep -q "app-${OLD_COLOR}"; then
    log "错误: 旧版本容器不存在，无法回滚"
    exit 1
fi

# 启动旧环境
docker-compose start app-${OLD_COLOR}

# 等待就绪
sleep 10

# 健康检查
for i in {1..10}; do
    if curl -sf http://localhost:8001/health >/devdev/null 2>&1; then
        log "旧版本健康检查通过"
        break
    fi
    sleep 2
done

# 切换流量回旧版本
sed -i "s/upstream_app_color: $CURRENT_COLOR/upstream_app_color: $OLD_COLOR/g" /data/nginx/conf.d/app.conf
nginx -s reload

# 停止新版本
docker-compose stop app-${CURRENT_COLOR}

log "回滚完成! 当前运行: $OLD_COLOR"
```

### 3.3 数据库迁移脚本

```bash
#!/bin/bash
# scripts/migrate.sh

set -e

ENV=${1:-production}
COMMAND=${2:-upgrade}
REVISION=${3:-head}

case $ENV in
    dev)
        COMPOSE_FILE="docker-compose.yml -f docker-compose.dev.yml"
        ;;
    test)
        COMPOSE_FILE="docker-compose.yml -f docker-compose.test.yml"
        ;;
    staging)
        COMPOSE_FILE="docker-compose.yml -f docker-compose.staging.yml"
        ;;
    production)
        COMPOSE_FILE="docker-compose.yml -f docker-compose.prod.yml"
        ;;
    *)
        echo "未知环境: $ENV"
        exit 1
        ;;
esac

if [ "$COMMAND" = "upgrade" ]; then
    echo "执行数据库升级: $REVISION"
    docker-compose -f $COMPOSE_FILE exec -T app alembic upgrade $REVISION
elif [ "$COMMAND" = "downgrade" ]; then
    echo "执行数据库降级: $REVISION"
    docker-compose -f $COMPOSE_FILE exec -T app alembic downgrade $REVISION
elif [ "$COMMAND" = "revision" ]; then
    echo "创建新的迁移版本"
    docker-compose -f $COMPOSE_FILE exec -T app alembic revision --autogenerate -m "$REVISION"
else
    echo "用法: $0 [dev|test|staging|production] [upgrade|downgrade|revision] [revision|message]"
    exit 1
fi
```

### 3.4 备份脚本

```bash
#!/bin/bash
# scripts/backup.sh

set -e

BACKUP_DIR="/data/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

mkdir -p $BACKUP_DIR

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# 备份MySQL
backup_mysql() {
    log "开始备份MySQL..."
    docker exec mysql mysqldump -u root -p"${MYSQL_ROOT_PASSWORD}" \
        --all-databases --single-transaction --quick \
        | gzip > $BACKUP_DIR/mysql_${DATE}.sql.gz
    log "MySQL备份完成: mysql_${DATE}.sql.gz"
}

# 备份Redis
backup_redis() {
    log "开始备份Redis..."
    docker exec redis redis-cli BGSAVE
    sleep 5
    cp /data/redis/data/dump.rdb $BACKUP_DIR/redis_${DATE}.rdb
    log "Redis备份完成: redis_${DATE}.rdb"
}

# 备份应用数据
backup_app() {
    log "开始备份应用数据..."
    tar -czf $BACKUP_DIR/app_data_${DATE}.tar.gz -C /data app/logs app/uploads
    log "应用数据备份完成: app_data_${DATE}.tar.gz"
}

# 清理旧备份
cleanup_old_backups() {
    log "清理${RETENTION_DAYS}天前的备份..."
    find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete
    find $BACKUP_DIR -name "*.rdb" -mtime +$RETENTION_DAYS -delete
    log "清理完成"
}

# 主流程
main() {
    log "===== 开始备份 ====="
    backup_mysql
    backup_redis
    backup_app
    cleanup_old_backups
    log "===== 备份完成 ====="
    
    # 输出备份列表
    ls -lh $BACKUP_DIR
}

main
```

---

## 4. 部署流程

### 4.1 开发部署流程

```bash
# 本地开发环境启动
make dev

# 运行测试
make test

# 代码提交
git add .
git commit -m "feat: add new feature"
git push origin dev

# 自动触发CI/CD部署到开发环境
```

### 4.2 生产部署流程

```bash
# 1. 创建Release
gh release create v1.2.0 --title "v1.2.0" --notes "Release notes"

# 2. 自动触发预发环境部署
# GitHub Actions自动执行

# 3. 预发验证通过后，手动触发生产部署
# 在GitHub Actions页面点击"Run workflow"
# 选择 environment: production

# 4. 监控部署状态
# 查看GitHub Actions日志
# 查看Slack通知
```

### 4.3 Makefile

```makefile
# Makefile
.PHONY: dev test build deploy clean

# 开发环境
dev:
	docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# 运行测试
test:
	pytest tests/ -v --cov=app --cov-report=term-missing

# 构建镜像
build:
	docker build -t data-platform:latest .

# 代码格式化
format:
	black .
	isort .

# 代码检查
lint:
	flake8 .
	black --check .
	isort --check-only .

# 运行迁移
migrate:
	docker-compose exec app alembic upgrade head

# 创建迁移
migrate-create:
	docker-compose exec app alembic revision --autogenerate -m "$(message)"

# 备份
backup:
	./scripts/backup.sh

# 生产部署
deploy-prod:
	./scripts/deploy-blue-green.sh ghcr.io/username/data-platform:latest

# 回滚
rollback:
	./scripts/rollback.sh

# 清理
clean:
	docker-compose down -v
	docker system prune -f
```

---

## 5. 环境管理

### 5.1 环境变量配置

```bash
# .env.production
APP_ENV=production
DEBUG=false
LOG_LEVEL=WARNING

# 数据库
DATABASE_URL=mysql+pymysql://user:pass@mysql:3306/db
DATABASE_POOL_SIZE=20

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_PASSWORD=${REDIS_PASSWORD}

# Elasticsearch
ES_HOSTS=http://elasticsearch:9200

# 安全配置
SECRET_KEY=${SECRET_KEY}
JWT_EXPIRE_HOURS=24

# 外部服务
SENTRY_DSN=${SENTRY_DSN}
```

### 5.2 环境覆盖配置

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    environment:
      - APP_ENV=production
      - WORKERS=4
      - LOG_LEVEL=WARNING
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: always

  nginx:
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/ssl:/etc/nginx/ssl:ro
```

---

## 6. 监控告警

### 6.1 部署监控

```yaml
# .github/workflows/monitor.yml
name: Deployment Monitor

on:
  schedule:
    - cron: '*/5 * * * *'  # 每5分钟检查一次

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check production health
        run: |
          response=$(curl -s -o /dev/null -w "%{http_code}" https://example.com/health)
          if [ "$response" != "200" ]; then
            echo "::error::生产环境健康检查失败 (HTTP $response)"
            exit 1
          fi
          
      - name: Check response time
        run: |
          time=$(curl -s -o /dev/null -w "%{time_total}" https://example.com/api/health)
          if (( $(echo "$time > 2.0" | bc -l) )); then
            echo "::warning::响应时间过长: ${time}s"
          fi

  notify:
    needs: health-check
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Send alert
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "🚨 生产环境健康检查失败",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*生产环境异常*\n请立即检查系统状态"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 6.2 告警规则

```yaml
# prometheus/alert_rules.yml
groups:
  - name: deployment
    rules:
      - alert: DeploymentFailed
        expr: |
          time() - github_actions_workflow_run_timestamp{status="failure"} < 300
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "部署失败"
          
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) / 
          rate(http_requests_total[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "错误率过高"
          description: "5xx错误率超过10%"
```
