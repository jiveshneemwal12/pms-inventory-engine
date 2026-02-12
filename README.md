# Django PMS Backend

A Property Management System (PMS) backend built with Django, following strict architectural guidelines for inventory correctness and transactional integrity.

##  Architecture

### Apps Structure

```
apps/
├── common/         # Shared utilities (base models, exceptions, transaction helpers)
├── authn/          # Authentication & authorization
├── properties/     # Property & room definitions
├── inventory/      # 🔴 CRITICAL: Inventory calendar & availability
├── rates/          # Pricing & rate plans
├── reservations/   # Booking lifecycle & transactions
├── events/         # Event publishing (Kafka)
└── realtime/       # WebSocket support (read-only)
```

### Key Principles

1. **Inventory Correctness (CRITICAL)**
   - All inventory writes use PostgreSQL transactions
   - `select_for_update(nowait=True)` for row-level locking
   - No async inventory writes
   - Truth source: PostgreSQL only

2. **Service Layer Pattern**
   - Business logic in service classes
   - Controllers (views) are thin
   - Services coordinate with transactions

3. **Event Publishing**
   - Events published AFTER DB commit only
   - Topics: `pms.inventory.updated`, `pms.reservation.created`, `pms.reservation.cancelled`

4. **Realtime (WebSockets)**
   - Read-only, no writes
   - No booking logic
   - Live dashboards only


### Prerequisites

- Python 3.12+
- PostgreSQL 16+
- Redis 7+
- (Optional) Kafka for event streaming

### Installation

1. **Clone and setup virtual environment**

```bash
cd pms-backend
python -m venv env
env\Scripts\activate  # Windows
# source env/bin/activate  # Linux/Mac
```

2. **Install dependencies**

```bash
pip install -r requirements/dev.txt
```

3. **Setup database**

Create PostgreSQL database:
```sql
CREATE DATABASE pms_dev;
```

4. **Run migrations**

```bash
python manage.py migrate
```

5. **Create superuser**

```bash
python manage.py createsuperuser
```

6. **Run development server**

```bash
python manage.py runserver
```

### Bootstrap Inventory

After creating properties and room types in admin:

```bash
python scripts/bootstrap_inventory.py
```

## Docker Setup

### Using Docker Compose

```bash
cd docker
docker-compose up -d
```

This starts:
- PostgreSQL (port 5432)
- Redis (port 6379)
- Kafka + Zookeeper (port 9092)
- Django web (port 8000)
- Celery worker
- Daphne WebSocket server (port 8001)

### Run migrations in Docker

```bash
docker-compose exec web python manage.py migrate
docker-compose exec web python manage.py createsuperuser
```

## 📡 API Endpoints

### Authentication

- `POST /api/auth/token/` - Obtain JWT token
- `POST /api/auth/token/refresh/` - Refresh JWT token

### Inventory (SYNC)

- `POST /api/inventory/check/` - Check availability
- `POST /api/inventory/preload/` - Preload inventory calendar

### Reservations (SYNC)

- `POST /api/reservations/create/` - Create booking
- `POST /api/reservations/{id}/cancel/` - Cancel booking

### WebSockets

- `ws://localhost:8001/ws/inventory/` - Live inventory dashboard
- `ws://localhost:8001/ws/reservations/` - Live reservation feed

## Testing

```bash
pytest
```

## Environment Variables

Create `.env` file:

```env
# Django
DEBUG=True
SECRET_KEY=your-secret-key
ALLOWED_HOSTS=localhost,127.0.0.1

# Database
DB_NAME=pms_dev
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=localhost
DB_PORT=5432

# Redis
REDIS_HOST=localhost

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

## Security

- Custom User model with email authentication
- JWT tokens for API authentication
- Role-based access: `SUPER_ADMIN`, `PROPERTY_ADMIN`, `STAFF`

## Database Models

### Core Models

- **Property** - Hotel/property definition
- **RoomType** - Room types with total units
- **InventoryCalendar** - Daily inventory (total_units, booked_units)
- **Reservation** - Booking information
- **ReservationRoom** - Room allocations
- **RatePlan** / **DailyRate** - Pricing

## Development

### Code Style

```bash
black .
isort .
flake8 .
```

### Database Migrations

```bash
python manage.py makemigrations
python manage.py migrate
```

### Admin Interface

Access at: http://localhost:8000/admin

## Technology Stack

- **Framework**: Django 5.1.7
- **API**: Django REST Framework
- **Database**: PostgreSQL 16
- **Cache/Queue**: Redis 7
- **Task Queue**: Celery
- **WebSockets**: Django Channels
- **Events**: Kafka (optional)
- **Auth**: JWT (Simple JWT)

## Critical Rules

1. **Never** async write to inventory
2. **Always** use transactions for inventory updates
3. **Always** use `select_for_update` for inventory rows
4. **Always** publish events AFTER commit
5. **Never** put business logic in views
6. **Never** write from WebSocket consumers

## 📖 Documentation

For detailed architectural decisions and design patterns, see the inline documentation in:

- `apps/inventory/services.py` - Inventory service layer
- `apps/reservations/services.py` - Reservation service layer
- `apps/events/producer.py` - Event publishing
- `apps/common/transactions.py` - Transaction helpers


## 📄 License

Proprietary - All Rights Reserved

