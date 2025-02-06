# Security & Compliance

## Security Considerations

### Authentication Security
1. **Password Security**
   - Argon2 hashing
   - Minimum complexity requirements
   - Regular rotation prompts
   - Breach detection

2. **Session Management**
   - Short-lived JWT tokens (15 minutes)
   - Secure HTTP-only refresh tokens
   - Device fingerprinting
   - Session invalidation on security events

3. **Rate Limiting**
   ```python
   RATE_LIMITS = {
       "login": {
           "attempts": 5,
           "window": "1h",
           "lockout": "24h"
       },
       "api": {
           "requests": 100,
           "window": "1m",
           "burst": 20
       }
   }
   ```

## GDPR Compliance

### Data Protection Measures
```python
class DataProtectionService:
    async def handle_data_request(
        self,
        user_id: int,
        request_type: Literal["export", "delete"]
    ) -> DataRequestResponse:
        """Handle GDPR data request"""
        if request_type == "export":
            return await self._export_user_data(user_id)
        else:
            return await self._delete_user_data(user_id)

    async def _export_user_data(self, user_id: int) -> UserDataExport:
        """Export all user data in portable format"""
        data = {
            "profile": await self.db.users.get(user_id),
            "photos": await self.storage.list_user_photos(user_id),
            "preferences": await self.db.preferences.get(user_id),
            "activity": await self.db.activity.get_user_history(user_id)
        }
        return self._format_for_export(data)

    async def _delete_user_data(self, user_id: int) -> DeletionConfirmation:
        """Permanently delete all user data"""
        await self._cancel_subscriptions(user_id)
        await self._delete_photos(user_id)
        await self._remove_database_records(user_id)
        return {"status": "completed", "timestamp": datetime.utcnow()}
```

### CCPA Compliance
```python
class CCPAComplianceService:
    async def handle_privacy_request(
        self,
        user_id: int,
        request_type: str
    ) -> PrivacyResponse:
        """Handle CCPA privacy request"""
        handlers = {
            "do_not_sell": self._handle_do_not_sell,
            "access": self._handle_access_request,
            "delete": self._handle_deletion_request
        }
        
        handler = handlers.get(request_type)
        if not handler:
            raise ValueError(f"Invalid request type: {request_type}")
            
        return await handler(user_id)
```

## Risk Mitigation

### Security Monitoring
```python
class SecurityMonitor:
    def __init__(self, alert_service):
        self.alert_service = alert_service
        
    async def monitor_security_events(self):
        """Monitor and alert on security events"""
        while True:
            events = await self._gather_security_events()
            for event in events:
                if self._is_security_threat(event):
                    await self._handle_security_threat(event)
            await asyncio.sleep(10)

    async def _handle_security_threat(self, event: SecurityEvent):
        """Handle detected security threat"""
        await self.alert_service.send_alert(
            level="high",
            title=f"Security Threat Detected: {event.type}",
            details=event.details
        )
        await self._take_protective_action(event)
```

### Audit Logging
```python
class AuditLogger:
    def __init__(self, db):
        self.db = db
        
    async def log_security_event(
        self,
        event_type: str,
        user_id: Optional[int],
        details: dict
    ) -> None:
        """Log security-related event"""
        await self.db.audit_logs.create(
            AuditLog(
                event_type=event_type,
                user_id=user_id,
                timestamp=datetime.utcnow(),
                details=details,
                ip_address=self._get_ip_address(),
                user_agent=self._get_user_agent()
            )
        )
```

## Data Protection

### Encryption Service
```python
class EncryptionService:
    def __init__(self, key_manager):
        self.key_manager = key_manager
        
    async def encrypt_sensitive_data(
        self,
        data: Union[str, bytes],
        context: str
    ) -> EncryptedData:
        """Encrypt sensitive data with context"""
        key = await self.key_manager.get_key(context)
        nonce = secrets.token_bytes(12)
        
        cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
        cipher.update(context.encode())
        
        ciphertext, tag = cipher.encrypt_and_digest(
            data if isinstance(data, bytes) else data.encode()
        )
        
        return EncryptedData(
            ciphertext=ciphertext,
            nonce=nonce,
            tag=tag,
            context=context
        )
```

[Back to Overview](01_overview.md)
