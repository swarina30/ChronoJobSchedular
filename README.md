PR submission 
Distributed Job Scheduler
A high-performance, horizontally scalable distributed job scheduling system built with .NET 8.0, MySQL, Redis, and Docker. Designed to handle billions of jobs per day with queue-based dispatch, Redis heartbeat monitoring, and horizontal scalability.

🏗️ Architecture
Services
Job.Api: REST API for job management (Authentication with JWT, CRUD operations, job run history)
Job.Scheduler: Scheduler service (creates future runs, dispatches to queues, monitors heartbeats)
Job.Worker: Worker service (horizontally scaled, executes Python jobs)
Job.Monitor: Health monitoring service (health checks, metrics)
JobScheduler.Migrations: Database migration service
Shared Libraries
JobScheduler.Contracts: DTOs, Enums, Interfaces
JobScheduler.Data: EF Core DbContext, Repositories
JobScheduler.Infrastructure: Auth, Redis, Queue, Scheduling, Retry Strategies
🚀 Quick Start with Docker
One command to run the full stack (MySQL, Redis, migrations, API, Scheduler, Worker, Monitor):

Prerequisites
Docker and Docker Compose
1. Clone and run
git clone https://github.com/tarun200012/RemoteChronoJobSchedular.git
cd RemoteChronoJobSchedular
docker-compose up -d --build
This will:

Start MySQL 8.0 (port 3306) and Redis 7 (port 6379)
Run migrations (creates job_run, partitions; init scripts create users, job_definition, job_schedule_state)
Start Job.Api (port 5136), Job.Scheduler, Job.Worker, Job.Monitor (port 5137)
Wait ~30–60 seconds for MySQL to become healthy and migrations to complete, then:

2. Open Swagger and test
API: http://localhost:5136/swagger
Monitor: http://localhost:5137/swagger
3. Quick smoke test
# Sign up
curl -X POST http://localhost:5136/api/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"password123"}'

# Login (copy the token from the response)
curl -X POST http://localhost:5136/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Create a job (replace YOUR_JWT_TOKEN)
curl -X POST http://localhost:5136/api/jobs \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jobType":"Recurring","cronExpr":"*/1 * * * *","scriptContent":"print(\"Hello!\")","timeoutSeconds":30,"retryPolicy":{"maxAttempts":3,"strategy":"ExponentialBackoff","initialDelaySeconds":5,"maxDelaySeconds":300,"multiplier":2}}'
Then check Monitor → http://localhost:5137/api/health and http://localhost:5137/api/health/metrics for jobs and runs.

Stop
docker-compose down
# Optional: remove data
docker-compose down -v
📋 API Usage
1. Sign Up / Login
# Sign up
curl -X POST http://localhost:5136/api/auth/signup \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "email": "test@example.com",
    "password": "password123"
  }'

# Login
curl -X POST http://localhost:5136/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123"
  }'
Response:

{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "userId": "guid-here",
  "email": "test@example.com",
  "role": "User"
}
2. Create a Job
TOKEN="your-jwt-token-here"

# Create a recurring job (every minute)
curl -X POST http://localhost:5136/api/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "jobType": "Recurring",
    "cronExpr": "*/1 * * * *",
    "scriptContent": "print(\"Hello, World!\")",
    "timeoutSeconds": 30,
    "retryPolicy": {
      "maxAttempts": 3,
      "strategy": "ExponentialBackoff",
      "initialDelaySeconds": 5,
      "maxDelaySeconds": 300,
      "multiplier": 2
    }
  }'

# Create a one-time job
curl -X POST http://localhost:5136/api/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "jobType": "OneTime",
    "scheduledAt": "2026-01-26T10:00:00Z",
    "scriptContent": "print(\"One-time job\")",
    "timeoutSeconds": 60
  }'
3. View Job Runs
# Get all runs for a job
curl -X GET "http://localhost:5136/api/jobs/{jobId}/runs" \
  -H "Authorization: Bearer $TOKEN"

# Get specific run
curl -X GET "http://localhost:5136/api/jobs/{jobId}/runs/{runId}" \
  -H "Authorization: Bearer $TOKEN"
4. Manage Jobs
# List all jobs
curl -X GET "http://localhost:5136/api/jobs?page=1&pageSize=20" \
  -H "Authorization: Bearer $TOKEN"

# Pause a recurring job
curl -X PATCH "http://localhost:5136/api/jobs/{jobId}/pause" \
  -H "Authorization: Bearer $TOKEN"

# Resume a recurring job
curl -X PATCH "http://localhost:5136/api/jobs/{jobId}/resume" \
  -H "Authorization: Bearer $TOKEN"

# Update a job
curl -X PUT "http://localhost:5136/api/jobs/{jobId}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "cronExpr": "*/5 * * * *",
    "version": 1
  }'

# Delete a job
curl -X DELETE "http://localhost:5136/api/jobs/{jobId}" \
  -H "Authorization: Bearer $TOKEN"
🎯 Features
✅ Core Features
Job Types: One-time and Recurring (cron-based)
Python Script Execution: Workers execute Python scripts with full library support
Retry Policies: Exponential, Linear, and Fixed backoff strategies
Job Lifecycle: Create, Update, Pause, Resume, Delete
Job Run History: View execution history with detailed status
Authentication & Authorization: JWT-based with role-based access (Admin, User, Viewer)
User Management: Sign up, login, user roles
✅ Scalability Features
Horizontal Scaling: Multiple scheduler and worker instances
Queue-based Dispatch: Redis Lists for job distribution
Heartbeat Monitoring: Redis ZSET for worker liveness tracking
Database Partitioning: Hourly partitioning for job_run table
Batch Operations: Optimized database operations for high throughput
Optimistic Locking: Version-based concurrency control
✅ Reliability Features
CAS Updates: Compare-and-swap prevents duplicate execution
Retry Mechanisms: Configurable retry policies with backoff
Error Handling: Comprehensive error handling and logging
Health Monitoring: Dedicated monitoring service with metrics
Soft Deletes: Data retention with soft delete support
🏛️ System Design
Data Flow
Job Creation: User creates job via API → Stored in job_definition
Run Creation: Scheduler (FutureRunCreator) creates job_run entries for recurring jobs
Dispatch: Scheduler (RunDispatcher) moves pending runs to Redis queues
Execution: Workers pop from queues, execute Python scripts, update status
Monitoring: HeartbeatMonitor tracks running jobs via Redis ZSET
Database Schema
job_definition: Job metadata (cold table, immutable)
job_schedule_state: Scheduler cursor for recurring jobs (hot, bounded)
job_run: Execution records (hot, partitioned hourly by scheduled_at)
users: User accounts and authentication
Design Patterns
Builder Pattern: Fluent job creation (JobDefinitionBuilder)
Strategy Pattern: Retry policies (IRetryStrategy)
Repository Pattern: Data access abstraction
Background Services: Long-running loops in Scheduler/Worker
🐳 Docker Setup
Docker Compose Services
services:
  mysql:        # MySQL 8.0 database
  redis:        # Redis 7 for queues and heartbeats
  api:          # Job.Api service
  scheduler:    # Job.Scheduler service
  worker:       # Job.Worker service
  monitor:      # Job.Monitor service
  migrations:   # Database migration service
Environment Variables
Create .env file (optional):

MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=job_scheduler_dev
REDIS_PASSWORD=
JWT_SECRET_KEY=YourSuperSecretKeyForDevelopmentOnly-Minimum32Characters
Docker Commands
# Start everything (build + run; migrations run automatically)
docker-compose up -d --build

# View logs
docker-compose logs -f api
docker-compose logs -f scheduler
docker-compose logs -f worker

# Stop all services
docker-compose down

# Stop and remove volumes (data will be deleted)
docker-compose down -v
🔧 Configuration
Cron Expressions
Important: NCrontab only supports 5-field cron expressions (no seconds).

✅ Supported:

*/1 * * * * - Every minute
*/5 * * * * - Every 5 minutes
0 * * * * - Every hour
0 9 * * * - Every day at 9 AM
❌ Not Supported:

*/10 * * * * * - 6 fields (with seconds)
Retry Policies
{
  "maxAttempts": 3,
  "strategy": "ExponentialBackoff",  // or "LinearBackoff", "FixedBackoff"
  "initialDelaySeconds": 5,
  "maxDelaySeconds": 300,
  "multiplier": 2  // Only for ExponentialBackoff
}
📊 Monitoring
Health Endpoints
# Basic health check
curl http://localhost:5137/api/health

# Detailed metrics
curl http://localhost:5137/api/health/metrics
Metrics Available
Total jobs (Active, Paused, Deleted)
Job runs by status (Pending, Queued, Running, Completed, Failed)
Queue lengths
Worker heartbeats
Database connection status
🧪 Testing
Manual Testing with Swagger
Open http://localhost:5136/swagger
Click "Authorize" button
Enter token: Bearer your-jwt-token
Test endpoints interactively
Example Test Flow
Sign up / Login → Get JWT token
Create a recurring job with cron */1 * * * *
Wait 1-2 minutes
Check runs: GET /api/jobs/{jobId}/runs
Verify runs are being created and executed
📁 Project Structure
job-scheduler/
├── docker-compose.yml          # Docker Compose configuration
├── Dockerfile                  # Multi-stage build for all services
├── services/
│   ├── Job.Api/                # REST API service
│   ├── Job.Scheduler/          # Scheduler service
│   ├── Job.Worker/             # Worker service
│   ├── Job.Monitor/            # Health monitoring service
│   └── JobScheduler.Migrations/ # Migration service
├── shared/
│   ├── JobScheduler.Contracts/ # DTOs, Enums, Interfaces
│   ├── JobScheduler.Data/      # EF Core, Repositories
│   └── JobScheduler.Infrastructure/ # Auth, Redis, Queue, Scheduling
└── docker/
    └── mysql/
        └── init/               # SQL initialization scripts
🔐 Security
JWT Authentication: Secure token-based authentication
Password Hashing: SHA256 password hashing
Role-Based Access: Admin, User, Viewer roles
Ownership Checks: Users can only access their own jobs
Input Validation: Comprehensive request validation
🚀 Performance
High Throughput: Designed for 30k+ jobs/second
Batch Operations: Optimized database batch inserts/updates
Partitioning: Hourly partitioning for job_run table
Queue Scaling: Multiple Redis queues for load distribution
Horizontal Scaling: Scale workers and schedulers independently
📝 API Endpoints
Authentication
POST /api/auth/signup - Sign up new user
POST /api/auth/login - Login and get JWT token
Jobs
POST /api/jobs - Create job
GET /api/jobs - List jobs (paginated)
GET /api/jobs/{jobId} - Get job details
PUT /api/jobs/{jobId} - Update job
DELETE /api/jobs/{jobId} - Delete job
PATCH /api/jobs/{jobId}/pause - Pause recurring job
PATCH /api/jobs/{jobId}/resume - Resume recurring job
GET /api/jobs/{jobId}/runs - Get job run history
GET /api/jobs/{jobId}/runs/{runId} - Get specific run details
Health (Monitor Service)
GET /api/health - Basic health check
GET /api/health/metrics - Detailed metrics
🛠️ Development
Local Development (Without Docker)
# Start MySQL and Redis
docker-compose up -d mysql redis

# Run migrations
cd services/JobScheduler.Migrations
dotnet run

# Run API
cd services/Job.Api
dotnet run

# Run Scheduler (separate terminal)
cd services/Job.Scheduler
dotnet run

# Run Worker (separate terminal)
cd services/Job.Worker
dotnet run
Build
# Build all projects
dotnet build

# Build specific service
dotnet build services/Job.Api/Job.Api.csproj
📚 Technology Stack
.NET 8.0: Core framework
MySQL 8.0: Primary database
Redis 7: Queues and heartbeats
Entity Framework Core: ORM
Serilog: Structured logging
JWT: Authentication
Docker: Containerization
NCrontab: Cron expression parsing
🤝 Contributing
Fork the repository
Create a feature branch
Make your changes
Submit a pull request
