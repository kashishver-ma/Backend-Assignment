# Webhook API

A FastAPI-based webhook ingestion service with HMAC verification, SQLite persistence, pagination, and metrics.

## Tech Stack

- Python 3.14.2
- FastAPI
- SQLite
- Prometheus client
- Docker 29.1.5
- Docker Compose v5.0.1

## Quick Start

### Using Docker (Recommended)

```bash
# Set environment variables
export WEBHOOK_SECRET="testsecret"
export DATABASE_URL="sqlite:////data/app.db"

# /data folder is automatically created by Docker in a volume)

# Start the service
docker compose up -d --build

# API will be available at
 http://localhost:8000
```

**Useful Commands:**

```bash
docker compose down -v        # Stop and clean up
docker compose logs -f api    # View logs
```

### Using Python Locally

```bash
# 1. Create virtual environment
python -m venv venv
source venv/bin/activate     # On Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Set environment variables
export WEBHOOK_SECRET="testsecret"              # On Windows: $env:WEBHOOK_SECRET="testsecret"
export DATABASE_URL="sqlite:///./app.db"        # On Windows: $env:DATABASE_URL="sqlite:///./app.db"

# 4. Start server
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

## API Endpoints

### 1. Health Checks

```bash
curl http://localhost:8000/health/live
curl http://localhost:8000/health/ready
```

**Response:** `{ "status": "ok" }`

### 2. Webhook Endpoint

**POST** `/webhook`

Receives webhook messages with HMAC signature verification.

**Example:**

```bash
curl -X POST http://localhost:8000/webhook \
  -H "Content-Type: application/json" \
  -H "X-Signature: <hmac-signature>" \
  -d '{
    "message_id": "m1",
    "from": "+919876543210",
    "to": "+14155550100",
    "ts": "2025-01-15T10:00:00Z",
    "text": "Hello"
  }'
```

**Responses:**

- `200` - Success
- `401` - Invalid signature
- `400` - Invalid request

### 3. Get Messages

**GET** `/messages`

Retrieve stored messages with filtering and pagination.

**Query Parameters:**

- `limit` (default: 50, max: 100) - Number of results
- `offset` (default: 0) - Skip N results
- `from` - Filter by sender phone number
- `since` - Filter by timestamp (ISO-8601)
- `q` - Search in message text

**Examples:**

```bash
# Get all messages
curl http://localhost:8000/messages

# Pagination
curl http://localhost:8000/messages?limit=10&offset=20

# Filter by sender
curl http://localhost:8000/messages?from=+919876543210

# Filter by time
curl http://localhost:8000/messages?since=2025-01-15T09:30:00Z

# Search text
curl http://localhost:8000/messages?q=Hello
```

**Response:**

```json
{
  "data": [...],
  "total": 100,
  "limit": 50,
  "offset": 0
}
```

### 4. Statistics

**GET** `/stats`

Get aggregated statistics about messages.

```bash
curl http://localhost:8000/stats
```

**Response:**

```json
{
  "total_messages": 150,
  "senders_count": 25,
  "messages_per_sender": {...},
  "first_message_ts": "2025-01-15T10:00:00Z",
  "last_message_ts": "2025-01-28T15:30:00Z"
}
```

### 5. Metrics

**GET** `/metrics`

Prometheus-compatible metrics endpoint.

```bash
curl http://localhost:8000/metrics
```

## Features

### HMAC Verification

- Uses HMAC-SHA256 for signature validation
- Signature calculated on raw request body
- Compared with `X-Signature` header
- Invalid signatures return `401 Unauthorized`

### Data Persistence

- SQLite database for message storage
- Automatic deduplication by `message_id`
- Efficient indexing for fast queries

### Pagination

- Offset-limit based pagination
- Default limit: 50, maximum: 100
- Results sorted by timestamp, then message_id

### Logging

- Structured JSON logging
- Logs include request metadata and message details
- Easy to parse and analyze

### Metrics

- HTTP request counts by path and status
- Webhook outcome tracking (created/duplicate/invalid)
- Request latency histograms

## Running Tests

```bash
# Run all tests
python tests/test_webhook.py
python tests/test_message.py
python tests/test_stats.py

# Or use Makefile (if available)
make test
```

## Makefile Commands

```bash
make up      # Start services with Docker
make down    # Stop and remove containers
make logs    # View service logs
make test    # Run test suite
```

## Environment Variables

| Variable         | Description                      | Example              |
| ---------------- | -------------------------------- | -------------------- |
| `WEBHOOK_SECRET` | Secret key for HMAC verification | `testsecret`         |
| `DATABASE_URL`   | SQLite database path             | `sqlite:///./app.db` |

## Project Structure

```
.
├── app/
│   ├── main.py           # FastAPI application
│   ├── models.py         # Data models
│   └── storage.py
|   |__ metrics.py
|   |__ config.py
|   |__ logging_utils.py
├── tests/
│   ├── test_webhook.py   # Webhook tests
│   ├── test_message.py   # Message retrieval tests
│   └── test_stats.py     # Statistics tests
├── docker-compose.yml    # Docker composition
├── Dockerfile           # Docker image definition
├── requirements.txt     # Python dependencies
└── README.md           # This file
```

## Troubleshooting

**Docker not working?**

- Use the local Python run method
- Ensure Docker daemon is running
- Check port 8000 is not already in use

**Database errors?**

- Verify `DATABASE_URL` environment variable
- Ensure write permissions for SQLite file
- Check disk space availability

**Signature verification failing?**

- Verify `WEBHOOK_SECRET` matches between sender and server
- Ensure signature is calculated on raw request body
- Check `X-Signature` header is present

#### Created By : Kashish
