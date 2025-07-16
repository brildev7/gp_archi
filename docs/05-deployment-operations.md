# 배포 및 운영 가이드

## 📋 개요

이 문서는 Azure 클라우드 환경에서 AI 교육 플랫폼을 배포하고 운영하는 방법을 상세히 설명합니다. Infrastructure as Code (IaC) 접근방식을 통해 일관되고 재현 가능한 배포를 제공합니다.

## 🚀 배포 환경 구성

### 환경별 배포 전략

#### 개발 환경 (Development)
```yaml
목적: 개발 및 초기 테스트
리소스 규모: 최소
비용 최적화: 우선
특징:
  - 단일 AZ 배포
  - 기본 SKU 사용
  - 개발용 데이터베이스
  - 최소 복제본 수
```

#### 스테이징 환경 (Staging)
```yaml
목적: 프로덕션 배포 전 검증
리소스 규모: 중간
특징:
  - 프로덕션과 유사한 구성
  - 실제 데이터 유사 테스트 데이터
  - 성능 및 부하 테스트
  - 자동화된 테스트 실행
```

#### 프로덕션 환경 (Production)
```yaml
목적: 실제 서비스 운영
리소스 규모: 최대
고가용성: 우선
특징:
  - 다중 AZ 배포
  - 프리미엄 SKU 사용
  - 백업 및 재해복구
  - 모니터링 및 알림
```

## 🛠️ 배포 프로세스

### 1. 사전 준비

#### Azure CLI 설정
```bash
# Azure CLI 로그인
az login

# 구독 설정
az account set --subscription "your-subscription-id"

# 리소스 그룹 생성
az group create \
  --name "rg-ai-education-platform-prod" \
  --location "Korea Central"
```

#### 환경 변수 설정
```bash
# .env.production 파일 생성
cat > .env.production << EOF
# Azure 구독 정보
AZURE_SUBSCRIPTION_ID=your-subscription-id
AZURE_TENANT_ID=your-tenant-id
AZURE_CLIENT_ID=your-client-id
AZURE_CLIENT_SECRET=your-client-secret

# OpenAI API
OPENAI_API_KEY=your-openai-api-key
OPENAI_ORG_ID=your-organization-id

# 환경 설정
ENVIRONMENT=production
AZURE_LOCATION="Korea Central"
PROJECT_NAME=ai-education-platform
EOF
```

### 2. Infrastructure as Code 배포

#### Terraform 초기화 및 계획
```bash
# Terraform 초기화
cd terraform
terraform init \
  -backend-config="resource_group_name=rg-terraform-state" \
  -backend-config="storage_account_name=saterraformstate" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=prod.terraform.tfstate"

# 배포 계획 생성
terraform plan \
  -var="environment=prod" \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -out=tfplan-prod

# 계획 검토 및 승인 후 적용
terraform apply tfplan-prod
```

#### Azure Resource Manager (ARM) 템플릿 배포 (대안)
```bash
# ARM 템플릿 검증
az deployment group validate \
  --resource-group "rg-ai-education-platform-prod" \
  --template-file "infrastructure/arm/main.json" \
  --parameters "infrastructure/arm/parameters.prod.json"

# ARM 템플릿 배포
az deployment group create \
  --resource-group "rg-ai-education-platform-prod" \
  --template-file "infrastructure/arm/main.json" \
  --parameters "infrastructure/arm/parameters.prod.json" \
  --name "ai-education-platform-deployment"
```

### 3. 애플리케이션 배포

#### Docker 이미지 빌드 및 푸시
```bash
# Azure Container Registry에 로그인
az acr login --name acraieducationplatformprod

# 각 서비스별 이미지 빌드
services=("lesson-plan-ai" "content-gen-ai" "assessment-ai" "teaching-agent" "streaming-ai" "gateway")

for service in "${services[@]}"; do
  echo "Building $service..."
  docker build -t acraieducationplatformprod.azurecr.io/$service:latest \
    -f ai-backend/$service/Dockerfile \
    ai-backend/$service/
  
  echo "Pushing $service..."
  docker push acraieducationplatformprod.azurecr.io/$service:latest
done
```

#### Kubernetes 배포 (선택사항)
```bash
# AKS 클러스터 연결
az aks get-credentials \
  --resource-group "rg-ai-education-platform-prod" \
  --name "aks-ai-education-platform-prod"

# 네임스페이스 생성
kubectl create namespace ai-education-platform

# ConfigMap 및 Secret 생성
kubectl create configmap app-config \
  --from-env-file=.env.production \
  -n ai-education-platform

kubectl create secret generic app-secrets \
  --from-literal=openai-api-key=$OPENAI_API_KEY \
  --from-literal=azure-sql-connection=$AZURE_SQL_CONNECTION \
  -n ai-education-platform

# 애플리케이션 배포
kubectl apply -f kubernetes/manifests/ -n ai-education-platform
```

#### Container Instances 배포
```bash
# AI 서비스별 Container Instance 생성
for service in "${services[@]}"; do
  az container create \
    --resource-group "rg-ai-education-platform-prod" \
    --name "ci-$service-prod" \
    --image "acraieducationplatformprod.azurecr.io/$service:latest" \
    --cpu 2 \
    --memory 4 \
    --registry-login-server "acraieducationplatformprod.azurecr.io" \
    --registry-username $(az acr credential show --name acraieducationplatformprod --query username -o tsv) \
    --registry-password $(az acr credential show --name acraieducationplatformprod --query passwords[0].value -o tsv) \
    --environment-variables \
      ENVIRONMENT=production \
      REDIS_URL=$AZURE_REDIS_CONNECTION \
      POSTGRES_URL=$AZURE_SQL_CONNECTION \
      OPENAI_API_KEY=$OPENAI_API_KEY \
    --ports 8000 \
    --restart-policy Always
done
```

## 📊 모니터링 및 관찰성

### 1. Azure Monitor 설정

#### Application Insights 구성
```bash
# Application Insights 구성 요소 생성
az monitor app-insights component create \
  --app "ai-education-platform-insights" \
  --location "Korea Central" \
  --resource-group "rg-ai-education-platform-prod" \
  --application-type web

# 계측 키 가져오기
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app "ai-education-platform-insights" \
  --resource-group "rg-ai-education-platform-prod" \
  --query instrumentationKey -o tsv)
```

#### 로그 분석 워크스페이스 구성
```bash
# Log Analytics 워크스페이스 생성
az monitor log-analytics workspace create \
  --workspace-name "law-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --location "Korea Central"

# 진단 설정 구성
az monitor diagnostic-settings create \
  --name "diagnostic-settings" \
  --resource "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/rg-ai-education-platform-prod/providers/Microsoft.ContainerInstance/containerGroups/ci-gateway-prod" \
  --workspace "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/rg-ai-education-platform-prod/providers/Microsoft.OperationalInsights/workspaces/law-ai-education-platform-prod" \
  --logs '[{"category":"ContainerInstanceLog","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

### 2. 커스텀 대시보드 구성

#### Grafana 대시보드 설정
```yaml
# grafana-dashboard.json
{
  "dashboard": {
    "title": "AI Education Platform Monitoring",
    "panels": [
      {
        "title": "API Response Times",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "AI Service Utilization",
        "type": "graph",
        "targets": [
          {
            "expr": "openai_api_calls_total",
            "legendFormat": "OpenAI API Calls"
          }
        ]
      },
      {
        "title": "Active Sessions",
        "type": "stat",
        "targets": [
          {
            "expr": "signalr_connections_total",
            "legendFormat": "Active Connections"
          }
        ]
      }
    ]
  }
}
```

### 3. 알림 규칙 설정

#### Azure Monitor 알림
```bash
# 액션 그룹 생성
az monitor action-group create \
  --name "ai-education-platform-alerts" \
  --resource-group "rg-ai-education-platform-prod" \
  --action email admin alerts@yourdomain.com

# CPU 사용률 알림 규칙
az monitor metrics alert create \
  --name "high-cpu-usage" \
  --resource-group "rg-ai-education-platform-prod" \
  --scopes "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/rg-ai-education-platform-prod/providers/Microsoft.ContainerInstance/containerGroups/ci-gateway-prod" \
  --condition "avg Percentage CPU > 80" \
  --description "High CPU usage detected" \
  --action "ai-education-platform-alerts"

# 메모리 사용률 알림 규칙
az monitor metrics alert create \
  --name "high-memory-usage" \
  --resource-group "rg-ai-education-platform-prod" \
  --scopes "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/rg-ai-education-platform-prod/providers/Microsoft.ContainerInstance/containerGroups/ci-gateway-prod" \
  --condition "avg Memory Usage > 85" \
  --description "High memory usage detected" \
  --action "ai-education-platform-alerts"
```

## 🔒 보안 운영

### 1. Key Vault 관리

#### 시크릿 관리
```bash
# Key Vault에 시크릿 저장
az keyvault secret set \
  --vault-name "kv-ai-education-platform-prod" \
  --name "openai-api-key" \
  --value "$OPENAI_API_KEY"

az keyvault secret set \
  --vault-name "kv-ai-education-platform-prod" \
  --name "azure-sql-connection" \
  --value "$AZURE_SQL_CONNECTION"

# 시크릿 로테이션 스케줄 설정
az keyvault secret set-attributes \
  --vault-name "kv-ai-education-platform-prod" \
  --name "openai-api-key" \
  --expires "2024-12-31T23:59:59Z"
```

### 2. 네트워크 보안

#### NSG 규칙 구성
```bash
# 네트워크 보안 그룹 생성
az network nsg create \
  --name "nsg-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod"

# HTTP/HTTPS 트래픽 허용
az network nsg rule create \
  --resource-group "rg-ai-education-platform-prod" \
  --nsg-name "nsg-ai-education-platform-prod" \
  --name "AllowHTTP" \
  --protocol Tcp \
  --priority 100 \
  --destination-port-range 80 \
  --access Allow

az network nsg rule create \
  --resource-group "rg-ai-education-platform-prod" \
  --nsg-name "nsg-ai-education-platform-prod" \
  --name "AllowHTTPS" \
  --protocol Tcp \
  --priority 101 \
  --destination-port-range 443 \
  --access Allow
```

## 🔄 백업 및 재해복구

### 1. 데이터베이스 백업

#### Azure SQL Database 백업
```bash
# 자동 백업 정책 확인
az sql db show \
  --name "sqldb-education-prod" \
  --server "sql-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --query '{name:name, earliestRestoreDate:earliestRestoreDate, defaultSecondaryLocation:defaultSecondaryLocation}'

# 수동 백업 (장기 보존)
az sql db ltr-backup list \
  --location "Korea Central" \
  --server "sql-ai-education-platform-prod"

# 백업에서 복원
az sql db restore \
  --dest-name "sqldb-education-restored" \
  --server "sql-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --source-database "sqldb-education-prod" \
  --time "2024-01-01T00:00:00Z"
```

#### Cosmos DB 백업
```bash
# 백업 정책 확인
az cosmosdb show \
  --name "cosmos-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --query 'backupPolicy'

# 연속 백업 모드 활성화
az cosmosdb update \
  --name "cosmos-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --backup-policy-type Continuous
```

### 2. 애플리케이션 백업

#### Container Registry 이미지 백업
```bash
# 이미지 목록 백업
az acr repository list \
  --name "acraieducationplatformprod" > backup/container-images-$(date +%Y%m%d).txt

# 이미지 내보내기
az acr import \
  --name "acraieducationplatformprod" \
  --source "acraieducationplatformprod.azurecr.io/gateway:latest" \
  --target-registry "acrbackupeducationplatform" \
  --target-name "gateway:backup-$(date +%Y%m%d)"
```

## 📈 성능 최적화

### 1. 자동 스케일링 구성

#### Container Instances 스케일링
```yaml
# scaling-policy.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-service-deployment
  minReplicas: 2
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
```

### 2. 캐싱 최적화

#### Redis 클러스터 구성
```bash
# Redis Cache 스케일링
az redis update \
  --name "redis-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --sku Premium \
  --vm-size P3

# Redis 클러스터 활성화
az redis patch-schedule set \
  --name "redis-ai-education-platform-prod" \
  --resource-group "rg-ai-education-platform-prod" \
  --schedule-entries '[{"dayOfWeek":"Sunday","startHourUtc":2,"maintenanceWindow":"PT5H"}]'
```

## 🔧 운영 자동화

### 1. 배포 자동화

#### GitHub Actions 워크플로우
```yaml
# .github/workflows/production-deploy.yml
name: Production Deployment

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Build and Deploy Infrastructure
      run: |
        cd terraform
        terraform init
        terraform plan -out=tfplan
        terraform apply tfplan
    
    - name: Build and Push Images
      run: |
        az acr login --name acraieducationplatformprod
        docker-compose -f docker-compose.prod.yml build
        docker-compose -f docker-compose.prod.yml push
    
    - name: Deploy Services
      run: |
        az container restart --resource-group rg-ai-education-platform-prod --name ci-gateway-prod
        az container restart --resource-group rg-ai-education-platform-prod --name ci-lesson-plan-ai-prod
```

### 2. 모니터링 자동화

#### 헬스 체크 스크립트
```bash
#!/bin/bash
# health-check.sh

SERVICES=("gateway" "lesson-plan-ai" "content-gen-ai" "assessment-ai" "teaching-agent" "streaming-ai")
RESOURCE_GROUP="rg-ai-education-platform-prod"

echo "🔍 Starting health check at $(date)"

for service in "${SERVICES[@]}"; do
  echo "Checking $service..."
  
  # Container Instance 상태 확인
  status=$(az container show \
    --resource-group "$RESOURCE_GROUP" \
    --name "ci-$service-prod" \
    --query "instanceView.state" -o tsv)
  
  if [ "$status" != "Running" ]; then
    echo "❌ $service is not running (Status: $status)"
    
    # Slack 알림 전송
    curl -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"🚨 Service Alert: $service is not running (Status: $status)\"}" \
      "$SLACK_WEBHOOK_URL"
    
    # 서비스 재시작 시도
    echo "🔄 Attempting to restart $service..."
    az container restart \
      --resource-group "$RESOURCE_GROUP" \
      --name "ci-$service-prod"
  else
    echo "✅ $service is healthy"
  fi
done

echo "🏁 Health check completed at $(date)"
```

### 3. 로그 수집 및 분석

#### ELK 스택 구성 (선택사항)
```yaml
# docker-compose.monitoring.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    ports:
      - "5044:5044"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
```

## 📋 운영 체크리스트

### 일일 운영 체크리스트
- [ ] 시스템 헬스 체크 실행
- [ ] 에러 로그 검토
- [ ] 성능 메트릭 확인
- [ ] 백업 상태 점검
- [ ] 보안 알림 검토

### 주간 운영 체크리스트
- [ ] 리소스 사용량 분석
- [ ] 비용 최적화 검토
- [ ] 보안 패치 적용
- [ ] 데이터베이스 성능 튜닝
- [ ] 사용자 피드백 검토

### 월간 운영 체크리스트
- [ ] 재해복구 테스트
- [ ] 보안 감사 실행
- [ ] 성능 벤치마크 테스트
- [ ] 아키텍처 개선점 검토
- [ ] 비용 분석 및 예산 계획

이 가이드를 통해 Azure 환경에서 AI 교육 플랫폼을 안정적으로 배포하고 운영할 수 있습니다. 