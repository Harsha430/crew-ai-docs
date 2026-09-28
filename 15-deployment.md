# Deployment: Production Readiness

**Taking CrewAI from development to production**

---

## Deployment Checklist

```
Pre-Deployment
├─ [ ] Code review complete
├─ [ ] Tests passing
├─ [ ] Load testing done
├─ [ ] Security audit passed
├─ [ ] Costs estimated
├─ [ ] Monitoring configured
└─ [ ] Runbooks written

Staging Environment
├─ [ ] Deploy to staging
├─ [ ] Run full test suite
├─ [ ] Performance testing
├─ [ ] Security scanning
├─ [ ] Cost validation
└─ [ ] Team sign-off

Production Rollout
├─ [ ] Canary deployment (1% traffic)
├─ [ ] Monitor metrics
├─ [ ] Gradual rollout (10% → 50% → 100%)
├─ [ ] Alert all stakeholders
├─ [ ] Prepare rollback plan
└─ [ ] Post-deployment validation
```

---

## Environment Configuration

### Development Environment

```python
# .env.development
OPENAI_API_KEY=sk-dev-...
ANTHROPIC_API_KEY=sk-dev-...
ENVIRONMENT=development
DEBUG=true
LOG_LEVEL=DEBUG
MAX_RETRIES=1
TIMEOUT_SECONDS=30
```

### Staging Environment

```python
# .env.staging
OPENAI_API_KEY=sk-staging-...
ANTHROPIC_API_KEY=sk-staging-...
ENVIRONMENT=staging
DEBUG=false
LOG_LEVEL=INFO
MAX_RETRIES=3
TIMEOUT_SECONDS=60
ALERT_EMAIL=devops@company.com
```

### Production Environment

```python
# .env.production (from secrets manager)
OPENAI_API_KEY=sk-prod-...
ANTHROPIC_API_KEY=sk-prod-...
ENVIRONMENT=production
DEBUG=false
LOG_LEVEL=WARNING
MAX_RETRIES=5
TIMEOUT_SECONDS=120
ALERT_EMAIL=oncall@company.com
ALERT_SLACK_CHANNEL=#crewai-alerts
```

### Loading Configuration

```python
import os
from dotenv import load_dotenv

def load_config():
    """Load environment-specific config"""
    env = os.getenv("ENVIRONMENT", "development")
    load_dotenv(f".env.{env}")
    
    config = {
        "api_key": os.getenv("OPENAI_API_KEY"),
        "debug": os.getenv("DEBUG", "false").lower() == "true",
        "log_level": os.getenv("LOG_LEVEL", "INFO"),
        "max_retries": int(os.getenv("MAX_RETRIES", "3")),
        "timeout": int(os.getenv("TIMEOUT_SECONDS", "60"))
    }
    
    return config

# Usage
config = load_config()
```

---

## Docker Deployment

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose

```yaml
version: '3.8'

services:
  crewai:
    build: .
    environment:
      ENVIRONMENT: production
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    ports:
      - "8000:8000"
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    
  grafana:
    image: grafana/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
```

---

## Kubernetes Deployment

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crewai-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: crewai-app
  template:
    metadata:
      labels:
        app: crewai-app
    spec:
      containers:
      - name: crewai
        image: company/crewai-app:v1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        env:
        - name: ENVIRONMENT
          value: production
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: crewai-secrets
              key: openai-api-key
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: crewai-secrets
              key: anthropic-api-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: crewai-service
  namespace: production
spec:
  selector:
    app: crewai-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: crewai-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: crewai-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Canary Deployment

```python
# Gradual rollout strategy
ROLLOUT_STAGES = [
    {"percentage": 1, "duration_minutes": 30},   # 1% traffic for 30 min
    {"percentage": 5, "duration_minutes": 30},   # 5% traffic for 30 min
    {"percentage": 25, "duration_minutes": 60},  # 25% traffic for 1 hour
    {"percentage": 50, "duration_minutes": 60},  # 50% traffic for 1 hour
    {"percentage": 100, "duration_minutes": 0}   # 100% traffic
]

def canary_deploy(new_version, stages):
    """Deploy gradually, monitoring at each stage"""
    for stage in stages:
        percentage = stage["percentage"]
        duration = stage["duration_minutes"]
        
        print(f"Routing {percentage}% to {new_version}")
        set_traffic_split(new_version, percentage)
        
        if duration > 0:
            print(f"Monitoring for {duration} minutes...")
            monitor_metrics(duration_minutes=duration)
            
            if should_rollback():
                print(f"Rolling back {new_version}")
                rollback(new_version)
                return False
    
    print("Deployment successful")
    return True
```

---

## Health Checks

### Liveness Probe (Is it running?)

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.get("/health")
def health_check():
    """Liveness probe - check if service is running"""
    return {"status": "alive"}
```

### Readiness Probe (Is it ready for traffic?)

```python
@app.get("/ready")
def readiness_check():
    """Readiness probe - check if ready to handle requests"""
    checks = {
        "database": check_database(),
        "llm_api": check_llm_api(),
        "memory": check_memory(),
        "tools": check_tools_available()
    }
    
    if all(checks.values()):
        return {"status": "ready"}
    else:
        return {"status": "not_ready", "details": checks}, 503
```

---

## Monitoring & Alerting

### Prometheus Alerts

```yaml
groups:
- name: crewai_alerts
  rules:
  - alert: HighErrorRate
    expr: rate(crewai_errors_total[5m]) > 0.05
    for: 5m
    annotations:
      summary: "High error rate: {{ $value }}"
      
  - alert: HighLatency
    expr: histogram_quantile(0.99, crewai_latency) > 10000
    for: 5m
    annotations:
      summary: "High p99 latency: {{ $value }}ms"
      
  - alert: HighCost
    expr: increase(crewai_cost_usd[1h]) > 100
    for: 5m
    annotations:
      summary: "High hourly cost: ${{ $value }}"
      
  - alert: LowAvailability
    expr: up{job="crewai"} == 0
    for: 2m
    annotations:
      summary: "CrewAI service down"
```

---

## Rollback Strategy

```python
def rollback(current_version: str, previous_version: str):
    """Rollback to previous version"""
    
    # Check current metrics
    error_rate = get_error_rate()
    latency = get_p99_latency()
    
    if error_rate > 0.1 or latency > 30000:
        print(f"Issues detected. Rolling back to {previous_version}")
        
        # Update traffic split
        set_traffic_split(current_version, 0)
        set_traffic_split(previous_version, 100)
        
        # Monitor
        sleep(5 * 60)
        
        # Verify rollback
        if get_error_rate() < 0.02:
            print("Rollback successful")
            return True
        else:
            print("Rollback failed")
            return False
```

---

## Secrets Management

### AWS Secrets Manager

```python
import boto3
import json

def get_secret(secret_name: str, region_name: str = "us-east-1"):
    """Get secret from AWS Secrets Manager"""
    
    client = boto3.client('secretsmanager', region_name=region_name)
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        if 'SecretString' in response:
            return json.loads(response['SecretString'])
        else:
            return response['SecretBinary']
    except Exception as e:
        raise Exception(f"Failed to get secret: {e}")

# Usage
secrets = get_secret("crewai/production/secrets")
openai_key = secrets["openai_api_key"]
```

---

## Cost Monitoring

### Track Deployment Costs

```python
def track_deployment_cost(version: str, duration_hours: float):
    """Track cost for deployed version"""
    
    total_tokens = get_total_tokens(version)
    
    # Pricing (example)
    input_cost = (total_tokens["input"] / 1_000_000) * 3
    output_cost = (total_tokens["output"] / 1_000_000) * 15
    total_cost = input_cost + output_cost
    
    cost_per_hour = total_cost / duration_hours
    
    log_metric("deployment_cost", {
        "version": version,
        "total_cost": total_cost,
        "cost_per_hour": cost_per_hour,
        "tokens": total_tokens
    })
    
    return cost_per_hour
```

---

## Production Runbook

```markdown
# CrewAI Production Runbook

## Startup Procedure
1. Verify environment variables configured
2. Run health check
3. Monitor error rate for 5 minutes
4. If errors > 2%, investigate before proceeding

## Incident Response
1. Check monitoring dashboard
2. Review recent deployments
3. Check external service status
4. If LLM API down: route to fallback model
5. If critical: page oncall engineer
6. Prepare rollback if needed

## Rollback Procedure
1. Identify version to rollback to
2. Set traffic split to previous version
3. Monitor metrics for 5 minutes
4. If successful: keep rollback
5. If failed: escalate to engineering team

## Scaling
1. Monitor CPU/memory usage
2. If > 80% utilization: trigger autoscaling
3. Set HPA min_replicas to handle peak load
4. Monitor cost increases

## Monitoring
- Dashboard: https://grafana.company.com/crewai
- Alerts: #crewai-alerts Slack channel
- Logs: CloudWatch/ELK dashboard
```

---

## Deployment Checklist Summary

✅ **DO:**
- Use environment-specific configs
- Test in staging before production
- Implement canary deployment
- Monitor metrics during rollout
- Have rollback plan ready
- Use health checks (liveness + readiness)
- Store secrets securely
- Document runbooks
- Alert on anomalies

❌ **DON'T:**
- Deploy directly to production (staging first!)
- Use hardcoded credentials
- Skip load testing
- Deploy at peak hours (unless necessary)
- Ignore monitoring alerts
- Forget rollback plan
- Disable health checks
- Deploy without team notification

---

**Next**: [Evaluation](./16-evaluation.md)
