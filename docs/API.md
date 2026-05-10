# API Documentation - AI-Driven Hybrid Kubernetes System

## Table of Contents
1. [Overview](#overview)
2. [Authentication](#authentication)
3. [LLM Service API](#llm-service-api)
4. [Approval Dashboard API](#approval-dashboard-api)
5. [Request/Response Formats](#requestresponse-formats)
6. [Rate Limiting](#rate-limiting)
7. [Error Codes](#error-codes)
8. [Code Examples](#code-examples)

---

## Overview

The AI-Driven Hybrid Kubernetes System provides two main REST APIs:

1. **LLM Service API** (`https://api.example.com/llm`) - AI/ML operations, RAG queries, and document management
2. **Approval Dashboard API** (`https://api.example.com/api`) - User management, approval workflows, and notifications

All APIs use:
- **Protocol**: HTTPS only
- **Format**: JSON request/response bodies
- **Authentication**: JWT tokens (except public endpoints)
- **Versioning**: URL path versioning (e.g., `/api/v1/...`)

### Base URLs

- **Development**: `http://localhost:8000` (LLM) / `http://localhost:3000` (Approval)
- **Staging**: `https://api-staging.example.com`
- **Production**: `https://api.example.com`

### API Status

Check API health:
```bash
# LLM Service
GET /llm/health

# Approval Backend
GET /api/health
```

---

## Authentication

### Authentication Flow

```
1. User logs in with username/password
   POST /api/auth/login

2. Server returns JWT access token + refresh token
   Response: { "accessToken": "...", "refreshToken": "..." }

3. Client includes access token in subsequent requests
   Header: Authorization: Bearer <access-token>

4. When access token expires (15 min), use refresh token
   POST /api/auth/refresh

5. Server returns new access token
```

### JWT Token Structure

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "userId": "123",
    "username": "john.doe",
    "email": "john@example.com",
    "role": "approver",
    "iat": 1234567890,
    "exp": 1234568790
  },
  "signature": "..."
}
```

### Token Lifetimes

- **Access Token**: 15 minutes
- **Refresh Token**: 7 days

### Authorization Header

All authenticated requests must include:

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Role-Based Access Control (RBAC)

| Role      | Permissions                                           |
|-----------|-------------------------------------------------------|
| `admin`   | Full access to all endpoints                          |
| `approver`| Can approve/deny requests, view all data             |
| `viewer`  | Read-only access, cannot approve                      |
| `service` | Service-to-service authentication (internal)          |

---

## LLM Service API

Base URL: `/llm/api/v1`

### 1. Health Check

Check service health and configuration.

**Endpoint**: `GET /llm/health`

**Authentication**: None

**Response**:
```json
{
  "status": "healthy",
  "service": "llm-service",
  "version": "1.0.0",
  "providers": {
    "openai": "configured",
    "gemini": "not configured"
  }
}
```

**cURL Example**:
```bash
curl -X GET https://api.example.com/llm/health
```

---

### 2. Chat Completion

Get AI-generated response to a conversation.

**Endpoint**: `POST /llm/api/v1/chat`

**Authentication**: Required (Bearer token)

**Request Body**:
```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "What is the capital of France?"
    }
  ],
  "provider": "openai",
  "max_tokens": 500,
  "temperature": 0.7,
  "stream": false
}
```

**Request Parameters**:

| Parameter     | Type     | Required | Description                                    |
|---------------|----------|----------|------------------------------------------------|
| `messages`    | array    | Yes      | Array of message objects (role + content)      |
| `provider`    | string   | No       | LLM provider: "openai" or "gemini" (auto if omitted) |
| `max_tokens`  | integer  | No       | Maximum tokens in response (default: 1000)     |
| `temperature` | float    | No       | Sampling temperature 0.0-2.0 (default: 0.7)    |
| `stream`      | boolean  | No       | Enable streaming (default: false)              |

**Response**:
```json
{
  "response": "The capital of France is Paris.",
  "provider": "openai",
  "tokens_used": 25
}
```

**Status Codes**:
- `200 OK` - Success
- `400 Bad Request` - Invalid request format
- `401 Unauthorized` - Missing or invalid token
- `429 Too Many Requests` - Rate limit exceeded
- `500 Internal Server Error` - Server error

**cURL Example**:
```bash
curl -X POST https://api.example.com/llm/api/v1/chat \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is the capital of France?"}
    ],
    "provider": "openai",
    "temperature": 0.7
  }'
```

**Python Example**:
```python
import requests

url = "https://api.example.com/llm/api/v1/chat"
headers = {
    "Authorization": "Bearer YOUR_TOKEN",
    "Content-Type": "application/json"
}
payload = {
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is the capital of France?"}
    ],
    "provider": "openai",
    "temperature": 0.7
}

response = requests.post(url, json=payload, headers=headers)
print(response.json())
```

---

### 3. Streaming Chat Completion

Get streaming response for real-time updates.

**Endpoint**: `POST /llm/api/v1/chat/stream`

**Authentication**: Required

**Request Body**: Same as Chat Completion

**Response**: Server-Sent Events (SSE)

```
data: {"chunk": "The"}

data: {"chunk": " capital"}

data: {"chunk": " of"}

data: {"chunk": " France"}

data: {"chunk": " is"}

data: {"chunk": " Paris"}

data: {"chunk": "."}

data: [DONE]
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/llm/api/v1/chat/stream \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -N \
  -d '{
    "messages": [
      {"role": "user", "content": "Write a short poem"}
    ]
  }'
```

**JavaScript Example**:
```javascript
const eventSource = new EventSource(
  'https://api.example.com/llm/api/v1/chat/stream',
  {
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN',
      'Content-Type': 'application/json'
    },
    method: 'POST',
    body: JSON.stringify({
      messages: [
        { role: 'user', content: 'Write a short poem' }
      ]
    })
  }
);

eventSource.onmessage = (event) => {
  if (event.data === '[DONE]') {
    eventSource.close();
    return;
  }

  const data = JSON.parse(event.data);
  console.log(data.chunk);
};
```

---

### 4. Document Ingestion

Ingest documents into the vector store for RAG.

**Endpoint**: `POST /llm/api/v1/documents/ingest`

**Authentication**: Required

**Request Body**:
```json
{
  "text": "Your document text here. This will be chunked and embedded for semantic search.",
  "metadata": {
    "source": "user_upload",
    "document_id": "doc_123",
    "category": "technical",
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

**Request Parameters**:

| Parameter  | Type   | Required | Description                                 |
|------------|--------|----------|---------------------------------------------|
| `text`     | string | Yes      | Document text to ingest                     |
| `metadata` | object | No       | Custom metadata for filtering and retrieval |

**Response**:
```json
{
  "chunks_created": 15,
  "collection_name": "ai_documents",
  "status": "success"
}
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/llm/api/v1/documents/ingest \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Kubernetes is an open-source container orchestration platform...",
    "metadata": {
      "source": "documentation",
      "topic": "kubernetes"
    }
  }'
```

---

### 5. RAG Query

Perform Retrieval-Augmented Generation query.

**Endpoint**: `POST /llm/api/v1/rag/query`

**Authentication**: Required

**Request Body**:
```json
{
  "query": "What is Kubernetes?",
  "system_prompt": "You are a technical documentation assistant.",
  "top_k": 5
}
```

**Request Parameters**:

| Parameter       | Type    | Required | Description                                      |
|-----------------|---------|----------|--------------------------------------------------|
| `query`         | string  | Yes      | User question                                    |
| `system_prompt` | string  | No       | Custom system prompt (default: generic prompt)   |
| `top_k`         | integer | No       | Number of documents to retrieve (default: 5)     |

**Response**:
```json
{
  "answer": "Kubernetes is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.",
  "context_used": 5,
  "provider": "openai"
}
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/llm/api/v1/rag/query \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Kubernetes?",
    "top_k": 5
  }'
```

---

### 6. Document Retrieval

Retrieve relevant documents without generating an answer.

**Endpoint**: `POST /llm/api/v1/rag/retrieve`

**Authentication**: Required

**Request Body**:
```json
{
  "query": "kubernetes deployment",
  "top_k": 10
}
```

**Response**:
```json
{
  "documents": [
    {
      "text": "Kubernetes deployments describe the desired state...",
      "score": 0.95,
      "metadata": {
        "source": "documentation",
        "topic": "kubernetes"
      }
    },
    {
      "text": "A Deployment provides declarative updates...",
      "score": 0.89,
      "metadata": {
        "source": "user_upload",
        "document_id": "doc_456"
      }
    }
  ],
  "count": 2
}
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/llm/api/v1/rag/retrieve \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "kubernetes deployment",
    "top_k": 10
  }'
```

---

### 7. Async Task Submission

Submit long-running tasks to the message queue.

**Endpoint**: `POST /llm/api/v1/tasks/submit`

**Authentication**: Required

**Request Body**:
```json
{
  "task_type": "batch_document_ingestion",
  "payload": {
    "documents": [
      {
        "text": "Document 1 content...",
        "metadata": {"source": "batch_1"}
      },
      {
        "text": "Document 2 content...",
        "metadata": {"source": "batch_1"}
      }
    ]
  }
}
```

**Response**:
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "message": "Task 550e8400-e29b-41d4-a716-446655440000 submitted successfully"
}
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/llm/api/v1/tasks/submit \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "task_type": "batch_document_ingestion",
    "payload": {
      "documents": [...]
    }
  }'
```

---

### 8. Collection Management

Get information about vector collections.

**Endpoint**: `GET /llm/api/v1/collections/info`

**Authentication**: Required

**Response**:
```json
{
  "name": "ai_documents",
  "vectors_count": 15234,
  "points_count": 15234,
  "status": "green"
}
```

**Delete Collection**:

**Endpoint**: `DELETE /llm/api/v1/collections/{collection_name}`

**Authentication**: Required (Admin only)

**Response**:
```json
{
  "status": "success",
  "message": "Collection ai_documents deleted"
}
```

---

## Approval Dashboard API

Base URL: `/api/v1`

### 1. Authentication

#### Login

**Endpoint**: `POST /api/auth/login`

**Authentication**: None

**Request Body**:
```json
{
  "username": "john.doe",
  "password": "SecurePassword123!"
}
```

**Response**:
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 123,
    "username": "john.doe",
    "email": "john@example.com",
    "role": "approver"
  }
}
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john.doe",
    "password": "SecurePassword123!"
  }'
```

#### Logout

**Endpoint**: `POST /api/auth/logout`

**Authentication**: Required

**Request Body**:
```json
{
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response**:
```json
{
  "status": "success",
  "message": "Logged out successfully"
}
```

#### Refresh Token

**Endpoint**: `POST /api/auth/refresh`

**Authentication**: None (uses refresh token)

**Request Body**:
```json
{
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response**:
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

### 2. User Management

#### Get Current User

**Endpoint**: `GET /api/users/me`

**Authentication**: Required

**Response**:
```json
{
  "id": 123,
  "username": "john.doe",
  "email": "john@example.com",
  "role": "approver",
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-15T08:30:00Z"
}
```

**cURL Example**:
```bash
curl -X GET https://api.example.com/api/users/me \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### List Users

**Endpoint**: `GET /api/users`

**Authentication**: Required (Admin only)

**Query Parameters**:

| Parameter | Type    | Description                     |
|-----------|---------|---------------------------------|
| `page`    | integer | Page number (default: 1)        |
| `limit`   | integer | Items per page (default: 20)    |
| `role`    | string  | Filter by role                  |

**Response**:
```json
{
  "users": [
    {
      "id": 123,
      "username": "john.doe",
      "email": "john@example.com",
      "role": "approver",
      "createdAt": "2025-01-01T10:00:00Z"
    },
    {
      "id": 124,
      "username": "jane.smith",
      "email": "jane@example.com",
      "role": "viewer",
      "createdAt": "2025-01-02T11:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

**cURL Example**:
```bash
curl -X GET "https://api.example.com/api/users?page=1&limit=20&role=approver" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### Create User

**Endpoint**: `POST /api/users`

**Authentication**: Required (Admin only)

**Request Body**:
```json
{
  "username": "new.user",
  "email": "new.user@example.com",
  "password": "SecurePassword123!",
  "role": "viewer"
}
```

**Response**:
```json
{
  "id": 125,
  "username": "new.user",
  "email": "new.user@example.com",
  "role": "viewer",
  "createdAt": "2025-01-15T10:30:00Z"
}
```

#### Update User

**Endpoint**: `PUT /api/users/{userId}`

**Authentication**: Required (Admin or own profile)

**Request Body**:
```json
{
  "email": "updated.email@example.com",
  "role": "approver"
}
```

**Response**:
```json
{
  "id": 123,
  "username": "john.doe",
  "email": "updated.email@example.com",
  "role": "approver",
  "updatedAt": "2025-01-15T11:00:00Z"
}
```

#### Delete User

**Endpoint**: `DELETE /api/users/{userId}`

**Authentication**: Required (Admin only)

**Response**:
```json
{
  "status": "success",
  "message": "User deleted successfully"
}
```

---

### 3. Approval Management

#### List Approvals

**Endpoint**: `GET /api/approvals`

**Authentication**: Required

**Query Parameters**:

| Parameter | Type   | Description                                     |
|-----------|--------|-------------------------------------------------|
| `status`  | string | Filter by status: pending, approved, denied     |
| `page`    | int    | Page number (default: 1)                        |
| `limit`   | int    | Items per page (default: 20)                    |

**Response**:
```json
{
  "approvals": [
    {
      "id": 1001,
      "requestType": "deployment_approval",
      "requestData": {
        "service": "llm-service",
        "version": "v2.0.0",
        "environment": "production"
      },
      "status": "pending",
      "requestedBy": {
        "id": 122,
        "username": "system.bot"
      },
      "createdAt": "2025-01-15T09:00:00Z",
      "updatedAt": "2025-01-15T09:00:00Z"
    },
    {
      "id": 1000,
      "requestType": "scaling_approval",
      "requestData": {
        "service": "worker-service",
        "currentReplicas": 3,
        "targetReplicas": 10
      },
      "status": "approved",
      "requestedBy": {
        "id": 122,
        "username": "system.bot"
      },
      "approvedBy": {
        "id": 123,
        "username": "john.doe"
      },
      "createdAt": "2025-01-15T08:00:00Z",
      "approvedAt": "2025-01-15T08:15:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "totalPages": 8
  }
}
```

**cURL Example**:
```bash
curl -X GET "https://api.example.com/api/approvals?status=pending&page=1&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### Get Approval Details

**Endpoint**: `GET /api/approvals/{approvalId}`

**Authentication**: Required

**Response**:
```json
{
  "id": 1001,
  "requestType": "deployment_approval",
  "requestData": {
    "service": "llm-service",
    "version": "v2.0.0",
    "environment": "production",
    "changes": [
      "Updated AI model from GPT-3.5 to GPT-4",
      "Added new RAG pipeline features",
      "Increased memory limits"
    ]
  },
  "status": "pending",
  "requestedBy": {
    "id": 122,
    "username": "system.bot",
    "email": "bot@example.com"
  },
  "createdAt": "2025-01-15T09:00:00Z",
  "updatedAt": "2025-01-15T09:00:00Z",
  "expiresAt": "2025-01-15T21:00:00Z"
}
```

#### Create Approval Request

**Endpoint**: `POST /api/approvals`

**Authentication**: Required

**Request Body**:
```json
{
  "requestType": "deployment_approval",
  "requestData": {
    "service": "llm-service",
    "version": "v2.0.0",
    "environment": "production",
    "changes": [
      "Updated AI model",
      "Performance improvements"
    ]
  }
}
```

**Response**:
```json
{
  "id": 1002,
  "requestType": "deployment_approval",
  "requestData": {...},
  "status": "pending",
  "requestedBy": {
    "id": 123,
    "username": "john.doe"
  },
  "createdAt": "2025-01-15T10:30:00Z"
}
```

#### Approve Request

**Endpoint**: `POST /api/approvals/{approvalId}/approve`

**Authentication**: Required (Approver or Admin role)

**Request Body**:
```json
{
  "comment": "Approved after review. Changes look good."
}
```

**Response**:
```json
{
  "id": 1001,
  "status": "approved",
  "approvedBy": {
    "id": 123,
    "username": "john.doe"
  },
  "approvedAt": "2025-01-15T10:30:00Z",
  "comment": "Approved after review. Changes look good."
}
```

**cURL Example**:
```bash
curl -X POST https://api.example.com/api/approvals/1001/approve \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "comment": "Approved after review."
  }'
```

#### Deny Request

**Endpoint**: `POST /api/approvals/{approvalId}/deny`

**Authentication**: Required (Approver or Admin role)

**Request Body**:
```json
{
  "reason": "Security concerns. Please address the vulnerabilities first."
}
```

**Response**:
```json
{
  "id": 1001,
  "status": "denied",
  "deniedBy": {
    "id": 123,
    "username": "john.doe"
  },
  "deniedAt": "2025-01-15T10:30:00Z",
  "reason": "Security concerns. Please address the vulnerabilities first."
}
```

---

### 4. Notifications

#### Get Notifications

**Endpoint**: `GET /api/notifications`

**Authentication**: Required

**Query Parameters**:

| Parameter | Type    | Description                         |
|-----------|---------|-------------------------------------|
| `unread`  | boolean | Filter unread notifications         |
| `page`    | integer | Page number                         |
| `limit`   | integer | Items per page                      |

**Response**:
```json
{
  "notifications": [
    {
      "id": 5001,
      "type": "approval_request",
      "title": "New Approval Request",
      "message": "Deployment approval requested for llm-service v2.0.0",
      "data": {
        "approvalId": 1001
      },
      "read": false,
      "createdAt": "2025-01-15T09:00:00Z"
    },
    {
      "id": 5000,
      "type": "approval_completed",
      "title": "Approval Completed",
      "message": "Your scaling request has been approved",
      "data": {
        "approvalId": 1000
      },
      "read": true,
      "createdAt": "2025-01-15T08:15:00Z"
    }
  ],
  "unreadCount": 3,
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45
  }
}
```

#### Mark Notification as Read

**Endpoint**: `PUT /api/notifications/{notificationId}/read`

**Authentication**: Required

**Response**:
```json
{
  "status": "success",
  "message": "Notification marked as read"
}
```

#### Subscribe to WebSocket Notifications

**Endpoint**: `WS /api/notifications/subscribe`

**Authentication**: Required (token in query param or connection header)

**Connection**:
```javascript
const ws = new WebSocket('wss://api.example.com/api/notifications/subscribe?token=YOUR_TOKEN');

ws.onmessage = (event) => {
  const notification = JSON.parse(event.data);
  console.log('New notification:', notification);
};
```

**Message Format**:
```json
{
  "type": "approval_request",
  "data": {
    "id": 1002,
    "requestType": "deployment_approval",
    "status": "pending"
  }
}
```

---

## Request/Response Formats

### Standard Response Format

All API responses follow this structure:

**Success Response**:
```json
{
  "data": { ... },
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

**Error Response**:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Email format is invalid"
      }
    ]
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

### Pagination

List endpoints support pagination:

**Request**:
```
GET /api/users?page=2&limit=20
```

**Response**:
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 156,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": true
  }
}
```

### Filtering and Sorting

**Filtering**:
```
GET /api/approvals?status=pending&requestType=deployment_approval
```

**Sorting**:
```
GET /api/approvals?sort=-createdAt  # Descending
GET /api/approvals?sort=createdAt   # Ascending
```

---

## Rate Limiting

### Rate Limit Headers

All responses include rate limit headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995
X-RateLimit-Reset: 1642252800
```

### Rate Limits

| Endpoint Type      | Limit              | Window  |
|--------------------|--------------------|---------|
| Authentication     | 10 requests        | 1 minute|
| LLM API            | 100 requests       | 1 hour  |
| Approval API       | 1000 requests      | 1 hour  |
| Document Ingestion | 10 requests        | 1 hour  |

### Rate Limit Exceeded Response

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please try again later.",
    "retryAfter": 3600
  }
}
```

**HTTP Status**: `429 Too Many Requests`

---

## Error Codes

### HTTP Status Codes

| Code | Description                                              |
|------|----------------------------------------------------------|
| 200  | OK - Request succeeded                                   |
| 201  | Created - Resource created successfully                  |
| 204  | No Content - Request succeeded, no content to return     |
| 400  | Bad Request - Invalid request format                     |
| 401  | Unauthorized - Missing or invalid authentication         |
| 403  | Forbidden - Insufficient permissions                     |
| 404  | Not Found - Resource not found                           |
| 409  | Conflict - Resource conflict (e.g., duplicate)           |
| 422  | Unprocessable Entity - Validation error                  |
| 429  | Too Many Requests - Rate limit exceeded                  |
| 500  | Internal Server Error - Server error                     |
| 502  | Bad Gateway - Upstream service error                     |
| 503  | Service Unavailable - Service temporarily unavailable    |

### Application Error Codes

| Code                    | Description                                    |
|-------------------------|------------------------------------------------|
| `VALIDATION_ERROR`      | Request validation failed                      |
| `AUTHENTICATION_ERROR`  | Authentication failed                          |
| `AUTHORIZATION_ERROR`   | Insufficient permissions                       |
| `NOT_FOUND`             | Resource not found                             |
| `DUPLICATE_RESOURCE`    | Resource already exists                        |
| `RATE_LIMIT_EXCEEDED`   | Rate limit exceeded                            |
| `LLM_PROVIDER_ERROR`    | LLM provider returned error                    |
| `DATABASE_ERROR`        | Database operation failed                      |
| `QUEUE_ERROR`           | Message queue operation failed                 |
| `INTERNAL_ERROR`        | Internal server error                          |

### Error Response Examples

**Validation Error**:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      },
      {
        "field": "password",
        "message": "Password must be at least 8 characters"
      }
    ]
  }
}
```

**Authentication Error**:
```json
{
  "error": {
    "code": "AUTHENTICATION_ERROR",
    "message": "Invalid or expired token"
  }
}
```

**Authorization Error**:
```json
{
  "error": {
    "code": "AUTHORIZATION_ERROR",
    "message": "You don't have permission to perform this action"
  }
}
```

---

## Code Examples

### Python SDK Example

```python
import requests

class AISystemClient:
    def __init__(self, base_url, api_key=None):
        self.base_url = base_url
        self.api_key = api_key
        self.session = requests.Session()
        if api_key:
            self.session.headers.update({
                'Authorization': f'Bearer {api_key}'
            })

    def login(self, username, password):
        response = self.session.post(
            f'{self.base_url}/api/auth/login',
            json={'username': username, 'password': password}
        )
        data = response.json()
        self.api_key = data['accessToken']
        self.session.headers.update({
            'Authorization': f'Bearer {self.api_key}'
        })
        return data

    def chat(self, messages, provider='openai', temperature=0.7):
        response = self.session.post(
            f'{self.base_url}/llm/api/v1/chat',
            json={
                'messages': messages,
                'provider': provider,
                'temperature': temperature
            }
        )
        return response.json()

    def ingest_document(self, text, metadata=None):
        response = self.session.post(
            f'{self.base_url}/llm/api/v1/documents/ingest',
            json={'text': text, 'metadata': metadata}
        )
        return response.json()

    def rag_query(self, query, top_k=5):
        response = self.session.post(
            f'{self.base_url}/llm/api/v1/rag/query',
            json={'query': query, 'top_k': top_k}
        )
        return response.json()

    def get_approvals(self, status=None, page=1, limit=20):
        params = {'page': page, 'limit': limit}
        if status:
            params['status'] = status
        response = self.session.get(
            f'{self.base_url}/api/approvals',
            params=params
        )
        return response.json()

    def approve_request(self, approval_id, comment=None):
        response = self.session.post(
            f'{self.base_url}/api/approvals/{approval_id}/approve',
            json={'comment': comment}
        )
        return response.json()

# Usage
client = AISystemClient('https://api.example.com')
client.login('john.doe', 'password123')

# Chat
result = client.chat([
    {'role': 'user', 'content': 'What is Kubernetes?'}
])
print(result['response'])

# RAG Query
answer = client.rag_query('Explain Kubernetes deployments')
print(answer['answer'])

# Get pending approvals
approvals = client.get_approvals(status='pending')
for approval in approvals['approvals']:
    print(f"Approval {approval['id']}: {approval['requestType']}")
```

### JavaScript/Node.js Example

```javascript
const axios = require('axios');

class AISystemClient {
  constructor(baseUrl, apiKey = null) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
    this.client = axios.create({
      baseURL: baseUrl,
      headers: apiKey ? { 'Authorization': `Bearer ${apiKey}` } : {}
    });
  }

  async login(username, password) {
    const response = await this.client.post('/api/auth/login', {
      username,
      password
    });
    this.apiKey = response.data.accessToken;
    this.client.defaults.headers['Authorization'] = `Bearer ${this.apiKey}`;
    return response.data;
  }

  async chat(messages, options = {}) {
    const response = await this.client.post('/llm/api/v1/chat', {
      messages,
      provider: options.provider || 'openai',
      temperature: options.temperature || 0.7,
      max_tokens: options.maxTokens
    });
    return response.data;
  }

  async ingestDocument(text, metadata = null) {
    const response = await this.client.post('/llm/api/v1/documents/ingest', {
      text,
      metadata
    });
    return response.data;
  }

  async ragQuery(query, topK = 5) {
    const response = await this.client.post('/llm/api/v1/rag/query', {
      query,
      top_k: topK
    });
    return response.data;
  }

  async getApprovals(status = null, page = 1, limit = 20) {
    const params = { page, limit };
    if (status) params.status = status;

    const response = await this.client.get('/api/approvals', { params });
    return response.data;
  }

  async approveRequest(approvalId, comment = null) {
    const response = await this.client.post(
      `/api/approvals/${approvalId}/approve`,
      { comment }
    );
    return response.data;
  }
}

// Usage
(async () => {
  const client = new AISystemClient('https://api.example.com');
  await client.login('john.doe', 'password123');

  // Chat
  const chatResult = await client.chat([
    { role: 'user', content: 'What is Kubernetes?' }
  ]);
  console.log(chatResult.response);

  // RAG Query
  const ragResult = await client.ragQuery('Explain Kubernetes deployments');
  console.log(ragResult.answer);

  // Get approvals
  const approvals = await client.getApprovals('pending');
  approvals.approvals.forEach(approval => {
    console.log(`Approval ${approval.id}: ${approval.requestType}`);
  });
})();
```

### cURL Examples Collection

```bash
#!/bin/bash

# Configuration
BASE_URL="https://api.example.com"
USERNAME="john.doe"
PASSWORD="password123"

# Login and get token
TOKEN=$(curl -s -X POST "$BASE_URL/api/auth/login" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$USERNAME\",\"password\":\"$PASSWORD\"}" \
  | jq -r '.accessToken')

echo "Token: $TOKEN"

# Chat completion
curl -X POST "$BASE_URL/llm/api/v1/chat" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "What is Kubernetes?"}
    ]
  }'

# Ingest document
curl -X POST "$BASE_URL/llm/api/v1/documents/ingest" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Kubernetes is a container orchestration platform...",
    "metadata": {"source": "documentation"}
  }'

# RAG query
curl -X POST "$BASE_URL/llm/api/v1/rag/query" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Kubernetes?",
    "top_k": 5
  }'

# Get approvals
curl -X GET "$BASE_URL/api/approvals?status=pending" \
  -H "Authorization: Bearer $TOKEN"

# Approve request
curl -X POST "$BASE_URL/api/approvals/1001/approve" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "comment": "Approved after review"
  }'
```

### Postman Collection

Create a Postman collection with these requests:

1. **Environment Variables**:
   - `base_url`: `https://api.example.com`
   - `token`: (will be set after login)

2. **Pre-request Script** (for authenticated requests):
```javascript
if (!pm.environment.get('token')) {
  throw new Error('Please login first to get token');
}
```

3. **Collection Structure**:
```
AI System API
├── Authentication
│   ├── Login
│   ├── Logout
│   └── Refresh Token
├── LLM Service
│   ├── Health Check
│   ├── Chat Completion
│   ├── Streaming Chat
│   ├── Document Ingestion
│   ├── RAG Query
│   └── Document Retrieval
└── Approval Dashboard
    ├── List Approvals
    ├── Get Approval Details
    ├── Approve Request
    ├── Deny Request
    ├── List Users
    └── Get Notifications
```

---

## Conclusion

This API documentation provides comprehensive coverage of all endpoints in the AI-Driven Hybrid Kubernetes System. For additional support:

- Check the source code documentation
- Review the examples in the `/examples` directory
- Join our community Slack channel
- Submit issues on GitHub

**API Version**: v1.0.0
**Last Updated**: 2025-01-15
