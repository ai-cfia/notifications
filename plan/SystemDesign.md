# Notification Service System Design

## Overview

This document outlines the high-level design for a web-based notification service that enables push notifications to users' browsers. The system consists of a lightweight frontend for user interaction and subscription management, and a FastAPI backend that receives webhook messages and forwards them as push notifications. Subscriptions are persisted in a PostgreSQL database.

## Architecture Components

### Frontend (Client-Side)
- **Technology Stack**: HTML, CSS, TypeScript
- **Key Components**:
  - Simple web interface with notification permission request
  - Service Worker for handling push events
  - Push API integration for subscribing to notifications

### Backend (Server-Side)
- **Technology Stack**: FastAPI (Python)
- **Key Components**:
  - Webhook endpoint for receiving messages
  - Push notification service integration (Mozilla Push Service)
  - PostgreSQL database for subscription persistence
  - SQLAlchemy ORM for database operations

### Database
- **Technology**: PostgreSQL
- **Purpose**: Store push subscriptions and related metadata

## System Flow

### 1. User Subscription Flow
```
User → Frontend → Request Notification Permission → Browser API
         ↓
    Register Service Worker
         ↓
    Subscribe to Push Service → Get Push Subscription
         ↓
    Send Subscription to Backend → Store in PostgreSQL
```

### 2. Notification Delivery Flow
```
External Service → Webhook POST → FastAPI Backend
                                        ↓
                              Process Message Payload
                                        ↓
                              Query Active Subscriptions from PostgreSQL
                                        ↓
                              Send to Mozilla Push Service (for each subscription)
                                        ↓
                              Push Service → Service Worker → Display Notification
```

## Detailed Component Design

### Frontend Components

#### 1. Main Web Page (index.html)
- Minimal UI with a single button: "Enable Notifications"
- Status indicators for:
  - Notification permission state
  - Service worker registration status
  - Push subscription status

#### 2. TypeScript Application (app.ts)
**Key Functions:**
- `requestNotificationPermission()`: Request browser notification permission
- `registerServiceWorker()`: Register the service worker
- `subscribeToPush()`: Subscribe to push notifications and get subscription object
- `sendSubscriptionToServer()`: Send the subscription details to backend

#### 3. Service Worker (service-worker.js)
**Event Handlers:**
- `push`: Listen for incoming push messages
- `notificationclick`: Handle notification interactions
- `notificationclose`: Track notification dismissals

### Backend Components

#### 1. FastAPI Application Structure
```
notification-service/
├── main.py              # FastAPI application entry point
├── models.py            # Pydantic models for request/response
├── database.py          # Database connection and session management
├── schemas.py           # SQLAlchemy models
├── push_service.py      # Mozilla Push Service integration
├── alembic/             # Database migrations
│   └── versions/
├── alembic.ini          # Alembic configuration
└── requirements.txt     # Python dependencies
```

#### 2. API Endpoints

**POST /webhook**
- Receives incoming messages from external services
- Request body: Message payload (title, body, data)
- Queries all active subscriptions from database
- Triggers push notification to all subscribed clients

**POST /subscribe**
- Receives and stores push subscription from frontend
- Validates subscription data
- Stores in PostgreSQL with metadata (created_at, user_agent, etc.)
- Returns subscription ID

**DELETE /unsubscribe/{subscription_id}**
- Soft delete or remove push subscription from database
- Returns success/failure status

**GET /subscriptions/count** (optional)
- Returns count of active subscriptions for monitoring

#### 3. Push Service Integration
- Uses `pywebpush` library for Mozilla Push Service
- Handles VAPID key generation and management
- Manages TTL and urgency headers
- Implements retry logic for failed deliveries
- Handles expired subscriptions (removes from database)

### Database Design

#### Schema: Push Subscriptions Table
```sql
CREATE TABLE push_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint VARCHAR(512) NOT NULL UNIQUE,
    p256dh VARCHAR(256) NOT NULL,
    auth VARCHAR(256) NOT NULL,
    user_agent VARCHAR(512),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    last_successful_push TIMESTAMP WITH TIME ZONE,
    failed_attempts INTEGER DEFAULT 0,
    metadata JSONB
);

CREATE INDEX idx_active_subscriptions ON push_subscriptions(is_active);
CREATE INDEX idx_endpoint ON push_subscriptions(endpoint);
```

## Data Models

### Webhook Message Model
```typescript
{
  "title": string,
  "body": string,
  "icon"?: string,
  "badge"?: string,
  "data"?: object,
  "actions"?: Array<{
    "action": string,
    "title": string
  }>
}
```

### Push Subscription Model
```typescript
{
  "endpoint": string,
  "keys": {
    "p256dh": string,
    "auth": string
  }
}
```

### Database Subscription Record
```python
class PushSubscription(Base):
    __tablename__ = "push_subscriptions"
    
    id = Column(UUID, primary_key=True, default=uuid4)
    endpoint = Column(String(512), nullable=False, unique=True)
    p256dh = Column(String(256), nullable=False)
    auth = Column(String(256), nullable=False)
    user_agent = Column(String(512))
    created_at = Column(DateTime(timezone=True), default=func.now())
    updated_at = Column(DateTime(timezone=True), default=func.now(), onupdate=func.now())
    is_active = Column(Boolean, default=True)
    last_successful_push = Column(DateTime(timezone=True))
    failed_attempts = Column(Integer, default=0)
    metadata = Column(JSONB)
```

## Security Considerations

1. **HTTPS Required**: Both frontend and backend must be served over HTTPS
2. **VAPID Authentication**: Use VAPID keys for push service authentication
3. **Webhook Security**: Implement webhook signature verification or API key authentication
4. **CORS Configuration**: Properly configure CORS for frontend-backend communication
5. **Database Security**: 
   - Use connection pooling
   - Implement proper SQL injection prevention
   - Encrypt sensitive data at rest
   - Use environment variables for database credentials

## Deployment Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Browser   │────►│   Frontend   │     │ External Service│
│             │     │  (Static)    │     │   (Webhook)     │
└──────┬──────┘     └──────────────┘     └────────┬────────┘
       │                                           │
       │                                           ▼
       │            ┌──────────────┐     ┌─────────────────┐
       │            │   FastAPI    │────►│  Mozilla Push   │
       └───────────►│   Backend    │     │    Service      │
                    └──────┬───────┘     └─────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  PostgreSQL  │
                    │   Database   │
                    └──────────────┘
```

## Implementation Steps

1. **Database Setup**
   - Set up PostgreSQL database
   - Create database schema
   - Set up Alembic for migrations
   - Create initial migration

2. **Frontend Setup**
   - Create basic HTML structure with notification button
   - Implement TypeScript logic for permission and subscription
   - Create service worker for push event handling

3. **Backend Setup**
   - Initialize FastAPI project
   - Set up SQLAlchemy models and database connection
   - Implement webhook endpoint
   - Implement subscription management endpoints
   - Integrate Mozilla Push Service client
   - Add VAPID key generation

4. **Integration**
   - Test frontend subscription flow
   - Verify webhook → push notification flow
   - Handle error cases and edge scenarios
   - Implement subscription cleanup for expired endpoints

5. **Deployment**
   - Deploy PostgreSQL database (e.g., AWS RDS, DigitalOcean Managed Database)
   - Deploy frontend as static files (e.g., GitHub Pages, Netlify)
   - Deploy FastAPI backend (e.g., Heroku, Railway, DigitalOcean)
   - Configure HTTPS and domain settings
   - Set up database backups and monitoring

## Testing Strategy

1. **Unit Tests**
   - Test webhook payload validation
   - Test push service integration
   - Test database operations (CRUD for subscriptions)

2. **Integration Tests**
   - End-to-end notification flow
   - Permission handling scenarios
   - Service worker registration
   - Database transaction handling

3. **Manual Testing**
   - Cross-browser compatibility
   - Mobile device support
   - Notification interaction behaviors
   - Database failover scenarios

## Future Enhancements

1. **User Authentication**: Implement user accounts and targeted notifications
2. **Analytics**: Track notification delivery and interaction metrics
3. **Rich Notifications**: Support for images, action buttons, and custom layouts
4. **Batch Notifications**: Send notifications to specific user groups or segments
5. **Dashboard**: Admin interface for monitoring and manual notification sending
6. **Rate Limiting**: Implement rate limiting for webhook endpoint
7. **Notification Templates**: Store and manage reusable notification templates
8. **Scheduled Notifications**: Support for scheduling notifications for future delivery
