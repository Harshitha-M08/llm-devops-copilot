# Quick Start Guide

Get the Approval Dashboard Backend up and running in minutes!

## Prerequisites

- Node.js 18 or higher
- PostgreSQL 12 or higher
- npm or yarn

## Setup Steps

### 1. Install Dependencies

```bash
npm install
```

### 2. Set Up Database

Create a PostgreSQL database:

```sql
CREATE DATABASE approval_dashboard;
```

### 3. Configure Environment

Copy the example environment file and configure it:

```bash
cp .env.example .env
```

Edit `.env` and update:
- Database credentials (`DB_HOST`, `DB_USER`, `DB_PASSWORD`)
- JWT secret (`JWT_SECRET` - use a strong random string)
- Email settings (optional for testing)

### 4. Run Database Migration

This will create all tables and a default admin user:

```bash
npm run migrate
```

**Default Admin Credentials:**
- Email: `admin@approvaldashboard.com`
- Password: `Admin@123`

**Change the admin password immediately after first login!**

### 5. Start the Server

**Development mode** (with auto-reload):
```bash
npm run dev
```

**Production mode**:
```bash
npm start
```

The server will start on `http://localhost:5000`

### 6. Test the API

**Health Check:**
```bash
curl http://localhost:5000/health
```

**Login:**
```bash
curl -X POST http://localhost:5000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@approvaldashboard.com","password":"Admin@123"}'
```

## Docker Setup (Alternative)

If you prefer Docker:

```bash
# Start all services (backend + PostgreSQL)
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

## Testing with Postman

1. Import the Postman collection: `Approval_Dashboard_API.postman_collection.json`
2. Set the `baseUrl` variable to `http://localhost:5000/api/v1`
3. Login to get a token
4. Set the `token` variable with your JWT token
5. Start testing endpoints!

## Quick Test Flow

### 1. Register a New User

```bash
curl -X POST http://localhost:5000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "User@123",
    "full_name": "John Doe",
    "role": "user",
    "department": "Engineering"
  }'
```

### 2. Login

```bash
curl -X POST http://localhost:5000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"User@123"}'
```

Save the `token` from the response.

### 3. Create an Approval Request

```bash
curl -X POST http://localhost:5000/api/v1/approvals \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Deploy to Production",
    "description": "Request to deploy v2.0",
    "request_type": "deployment",
    "priority": "high",
    "approver_id": 1
  }'
```

### 4. Get All Approvals

```bash
curl http://localhost:5000/api/v1/approvals \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### 5. Get Approval Statistics

```bash
curl http://localhost:5000/api/v1/approvals/stats \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## WebSocket Testing

You can test WebSocket functionality using JavaScript in the browser console or a WebSocket client:

```javascript
// Install socket.io-client in your frontend
import { io } from 'socket.io-client';

const socket = io('http://localhost:5000', {
  auth: {
    token: 'YOUR_JWT_TOKEN'
  }
});

socket.on('connected', (data) => {
  console.log('Connected:', data);
});

socket.on('new_approval', (approval) => {
  console.log('New approval:', approval);
});

socket.on('approval_approved', (approval) => {
  console.log('Approval approved:', approval);
});
```

## Troubleshooting

### Database Connection Failed
- Ensure PostgreSQL is running
- Check database credentials in `.env`
- Verify database exists

### Port Already in Use
- Change `PORT` in `.env` to a different port
- Or stop the service using port 5000

### JWT Errors
- Ensure `JWT_SECRET` is set in `.env`
- Token may have expired - login again

### Email Not Sending
- Email is optional for testing
- Configure SMTP settings or use SendGrid
- Check email provider credentials

## Next Steps

1. **Change Admin Password**: Login and change the default admin password
2. **Create Users**: Register users and approvers
3. **Test Approval Flow**: Create, review, and manage approvals
4. **Integrate Frontend**: Connect your frontend application
5. **Configure Email**: Set up email notifications for production

## API Documentation

Full API documentation is available in `API_DOCS.md`

## Production Deployment

Before deploying to production:

1. Change all default passwords
2. Use strong JWT secret
3. Enable HTTPS
4. Configure proper CORS origin
5. Set up email notifications
6. Enable logging
7. Set up database backups
8. Use environment-specific `.env` files

## Support

For more information, see:
- `README.md` - Complete documentation
- `API_DOCS.md` - API reference
- `package.json` - Available scripts

## Common Commands

```bash
# Install dependencies
npm install

# Run migrations
npm run migrate

# Start development server
npm run dev

# Start production server
npm start

# Run tests
npm test

# Lint code
npm run lint

# Docker commands
docker-compose up -d          # Start services
docker-compose logs -f        # View logs
docker-compose down           # Stop services
docker-compose ps             # List services
```

---

**You're all set!** Start building amazing approval workflows! 🚀
