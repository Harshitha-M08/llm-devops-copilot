# API Documentation

## Base URL
```
http://localhost:5000/api/v1
```

## Authentication

All protected endpoints require a Bearer token in the Authorization header:
```
Authorization: Bearer <your_jwt_token>
```

---

## Authentication Endpoints

### Register User
**POST** `/auth/register`

**Request Body:**
```json
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "SecurePassword123",
  "full_name": "John Doe",
  "role": "user",
  "department": "Engineering"
}
```

**Response:**
```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "user": {
      "id": 1,
      "username": "john_doe",
      "email": "john@example.com",
      "full_name": "John Doe",
      "role": "user",
      "department": "Engineering"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### Login
**POST** `/auth/login`

**Request Body:**
```json
{
  "email": "john@example.com",
  "password": "SecurePassword123"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "id": 1,
      "username": "john_doe",
      "email": "john@example.com",
      "full_name": "John Doe",
      "role": "user",
      "department": "Engineering",
      "avatar_url": null
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### Get Profile
**GET** `/auth/profile`

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "username": "john_doe",
    "email": "john@example.com",
    "full_name": "John Doe",
    "role": "user",
    "department": "Engineering",
    "avatar_url": null,
    "is_active": true,
    "created_at": "2025-01-01T00:00:00.000Z"
  }
}
```

### Update Profile
**PUT** `/auth/profile`

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "full_name": "John Updated Doe",
  "department": "DevOps",
  "avatar_url": "https://example.com/avatar.jpg"
}
```

### Change Password
**POST** `/auth/change-password`

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "currentPassword": "OldPassword123",
  "newPassword": "NewPassword456"
}
```

### Refresh Token
**POST** `/auth/refresh-token`

**Request Body:**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Logout
**POST** `/auth/logout`

**Request Body:**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## Approval Endpoints

### Create Approval Request
**POST** `/approvals`

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "title": "Deploy to Production",
  "description": "Request approval to deploy version 2.0 to production",
  "request_type": "deployment",
  "priority": "high",
  "approver_id": 2,
  "metadata": {
    "version": "2.0.0",
    "environment": "production"
  },
  "attachments": [
    {
      "name": "deployment-plan.pdf",
      "url": "https://example.com/files/deployment-plan.pdf"
    }
  ],
  "due_date": "2025-12-31T23:59:59Z"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Approval request created successfully",
  "data": {
    "id": 1,
    "title": "Deploy to Production",
    "description": "Request approval to deploy version 2.0 to production",
    "request_type": "deployment",
    "priority": "high",
    "status": "pending",
    "requester_id": 1,
    "approver_id": 2,
    "metadata": {...},
    "attachments": [...],
    "due_date": "2025-12-31T23:59:59.000Z",
    "submitted_at": "2025-01-15T10:30:00.000Z",
    "created_at": "2025-01-15T10:30:00.000Z"
  }
}
```

### Get All Approvals
**GET** `/approvals`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Parameters:**
- `status` - Filter by status (pending, approved, rejected, in_review)
- `request_type` - Filter by request type
- `priority` - Filter by priority (low, medium, high, urgent)
- `requester_id` - Filter by requester
- `approver_id` - Filter by approver
- `limit` - Number of results (default: 50)
- `offset` - Offset for pagination (default: 0)

**Example:**
```
GET /approvals?status=pending&limit=20&offset=0
```

### Get Approval by ID
**GET** `/approvals/:id`

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "Deploy to Production",
    "description": "...",
    "status": "pending",
    "requester_username": "john_doe",
    "requester_name": "John Doe",
    "approver_username": "jane_smith",
    "approver_name": "Jane Smith",
    "history": [
      {
        "id": 1,
        "action": "created",
        "old_status": null,
        "new_status": "pending",
        "username": "john_doe",
        "created_at": "2025-01-15T10:30:00.000Z"
      }
    ]
  }
}
```

### Update Approval
**PUT** `/approvals/:id`

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "title": "Updated Title",
  "description": "Updated description",
  "priority": "urgent"
}
```

### Review Approval (Approve/Reject)
**POST** `/approvals/:id/review`

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "action": "approve",
  "comments": "Approved after review. Good to go!"
}
```

Action can be: `approve` or `reject`

### Delete Approval
**DELETE** `/approvals/:id`

**Headers:**
```
Authorization: Bearer <token>
```

### Get My Approvals
**GET** `/approvals/my-approvals`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Parameters:**
- `status` - Filter by status
- `limit` - Number of results
- `offset` - Pagination offset

### Get Pending Approvals (for review)
**GET** `/approvals/pending`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Parameters:**
- `limit` - Number of results
- `offset` - Pagination offset

### Get Statistics
**GET** `/approvals/stats`

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "total": "100",
    "pending": "25",
    "approved": "60",
    "rejected": "10",
    "in_review": "5"
  }
}
```

### Search Approvals
**GET** `/approvals/search`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Parameters:**
- `q` - Search query (required)
- `status` - Filter by status
- `request_type` - Filter by request type
- `limit` - Number of results

**Example:**
```
GET /approvals/search?q=production&status=pending
```

---

## WebSocket Connection

### Connect to WebSocket
```javascript
const socket = io('http://localhost:5000', {
  auth: {
    token: 'your_jwt_token'
  }
});

// Listen for connection
socket.on('connected', (data) => {
  console.log('Connected:', data);
});

// Listen for new approvals
socket.on('new_approval', (approval) => {
  console.log('New approval received:', approval);
});

// Listen for approval updates
socket.on('approval_updated', (approval) => {
  console.log('Approval updated:', approval);
});

// Join an approval room
socket.emit('join_approval', approvalId);

// Leave an approval room
socket.emit('leave_approval', approvalId);
```

---

## Error Responses

### 400 Bad Request
```json
{
  "success": false,
  "message": "Validation error message"
}
```

### 401 Unauthorized
```json
{
  "success": false,
  "message": "Access token is required"
}
```

### 403 Forbidden
```json
{
  "success": false,
  "message": "You do not have permission to access this resource"
}
```

### 404 Not Found
```json
{
  "success": false,
  "message": "Resource not found"
}
```

### 500 Internal Server Error
```json
{
  "success": false,
  "message": "Internal server error",
  "error": "Detailed error (development only)"
}
```

---

## Status Codes

- **200** - Success
- **201** - Created
- **400** - Bad Request
- **401** - Unauthorized
- **403** - Forbidden
- **404** - Not Found
- **409** - Conflict
- **500** - Internal Server Error

---

## Request Types

Common request types:
- `deployment`
- `code_review`
- `access_request`
- `budget`
- `leave`
- `purchase`
- `change_request`
- `other`

## Priority Levels

- `low`
- `medium`
- `high`
- `urgent`

## Approval Status

- `pending` - Awaiting review
- `in_review` - Under review
- `approved` - Approved
- `rejected` - Rejected
