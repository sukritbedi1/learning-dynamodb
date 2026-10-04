# 1. Setup & Architecture

## Local Environment: Ministack

Running local DynamoDB via **ministackorg/ministack** — AWS service emulator.

### Docker Compose Setup

**File:** `~/Documents/tools/ministack/docker-compose.yml`

```yaml
version: '3.8'

services:
  ministack:
    image: ministackorg/ministack:latest
    container_name: ministack
    ports:
      - "4566:4566"      # Main AWS API endpoint
      - "8000:8000"      # Legacy DynamoDB endpoint
    volumes:
      - ./data:/data     # Shared data mount
    environment:
      - SERVICES=dynamodb       # Only DynamoDB (not S3, Lambda, etc.)
      - DEBUG=0
      - DATA_DIR=/data/dynamodb # DynamoDB data persistence
      - DYNAMODB_SHARE_DB=1     # Shared DB for dev
    networks:
      - ministack-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/health || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

networks:
  ministack-net:
    driver: bridge
```

**Commands:**
```bash
# Start
cd ~/Documents/tools/ministack
source ~/.gvm/scripts/gvm
podman-compose up -d

# Stop
podman-compose down

# Logs
podman-compose logs -f ministack

# Verify
curl -X POST http://localhost:4566/ \
  -H "Content-Type: application/x-amz-json-1.0" \
  -H "X-Amz-Target: DynamoDB_20120810.ListTables" \
  -d '{}' | jq .
```

### Data Structure

```
~/Documents/tools/ministack/
├── docker-compose.yml
└── data/
    ├── dynamodb/    # DynamoDB persistent data
    ├── postgres/    # Future: PostgreSQL
    ├── rds/         # Future: RDS data
    └── redis/       # Future: Redis cache
```

**Key Design:** Single `/data` mount with service-specific subdirectories. Each service sets `DATA_DIR` env var to isolate persistence.

### Endpoints

- **Main:** `http://localhost:4566`
- **Legacy DynamoDB:** `http://localhost:8000`

### Credentials (Local Dev)

```
AWS_ACCESS_KEY_ID=local
AWS_SECRET_ACCESS_KEY=local
```

No actual AWS account needed. Dummy values sufficient.

### Tools

**DataGrip** — Database IDE with DynamoDB support
- Connection: `http://localhost:4566`
- Access Key: `local`
- Secret Key: `local`

---

## Why This Approach?

1. **Local-first:** No AWS charges, instant feedback
2. **Ministack:** Smaller, faster than full LocalStack
3. **Shared volume:** Extensible to multi-service setup (Postgres, Redis, etc.)
4. **Persistence:** Data survives container restart
5. **Tools:** GUI (DataGrip) + CLI (AWS SDK) for exploration

---

## Next Steps

1. Create first test table (via DataGrip or Go SDK)
2. Learn DynamoDB access patterns
3. Go SDK: CRUD operations
4. Advanced: Transactions, Streams, GSI
