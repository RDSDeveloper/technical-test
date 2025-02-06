# Scalability & Performance

## Database Sharding Strategy

### User Data Sharding
```python
class ShardManager:
    def __init__(self, config):
        self.num_shards = config.NUM_SHARDS
        self.shard_map = self._initialize_shards()
        
    def get_user_shard(self, user_id: int) -> str:
        """Determine shard for user data"""
        shard_key = hash(user_id) % self.num_shards
        return f"shard_{shard_key}"
        
    def get_photo_shard(self, upload_date: datetime) -> str:
        """Determine shard for photo data"""
        return f"shard_{upload_date.year}_{upload_date.month}"
```

## Session Management

### Distributed Session Handler
```python
REDIS_CONFIG = {
    'cluster_mode': True,
    'nodes': [
        {'host': 'primary', 'port': 6379},
        {'host': 'replica-1', 'port': 6379},
        {'host': 'replica-2', 'port': 6379}
    ],
    'read_from_replicas': True
}

class SessionManager:
    def __init__(self, redis_cluster):
        self.redis = redis_cluster
        self.token_ttl = timedelta(minutes=15)
        self.refresh_ttl = timedelta(days=30)
        
    async def create_session(
        self,
        user_id: int,
        device_info: dict
    ) -> SessionTokens:
        """Create new distributed session"""
        session_id = str(uuid.uuid4())
        
        # Store session data
        session_data = {
            "user_id": user_id,
            "device_info": device_info,
            "created_at": datetime.utcnow().isoformat()
        }
        
        # Store in Redis cluster
        await self.redis.set(
            f"session:{session_id}",
            json.dumps(session_data),
            ex=self.token_ttl
        )
        
        # Generate tokens
        access_token = self._generate_jwt(session_id, self.token_ttl)
        refresh_token = self._generate_refresh_token(session_id)
        
        return SessionTokens(
            access_token=access_token,
            refresh_token=refresh_token
        )
```

## Performance Optimization

### Caching Strategy
```python
class CacheManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.cache_config = {
            "user_profile": {
                "ttl": 300,  # 5 minutes
                "strategy": "write_through"
            },
            "photo_metadata": {
                "ttl": 3600,  # 1 hour
                "strategy": "lazy_loading"
            },
            "vector_data": {
                "ttl": 900,  # 15 minutes
                "strategy": "write_behind"
            }
        }
        
    async def get_cached_data(
        self,
        key: str,
        data_type: str,
        fetch_func: Callable
    ) -> Any:
        """Get data from cache or fetch if missing"""
        # Try cache first
        cached = await self.redis.get(key)
        if cached:
            return json.loads(cached)
            
        # Fetch and cache if missing
        data = await fetch_func()
        await self.cache_data(key, data, data_type)
        return data
```

### Load Balancing
```python
LOAD_BALANCER_CONFIG = {
    'algorithm': 'least_connections',
    'health_check': {
        'interval': '30s',
        'timeout': '5s',
        'healthy_threshold': 2,
        'unhealthy_threshold': 3
    },
    'sticky_sessions': {
        'enabled': True,
        'cookie_name': 'DAB_ROUTE'
    }
}

class HealthCheck:
    def __init__(self, config):
        self.config = config
        
    async def check_service_health(self, service: str) -> bool:
        """Check if service is healthy"""
        try:
            response = await self._ping_service(service)
            metrics = await self._gather_metrics(service)
            
            return (
                response.status == 200 and
                metrics.error_rate < 0.01 and
                metrics.latency_p95 < 500
            )
        except Exception:
            return False
```

## Monitoring & Alerts

### Performance Metrics
```python
ALERT_THRESHOLDS = {
    'auth_success_rate': {
        'warning': 95.0,  # Percentage
        'critical': 90.0
    },
    'api_latency': {
        'warning': 500,   # Milliseconds
        'critical': 1000
    },
    'error_rate': {
        'warning': 1.0,   # Percentage
        'critical': 5.0
    }
}

class PerformanceMonitor:
    def __init__(self, metrics_client, alert_service):
        self.metrics = metrics_client
        self.alerts = alert_service
        self.thresholds = ALERT_THRESHOLDS
        
    async def monitor_performance(self):
        """Monitor system performance metrics"""
        while True:
            metrics = await self._gather_metrics()
            await self._check_thresholds(metrics)
            await asyncio.sleep(60)
            
    async def _check_thresholds(self, metrics: dict):
        """Check metrics against thresholds"""
        for metric, value in metrics.items():
            if metric in self.thresholds:
                await self._evaluate_metric(metric, value)
```

[Back to Overview](01_overview.md)
