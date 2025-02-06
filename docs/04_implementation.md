# Implementation Details

## Core Authentication Implementation

### User Authentication Service
```python
class AuthService:
    def __init__(self, db, trial_manager, ai_service):
        self.db = db
        self.trial_manager = trial_manager
        self.ai_service = ai_service
        self.jwt_secret = settings.JWT_SECRET
        
    async def register_user(self, user_data: UserRegistration) -> User:
        """Register new user with trial access"""
        # Validate user data
        await self._validate_registration(user_data)
        
        # Create user record
        user = await self.db.users.create(user_data)
        
        # Initialize trial
        await self.trial_manager.initialize_trial(user.id)
        
        # Generate initial preferences
        preferences = await self.ai_service.generate_initial_preferences(user_data)
        await self.db.preferences.create(user.id, preferences)
        
        return user

    async def login_user(self, credentials: UserCredentials) -> TokenPair:
        """Authenticate user and generate session tokens"""
        user = await self._verify_credentials(credentials)
        return await self._generate_tokens(user)

    async def refresh_token(self, refresh_token: str) -> TokenPair:
        """Generate new token pair from refresh token"""
        user_id = await self._verify_refresh_token(refresh_token)
        user = await self.db.users.get(user_id)
        return await self._generate_tokens(user)
```

## Trial Management System

### Trial Manager Implementation
```python
class TrialManager:
    def __init__(self, db, stripe_client):
        self.db = db
        self.stripe = stripe_client
        self.trial_duration = timedelta(days=14)
        
    async def initialize_trial(self, user_id: int) -> Trial:
        """Set up new trial for user"""
        trial = Trial(
            user_id=user_id,
            start_date=datetime.utcnow(),
            end_date=datetime.utcnow() + self.trial_duration,
            status="active"
        )
        return await self.db.trials.create(trial)

    async def check_trial_status(self, user_id: int) -> TrialStatus:
        """Check current trial status and metrics"""
        trial = await self.db.trials.get_by_user(user_id)
        usage = await self.db.usage.get_trial_usage(user_id)
        
        return TrialStatus(
            days_remaining=(trial.end_date - datetime.utcnow()).days,
            features_used=usage.features_used,
            usage_percentage=usage.percentage,
            should_convert=self._should_prompt_conversion(usage)
        )

    async def convert_to_premium(
        self,
        user_id: int,
        payment_data: PaymentInfo
    ) -> ConversionResult:
        """Convert trial user to premium"""
        # Create Stripe subscription
        subscription = await self.stripe.subscriptions.create(
            customer=payment_data.customer_id,
            price=settings.PREMIUM_PRICE_ID
        )
        
        # Update user status
        await self.db.users.update_status(
            user_id,
            new_status="premium",
            subscription_id=subscription.id
        )
        
        return ConversionResult(
            success=True,
            subscription_id=subscription.id
        )
```

## Premium Features Management

### Feature Access Control
```python
class FeatureManager:
    def __init__(self, db, trial_manager):
        self.db = db
        self.trial_manager = trial_manager
        
    async def check_feature_access(
        self,
        user_id: int,
        feature_name: str
    ) -> bool:
        """Check if user can access specific feature"""
        user = await self.db.users.get(user_id)
        
        if user.status == "premium":
            return True
            
        if user.status == "trial":
            trial_status = await self.trial_manager.check_trial_status(user_id)
            feature_limits = await self.db.features.get_trial_limits(feature_name)
            return self._check_trial_access(trial_status, feature_limits)
            
        return False

    async def track_feature_usage(
        self,
        user_id: int,
        feature_name: str
    ) -> None:
        """Record feature usage for analytics"""
        await self.db.usage.record_feature_use(
            user_id=user_id,
            feature=feature_name,
            timestamp=datetime.utcnow()
        )
```

## Database Models

### User Model
```python
class User(BaseModel):
    id: int
    email: str
    hashed_password: str
    status: Literal["trial", "premium", "expired"]
    created_at: datetime
    subscription_id: Optional[str]
    
    class Config:
        orm_mode = True

class Trial(BaseModel):
    id: int
    user_id: int
    start_date: datetime
    end_date: datetime
    status: Literal["active", "converted", "expired"]
    features_used: Dict[str, int]
    
    class Config:
        orm_mode = True
```

[Back to Overview](01_overview.md)
