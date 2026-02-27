# Redis Implementation Explained

## What is Redis?

**Redis** (Remote Dictionary Server) is a **software server** (not a platform like AWS/Azure). It's a database that runs as a service on a machine, similar to how PostgreSQL or MySQL run as services.

### Understanding Redis vs. Other Databases

Think of it like this:

```
Regular Database (PostgreSQL, MySQL):
├── Data stored on DISK (hard drive)
├── Slower (disk I/O)
└── Persistent (survives restarts)

Redis:
├── Data stored in RAM (memory)
├── Very fast (memory access)
└── Can be persistent (optional)
```

### Key Characteristics

- **In-memory storage**: Data is stored in RAM, not on disk (much faster)
- **Key-value store**: You store data with a key (like a filename) and retrieve it later
- **Temporary storage**: Data can expire or be cleared (good for caching)
- **Very fast**: Can handle millions of operations per second
- **Software server**: Runs as a separate service/process on a machine

### Common Use Cases

- Caching frequently accessed data
- Session storage
- Real-time analytics
- Message queues
- Temporary data storage

---

## Where Does Redis Run?

Redis runs as a **separate service/server**. You can run it in several ways:

### 1. Local Development (On Your Computer)

```bash
# Install Redis
brew install redis        # macOS
sudo apt-get install redis-server  # Linux

# Start Redis server
redis-server

# Redis now runs on: localhost:6379
```

- Runs on your local machine
- You manually start it: `redis-server`
- Data stored in your computer's RAM
- Accessible at `redis://localhost:6379`

### 2. Cloud Service (Production)

- **Azure Redis Cache** (used in this project's production)
- **AWS ElastiCache**
- **Google Cloud Memorystore**
- **Managed Redis services** from various providers

These are fully managed services where the cloud provider runs Redis for you.

### 3. Docker Container

```bash
docker run -d -p 6379:6379 redis
```

- Runs Redis in a container
- Isolated from your host system
- Easy to start/stop

---

## Where is Data Stored?

### Primary Storage: RAM (Memory)

- **Data lives in RAM** (your computer's memory or server's memory)
- **Very fast** reads/writes (memory is much faster than disk)
- **Volatile**: Data is lost if the server restarts (unless persistence is enabled)

### Optional Persistence: Disk

Redis can optionally save snapshots to disk:
- **RDB snapshots**: Periodic snapshots of data
- **AOF (Append Only File)**: Logs every write operation
- Used for **recovery** after restart, not for normal reads
- Data is still primarily accessed from RAM

### Visual Representation

```
┌─────────────────────────────────────────┐
│  Your Application (Predictive Audience) │
│  pa/api/main.py, pa/flows/...          │
└──────────────┬──────────────────────────┘
               │
               │ Connects via REDIS_URL
               │ (redis://host:port)
               ▼
┌─────────────────────────────────────────┐
│         Redis Server                     │
│  ┌─────────────────────────────────┐   │
│  │  RAM (Memory)                    │   │
│  │  ┌───────────────────────────┐   │   │
│  │  │ run_metadata:70132:...   │   │   │
│  │  │ features:70132:email.rfm  │   │   │
│  │  │ (All data stored here)     │   │   │
│  │  └───────────────────────────┘   │   │
│  └─────────────────────────────────┘   │
│                                          │
│  Optional: Disk persistence             │
│  (for backup/recovery)                  │
└─────────────────────────────────────────┘
```

---

## Redis in This Codebase

### Development Environment

```python
# Default in pa/__init__.py
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
```

- Runs **locally on your machine**
- You start it manually: `redis-server`
- Data stored in **your computer's RAM**
- Accessible at `localhost:6379`

### Production Environment (Azure)

```bicep
// From infrastructure/azure/worker-container-instance.bicep
{
  name: 'REDIS_URL'
  value: '@Microsoft.KeyVault(SecretUri=.../REDIS-URL/)'
}
```

- Uses **Azure Redis Cache** (managed service)
- Runs on **Azure infrastructure**
- Data stored in **Azure VM RAM**
- URL like: `redis://your-redis-cache.redis.cache.windows.net:6380`

---

## Comparison with Other Storage in This Project

| Storage Type | Location | Speed | Persistence | Use Case |
|-------------|----------|-------|-------------|----------|
| **Redis** | RAM | Very fast | Optional | Caching, temporary data |
| **PostgreSQL** (DBOS) | Disk | Fast | Always | Workflow state, history |
| **Synapse** (Azure) | Disk (cloud) | Fast | Always | Data warehouse, permanent data |
| **File System** | Disk | Slow | Always | CSV files, logs |

### Data Storage Hierarchy in This Project

1. **Synapse (Azure)**: Permanent data
   - Email events, metrics tables
   - Source of truth for all historical data

2. **PostgreSQL (DBOS)**: Workflow state
   - Workflow execution history
   - Step completion records
   - System database for orchestration

3. **Redis**: Temporary/cached data
   - Run metadata (when steps last ran)
   - Feature cache (computed scores)
   - Fast access for smart skipping

4. **File System**: CSV files
   - Features, segments
   - Backup/persistence layer
   - Fallback when Redis unavailable

---

## Why Redis is Used in This Codebase

Redis serves **two main purposes** in the Predictive Audience pipeline:

### 1. **Run Metadata Tracking** (Smart Skipping)
Tracks when pipeline steps last ran, so the system can skip steps that ran recently (within 24 hours).

**Example:**
- If email metrics were computed 2 hours ago, skip recomputing them
- If features were generated yesterday, skip regenerating them today

### 2. **Feature Caching** (Performance)
Stores computed features (RFM scores, EPI scores, etc.) in memory for fast retrieval instead of reading from CSV files.

**Example:**
- Instead of reading `70132_email_rfm_20250115.csv` from disk every time
- Store it in Redis and retrieve it instantly from memory

---

## How Redis is Configured

### Environment Variable
Redis connection is configured via the `REDIS_URL` environment variable:

```python
# In pa/__init__.py
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
```

**URL Format:**
- `redis://localhost:6379` - Local Redis (default)
- `redis://hostname:6379` - Remote Redis
- `rediss://hostname:6380` - Redis with SSL
- `unix:///path/to/socket` - Unix socket connection

**Default:** If not set, defaults to `redis://localhost:6379`

---

## Implementation Details

### 1. Run Metadata Tracking (`pa/utils/run_metadata.py`)

#### Purpose
Tracks the last execution time for each pipeline step (metrics, features, segments, LTV) per client and channel.

#### Data Structure in Redis

**Key Pattern:**
```
run_metadata:{client_id}:{channel}:{step}
```

**Examples:**
- `run_metadata:70132:Email:metrics` - Last time email metrics were computed for client 70132
- `run_metadata:70132:Email:features` - Last time email features were generated
- `run_metadata:70132:LP:segments` - Last time LP segments were created
- `run_metadata:70132:ltv` - Last time LTV was calculated (cross-channel)

**Value Format (JSON):**
```json
{
  "last_run_date": "2025-01-15T10:30:00",
  "updated_at": "2025-01-15T10:30:00"
}
```

#### Key Functions

##### `get_last_run_date(client_id, channel, step)`
Retrieves when a step last ran.

```python
async def get_last_run_date(client_id: int, channel: str, step: str) -> Optional[datetime]:
    """
    Get the last run date for a specific step.
    
    Returns:
        datetime of last run or None if never run
    """
    if not _is_redis_available():
        return None  # Graceful: return None if Redis unavailable
    
    redis = await aioredis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
    try:
        key = f"run_metadata:{client_id}:{channel}:{step}"
        metadata_str = await redis.get(key)  # Get value from Redis
        
        if metadata_str:
            metadata = json.loads(metadata_str)  # Parse JSON
            return datetime.fromisoformat(metadata["last_run_date"])
        
        return None  # Never run before
    finally:
        await redis.close()
```

**Usage in Pipeline:**
```python
# In channel_pipeline.py
if skip_if_recent:
    if await should_skip_step(client_id, "Email", "features"):
        logger.info("Skipping email features (recent run exists)")
        return False  # Skip this step
```

##### `update_last_run_date(client_id, channel, step, run_date)`
Stores when a step completed.

```python
async def update_last_run_date(client_id: int, channel: str, step: str, run_date: datetime):
    """
    Update the last run date for a specific step.
    """
    if not _is_redis_available():
        return  # Graceful: silently skip if Redis unavailable
    
    redis = await aioredis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
    try:
        key = f"run_metadata:{client_id}:{channel}:{step}"
        metadata = {
            "last_run_date": run_date.isoformat(),
            "updated_at": datetime.now().isoformat(),
        }
        await redis.set(key, json.dumps(metadata))  # Store in Redis
    finally:
        await redis.close()
```

**Usage in Pipeline:**
```python
# After completing a step
await update_last_run_date(client_id, "Email", "features", datetime.now())
```

##### `should_skip_step(client_id, channel, step, max_age_days=1)`
Determines if a step should be skipped based on last run date.

```python
async def should_skip_step(client_id: int, channel: str, step: str, max_age_days: int = 1) -> bool:
    """
    Determine if a step should be skipped based on last run date.
    
    Returns:
        True if step should be skipped (recent run exists), False otherwise
    """
    last_run = await get_last_run_date(client_id, channel, step)
    
    if last_run is None:
        return False  # Never run, don't skip
    
    age_days = (datetime.now() - last_run).days
    should_skip = age_days < max_age_days  # Skip if run within max_age_days
    
    if should_skip:
        logger.info(f"Skipping {step} for {client_id}/{channel}: last run {age_days} days ago")
    
    return should_skip
```

**Example Flow:**
1. Pipeline starts for client 70132, Email channel
2. Checks `run_metadata:70132:Email:features` in Redis
3. Finds last run was 2 hours ago
4. Since 2 hours < 24 hours (max_age_days=1), skips feature generation
5. Continues to next step

---

### 2. Feature Caching (`pa/ds/features.py`)

#### Purpose
Stores computed features (CSV data) in Redis for fast retrieval, avoiding slow disk reads.

#### Data Structure in Redis

**Key Pattern:**
```
features:{client_id}:{channel}.{feature_type}
```

**Examples:**
- `features:70132:email.rfm` - Email RFM scores for client 70132
- `features:70132:lp.epi` - Landing Page EPI scores
- `features:70132:email.mov` - Email Movement scores

**Value Format:**
CSV string (entire CSV file content stored as text)

```
RecordId,rfm_score,mov_score,epi_score
12345,0.85,0.72,0.91
12346,0.62,0.45,0.78
...
```

#### Key Functions

##### `get_available_features(client_id, channel)`
Checks both Redis and file system for available features.

```python
async def get_available_features(client_id: int, channel: str) -> Dict[str, str]:
    """
    Get available features for a given client and channel.
    Checks both Redis and file system.
    """
    available_features = {}
    
    # Step 1: Check Redis if available
    if REDIS_URL and REDIS_URL.startswith(('redis://', 'rediss://', 'unix://')):
        try:
            redis = aioredis.Redis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
            try:
                # Search for keys matching pattern
                pattern = f"features:{client_id}:{channel}*"
                keys = await redis.keys(pattern)  # Find all matching keys
                
                for key in keys:
                    ch, feature_type = parse_feature_redis_key(key)
                    if feature_type in feature_descriptions:
                        available_features[feature_type] = feature_descriptions[feature_type]
            except RedisError as e:
                logger.warning(f"Redis error: {str(e)}")
            finally:
                await redis.close()
        except Exception as e:
            logger.warning(f"Failed to connect to Redis: {str(e)}")
    
    # Step 2: Check file system (fallback)
    file_pattern = os.path.join(FEATURES_DIR, f"{client_id}_{channel}*.csv")
    files = glob.glob(file_pattern)
    for file in files:
        feature_type, date = parse_feature_filename(file)
        if feature_type not in available_features:
            available_features[feature_type] = feature_descriptions[feature_type]
            
            # Step 3: Try to populate Redis from CSV (for future fast access)
            if REDIS_URL and REDIS_URL.startswith(('redis://', 'rediss://', 'unix://')):
                try:
                    redis = aioredis.Redis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
                    try:
                        with open(file, 'r') as csv_file:
                            csv_content = csv_file.read()
                            redis_key = f"features:{client_id}:{channel}.{feature_type}"
                            await redis.set(redis_key, csv_content)  # Store CSV in Redis
                    finally:
                        await redis.close()
                except Exception as e:
                    logger.debug(f"Could not add CSV to Redis: {str(e)}")
    
    return available_features
```

**Flow:**
1. First checks Redis for cached features (fast)
2. If not in Redis, checks file system (slower)
3. If found in file system, copies to Redis for next time (optimization)

##### `get_features_from_redis(client_id, feature_type, feature_name)`
Retrieves features from Redis and converts to DataFrame.

```python
def get_features_from_redis(client_id: int, feature_type: str, feature_name: str) -> pd.DataFrame:
    """
    Retrieve features from Redis and return as a DataFrame.
    """
    if not REDIS_URL or not REDIS_URL.startswith(('redis://', 'rediss://', 'unix://')):
        logger.warning("Invalid or missing REDIS_URL")
        return pd.DataFrame()  # Return empty DataFrame
    
    try:
        r = redis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
        
        # Construct Redis key
        key = f"features:{client_id}:{feature_type}.{feature_name}"
        
        # Retrieve CSV data from Redis
        csv_data = r.get(key)
        
        if csv_data is None:
            logger.warning(f"No data found in Redis for key: {key}")
            return pd.DataFrame()
        
        # Parse CSV string into DataFrame
        df = pd.read_csv(io.StringIO(csv_data))
        
        # Select only required columns
        column_name = f"{feature_name}_score"
        result_df = df[['RecordId', column_name]].copy()
        
        return result_df
    
    except redis.RedisError as e:
        logger.warning(f"Redis error: {str(e)}")
        return pd.DataFrame()  # Graceful: return empty DataFrame
```

**Usage:**
```python
# In workbench/ui.py
df = get_features_from_redis(selected_client_id, "email", "rfm")
# Returns DataFrame with RecordId and rfm_score columns
```

##### `store_features_in_redis(client_id, channel, features, baseline_date)`
Stores computed features in Redis (from `pa/computer.py`).

```python
async def store_features_in_redis(client_id: int, channel: str, features, baseline_date: str):
    """
    Store features in Redis for fast retrieval later.
    """
    redis = await aioredis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
    try:
        key = f"features:{client_id}:{channel}:{baseline_date}"
        
        # Convert Polars DataFrame to CSV string
        csv_buffer = io.StringIO()
        features.write_csv(csv_buffer)
        csv_string = csv_buffer.getvalue()
        
        # Store in Redis
        await redis.set(key, csv_string)
        logger.info(f"Stored features in Redis with key: {key}")
    finally:
        await redis.close()
```

---

## Graceful Degradation (Redis Optional)

**Important:** Redis is **optional** in this codebase. The system works without it, but with reduced functionality.

### How It Works

Every Redis operation checks if Redis is available first:

```python
def _is_redis_available() -> bool:
    """Check if Redis is configured and available."""
    if not REDIS_URL:
        return False
    if not REDIS_URL.startswith(("redis://", "rediss://", "unix://")):
        return False
    return True
```

### Behavior When Redis is Unavailable

#### Run Metadata Tracking
- **With Redis:** Tracks last run dates, enables smart skipping
- **Without Redis:** Always returns `None` for last run dates, never skips steps (always runs)

```python
async def get_last_run_date(...):
    if not _is_redis_available():
        logger.debug("Redis not available, skipping get_last_run_date")
        return None  # No error, just returns None
```

#### Feature Caching
- **With Redis:** Features cached in memory for fast access
- **Without Redis:** Always reads from file system (slower but works)

```python
def get_features_from_redis(...):
    if not REDIS_URL or not REDIS_URL.startswith(...):
        logger.warning("Invalid or missing REDIS_URL")
        return pd.DataFrame()  # Returns empty, caller falls back to file system
```

### Error Handling Pattern

All Redis operations use try-except blocks and log warnings instead of raising errors:

```python
try:
    redis = await aioredis.from_url(REDIS_URL, ...)
    # ... Redis operations ...
except Exception as e:
    logger.warning(f"Redis error: {str(e)}. Continuing without Redis.")
    # Don't raise - allow pipeline to continue
finally:
    await redis.close()
```

**Result:** Pipeline never fails due to Redis issues. It just runs slower or without smart skipping.

---

## Redis Client Libraries Used

### Async Redis (`aioredis`)
Used for async/await operations in the pipeline:

```python
from redis import asyncio as aioredis

redis = await aioredis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
await redis.set(key, value)
value = await redis.get(key)
await redis.close()
```

### Sync Redis (`redis`)
Used for synchronous operations:

```python
import redis

r = redis.from_url(REDIS_URL, encoding="utf8", decode_responses=True)
r.set(key, value)
value = r.get(key)
```

---

## Data Flow Examples

### Example 1: Pipeline Run with Redis

1. **Pipeline starts** for client 70132, Email channel
2. **Check Redis:** `run_metadata:70132:Email:features`
   - Found: Last run 2 hours ago
   - Decision: Skip feature generation (within 24 hours)
3. **Continue to segments step**
4. **After segments complete:** Update Redis
   - Key: `run_metadata:70132:Email:segments`
   - Value: `{"last_run_date": "2025-01-15T14:30:00", ...}`
5. **Store features in Redis:**
   - Key: `features:70132:email.rfm`
   - Value: CSV content of RFM scores

### Example 2: Feature Retrieval with Redis

1. **User requests features** for client 70132, email channel
2. **Check Redis first:** `features:70132:email.rfm`
   - Found: CSV data in memory
   - Return: DataFrame instantly (fast!)
3. **If not in Redis:**
   - Read from file: `data/features/70132_email_rfm_20250115.csv`
   - Store in Redis for next time
   - Return: DataFrame (slower this time, faster next time)

### Example 3: Pipeline Run Without Redis

1. **Pipeline starts** for client 70132, Email channel
2. **Check Redis:** `_is_redis_available()` returns `False`
3. **Skip check:** Always runs all steps (no smart skipping)
4. **Features:** Always read from file system (no caching)
5. **Pipeline completes:** Works fine, just slower

---

## Configuration in Production

### Azure Infrastructure

Redis URL is stored in Azure Key Vault and injected into containers:

```bicep
// In infrastructure/azure/worker-container-instance.bicep
environmentVariables: [
  {
    name: 'REDIS_URL'
    value: '@Microsoft.KeyVault(SecretUri=https://${keyVaultName}.vault.azure.net/secrets/REDIS-URL/)'
  }
]
```

### Environment Templates

Config templates show Redis configuration:

```bash
# In config/dev.env.template
REDIS_URL=redis://localhost:6379

# In config/production.env.template
REDIS_URL=redis://your-redis-instance:6379
```

---

## Summary

### What Redis Does
1. **Run Metadata:** Tracks when steps last ran (enables smart skipping)
2. **Feature Caching:** Stores computed features for fast retrieval

### Key Design Principles
- **Optional:** System works without Redis (graceful degradation)
- **Fast:** In-memory storage for performance
- **Resilient:** Errors don't crash the pipeline (warnings only)
- **Fallback:** Always falls back to file system if Redis unavailable

### When to Use Redis
- **Development:** Optional (can use local Redis or skip)
- **Production:** Recommended for performance and smart skipping
- **Testing:** Can test without Redis to verify fallback behavior

### Current Status
- ✅ Fully implemented with async support
- ✅ Graceful error handling
- ✅ File system fallback
- ✅ Production-ready (configured in Azure)

---

## Setting Up Redis in Azure

This section provides a complete guide for creating and configuring Azure Redis Cache for the Predictive Audience application.

### Step 1: Create Azure Redis Cache Instance

#### Option A: Using Azure Portal

1. **Navigate to Azure Portal**: https://portal.azure.com
2. **Create a Resource**: Click "Create a resource"
3. **Search for Redis**: Type "Azure Cache for Redis"
4. **Create Instance**:
   - **Subscription**: Select your subscription
   - **Resource Group**: Select your resource group (e.g., `exp_ai_rg`)
   - **DNS name**: Choose a unique name (e.g., `pa-redis-dev` or `pa-redis-prod`)
   - **Location**: Select same region as your other resources (e.g., `East US`)
   - **Pricing tier**: 
     - **Development**: Basic (C0) - 250 MB, ~$15/month
     - **Production**: Standard (C1) - 1 GB, ~$55/month or higher
   - **Port**: Leave default (6379 for non-SSL, 6380 for SSL)
   - **Non-SSL port**: Enable for development (disable for production with SSL)
5. **Review and Create**: Click "Review + create" then "Create"

#### Option B: Using Azure CLI

```bash
# Set variables
export RESOURCE_GROUP="exp_ai_rg"
export REDIS_NAME="pa-redis-dev"  # Must be globally unique
export LOCATION="eastus"
export SKU="Basic"  # Options: Basic, Standard, Premium
export CAPACITY="C0"  # C0 = 250MB, C1 = 1GB, C2 = 2.5GB, etc.

# Create Redis Cache
az redis create \
  --resource-group $RESOURCE_GROUP \
  --name $REDIS_NAME \
  --location $LOCATION \
  --sku $SKU \
  --vm-size $CAPACITY \
  --enable-non-ssl-port  # Enable for development (disable for production)
```

**Note**: The Redis name must be globally unique (5-50 characters, alphanumeric and hyphens).

---

### Step 2: Get Redis Connection Details

#### Using Azure Portal

1. Navigate to your Redis Cache instance
2. Go to **Settings** → **Access keys**
3. Copy the **Primary connection string** or **Primary host** and **Primary access key**

#### Using Azure CLI

```bash
# Get Redis connection details
az redis show \
  --resource-group $RESOURCE_GROUP \
  --name $REDIS_NAME \
  --query "{hostName:hostName,sslPort:port,nonSslPort:nonSslPort}" \
  --output table

# Get access keys
az redis list-keys \
  --resource-group $RESOURCE_GROUP \
  --name $REDIS_NAME \
  --query "{primaryKey:primaryKey,secondaryKey:secondaryKey}" \
  --output table
```

#### Construct Redis URL

The Redis URL format is:
```
redis://[:password]@host:port
```

**For Non-SSL (Development):**
```bash
# Format: redis://:password@hostname:6379
REDIS_URL="redis://:YourPrimaryKey@pa-redis-dev.redis.cache.windows.net:6379"
```

**For SSL (Production):**
```bash
# Format: rediss://:password@hostname:6380
REDIS_URL="rediss://:YourPrimaryKey@pa-redis-prod.redis.cache.windows.net:6380"
```

**Note**: 
- Non-SSL uses port 6379
- SSL uses port 6380
- `rediss://` (with double 's') indicates SSL connection

---

### Step 3: Store Redis URL in Azure Key Vault

The infrastructure expects Redis URL to be stored in Key Vault as a secret named `REDIS-URL`.

#### Using Azure Portal

1. Navigate to your Key Vault (e.g., `cdp-keyvault-dev-755`)
2. Go to **Secrets** → **+ Generate/Import**
3. **Name**: `REDIS-URL`
4. **Value**: Paste your Redis connection string (e.g., `redis://:password@host:6379`)
5. **Content Type**: Leave empty or enter "Redis Connection String"
6. Click **Create**

#### Using Azure CLI

```bash
# Set variables
export KEY_VAULT_NAME="cdp-keyvault-dev-755"
export REDIS_URL="redis://:YourPrimaryKey@pa-redis-dev.redis.cache.windows.net:6379"

# Store in Key Vault
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "REDIS-URL" \
  --value "$REDIS_URL"
```

**Security Note**: Never commit Redis URLs or passwords to version control. Always use Key Vault.

**Important**: The Bicep templates automatically pull this from Key Vault:
```bicep
// From infrastructure/azure/worker-container-instance.bicep
{
  name: 'REDIS_URL'
  value: '@Microsoft.KeyVault(SecretUri=https://${keyVaultName}.vault.azure.net/secrets/REDIS-URL/)'
}
```

This means:
- ✅ No code changes needed
- ✅ Redis URL is injected at runtime
- ✅ Secrets are managed securely in Key Vault

---

### Step 4: Configure Firewall Rules (Optional but Recommended)

By default, Azure Redis Cache is accessible from anywhere. For production, restrict access to your resources.

#### Using Azure Portal

1. Navigate to your Redis Cache instance
2. Go to **Settings** → **Firewall and virtual networks**
3. **Firewall rules**: Add IP addresses or virtual networks
4. **Allow access from**: Select "Selected networks" or "All networks"

#### Using Azure CLI

```bash
# Add firewall rule for specific IP
az redis firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --name $REDIS_NAME \
  --rule-name "AllowAppService" \
  --start-ip-address "1.2.3.4" \
  --end-ip-address "1.2.3.4"

# Or allow access from virtual network
az redis update \
  --resource-group $RESOURCE_GROUP \
  --name $REDIS_NAME \
  --public-network-access Disabled \
  --subnet-id "/subscriptions/.../virtualNetworks/.../subnets/..."
```

---

### Step 5: Test Redis Connection

#### Test from Azure Portal

1. Navigate to your Redis Cache instance
2. Go to **Tools** → **Redis Console**
3. Run test commands:
   ```
   PING
   SET test "Hello Redis"
   GET test
   ```

#### Test from Application

After deploying, check application logs:

```bash
# Check API logs
az webapp log tail \
  --name pa-api-dev-v2 \
  --resource-group $RESOURCE_GROUP

# Check Worker logs
az container logs \
  --name pa-worker-dev \
  --resource-group $RESOURCE_GROUP
```

Look for:
- ✅ `"Redis not available"` messages should disappear
- ✅ `"Stored features in Redis"` messages when features are cached
- ✅ `"Updated last run date"` messages when pipeline runs

#### Test from Python (Local)

```python
import redis
import os

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")

try:
    r = redis.from_url(REDIS_URL, decode_responses=True)
    r.ping()
    print("✅ Redis connection successful!")
    
    # Test write/read
    r.set("test_key", "test_value")
    value = r.get("test_key")
    print(f"✅ Test value: {value}")
except Exception as e:
    print(f"❌ Redis connection failed: {e}")
```

---

### Step 6: Monitor Redis Usage

#### Using Azure Portal

1. Navigate to your Redis Cache instance
2. Go to **Metrics** to view:
   - **Cache hits/misses**: Check cache effectiveness
   - **Connected clients**: Number of active connections
   - **Memory usage**: Monitor cache size
   - **Operations**: Commands per second

#### Using Azure CLI

```bash
# Get metrics
az monitor metrics list \
  --resource /subscriptions/.../resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Cache/redis/$REDIS_NAME \
  --metric "cachehits" "cachemisses" \
  --start-time 2025-01-15T00:00:00Z \
  --end-time 2025-01-15T23:59:59Z
```

---

### Troubleshooting

#### Issue: Connection Timeout

**Symptoms**: Application can't connect to Redis

**Solutions**:
1. Check firewall rules (allow your IP/network)
2. Verify Redis URL format (correct port, password)
3. Check if Redis instance is running
4. Verify Key Vault secret exists and is accessible

#### Issue: Authentication Failed

**Symptoms**: `NOAUTH Authentication required` error

**Solutions**:
1. Verify access key is correct in Key Vault
2. Check Redis URL includes password: `redis://:password@host:port`
3. Regenerate access keys if needed

#### Issue: SSL/TLS Errors

**Symptoms**: Connection errors with SSL

**Solutions**:
1. For development: Use non-SSL port (6379) with `redis://`
2. For production: Use SSL port (6380) with `rediss://`
3. Verify firewall allows the correct port

#### Issue: Out of Memory

**Symptoms**: Redis operations fail with memory errors

**Solutions**:
1. Upgrade to larger SKU (C1 → C2 → C3)
2. Set TTL (time-to-live) on keys
3. Monitor memory usage and clean up old keys

---

### Cost Considerations

#### Pricing Tiers

| Tier | Size | Price (approx) | Use Case |
|------|------|----------------|----------|
| **Basic C0** | 250 MB | $15/month | Development |
| **Basic C1** | 1 GB | $55/month | Small production |
| **Standard C1** | 1 GB | $55/month | Production (with replication) |
| **Standard C2** | 2.5 GB | $130/month | Larger workloads |
| **Premium** | 6 GB+ | $300+/month | Enterprise |

#### Cost Optimization Tips

1. **Development**: Use Basic C0 (250 MB) - sufficient for testing
2. **Production**: Start with Basic C1, monitor usage, upgrade if needed
3. **Set TTL**: Configure key expiration to prevent memory bloat
4. **Monitor**: Set up alerts for memory usage (80% threshold)

---

### Security Best Practices

1. **Use SSL in Production**: Always use `rediss://` with port 6380
2. **Firewall Rules**: Restrict access to specific IPs/networks
3. **Key Vault**: Store Redis URL in Key Vault, never in code
4. **Access Keys**: Rotate keys periodically (every 90 days)
5. **Managed Identity**: Use managed identity for Key Vault access (already configured)
6. **Network Isolation**: Use Virtual Network integration for enhanced security

---

### Summary

After completing these steps:

1. ✅ Azure Redis Cache created
2. ✅ Redis URL stored in Key Vault
3. ✅ Application automatically picks up Redis URL from Key Vault
4. ✅ Redis connection verified
5. ✅ Monitoring configured

Your Predictive Audience application will now:
- ✅ Cache features in Redis for faster access
- ✅ Track run metadata for smart step skipping
- ✅ Improve pipeline performance

**Next Steps**: 
- Deploy your application (Redis URL will be automatically injected)
- Monitor Redis metrics in Azure Portal
- Adjust firewall rules as needed
- Scale up if memory usage is high

---

## Further Reading

- **Redis Documentation:** https://redis.io/docs/
- **aioredis Documentation:** https://aioredis.readthedocs.io/
- **REDIS_FIX.md:** Details about making Redis optional
- **Run Metadata Code:** `pa/utils/run_metadata.py`
- **Feature Caching Code:** `pa/ds/features.py`

