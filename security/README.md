# Compliance Calendar & Tracker

A full-stack compliance management application that helps organizations track regulatory deadlines, manage compliance tasks, and receive AI-powered compliance guidance.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend API | Spring Boot 3.2.5 (Java 17) |
| Database | PostgreSQL + Flyway Migrations |
| Authentication | Spring Security + JWT |
| Frontend | React + Vite + TailwindCSS |
| AI Service | Flask 3.1.3 (Python) |
| AI Provider | Groq API |
| Vector Store | ChromaDB |
| Embeddings | sentence-transformers |
| Caching | Redis |

## Team

| Role | Responsibility |
|------|---------------|
| Java Developer 1 | Spring Boot backend, REST APIs, database |
| Java Developer 2 | Authentication, JWT, security configuration |
| Java Developer 3 | File upload system, Redis caching |
| AI Developer 1 | RAG pipeline, ChromaDB integration |
| AI Developer 2 | Flask routes, Groq LLM integration |
| AI Developer 3 | Prompt engineering, AI service architecture |
| **Security Reviewer** | **VAPT, vulnerability analysis, security documentation** |

---

## Prerequisites

- Java 17+
- Maven 3.8+
- Node.js 18+ and npm 8+
- Python 3.10 or 3.11
- PostgreSQL 14+
- Redis 6+ (for caching)

---

## Setup Guide

### 1. Database Setup

```bash
# Start PostgreSQL
sudo service postgresql start  # Linux
brew services start postgresql # Mac

# Create database
psql -U postgres
CREATE USER compliance_user WITH PASSWORD 'your_password';
CREATE DATABASE compliance_db;
GRANT ALL PRIVILEGES ON DATABASE compliance_db TO compliance_user;
\q
```

### 2. Spring Boot Backend

```bash
# Configure environment
cp src/main/resources/application.properties.example \
   src/main/resources/application.properties
# Edit application.properties with your DB credentials

# Build and run
mvn clean install -DskipTests
mvn spring-boot:run

# API runs at: http://localhost:8080
```

### 3. Flask AI Service

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies (minimal set for development)
pip install flask flask-cors python-dotenv groq chromadb sentence-transformers

# Configure environment
cp .env.example .env
# Edit .env with your Groq API key and INTERNAL_API_KEY

# Run AI service
python app.py
# AI service runs at: http://localhost:5000
```

### 4. React Frontend

```bash
# Navigate to frontend directory
cd frontend

# Install dependencies
npm install

# Configure environment
cp .env.example .env.local
# Edit VITE_API_URL to point to your Spring Boot instance

# Start development server
npm run dev
# Frontend runs at: http://localhost:5173
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  React Frontend (:5173)                  │
│         React + Vite + TailwindCSS + Axios              │
└─────────────────────┬───────────────────────────────────┘
                      │ JWT Bearer Token
                      ▼
┌─────────────────────────────────────────────────────────┐
│             Spring Boot Backend (:8080)                  │
│  JWT Filter → Controllers → Services → JPA → PostgreSQL  │
│                        ↓ Redis Cache                     │
│              File Upload → /uploads directory            │
└─────────────────────┬───────────────────────────────────┘
                      │ X-Internal-Key (internal only)
                      ▼
┌─────────────────────────────────────────────────────────┐
│              Flask AI Service (:5000)                    │
│   /describe → Groq LLM + ChromaDB RAG                   │
│   /recommend → Compliance recommendations               │
│   /report → AI-generated compliance reports             │
└─────────────────────────────────────────────────────────┘
```

---

## API Endpoints

### Authentication (Spring Boot)
```
POST /api/auth/register   - Register new user
POST /api/auth/login      - Login, receive JWT
```

### Tasks (Spring Boot — JWT required)
```
GET    /api/tasks          - List user's tasks
POST   /api/tasks          - Create task
GET    /api/tasks/{id}     - Get task by ID
PUT    /api/tasks/{id}     - Update task
DELETE /api/tasks/{id}     - Delete task
```

### AI Service (Flask — Internal key required)
```
GET  /health              - Health check
POST /describe            - AI task description
POST /recommend           - Compliance recommendations
POST /report              - Generate compliance report
```

---

## Environment Variables

See `.env.example` for the full list. Key variables:

| Variable | Where | Purpose |
|----------|-------|---------|
| `VITE_API_URL` | Frontend `.env` | Spring Boot base URL |
| `INTERNAL_API_KEY` | Flask `.env` | Flask service auth key |
| `GROQ_API_KEY` | Flask `.env` | Groq AI provider |
| `JWT_SECRET` | Spring Boot `application.properties` | JWT signing key |
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | Spring Boot | PostgreSQL connection |

**Never commit `.env` to git.** Use `.env.example` as a template.

---

## Security

This project has undergone a full VAPT (Vulnerability Assessment and Penetration Testing) review. See the [security documentation](security/) for details.

Key security findings and their status:

| Finding | Severity | Status |
|---------|----------|--------|
| Flask debug=True | 🔴 Critical | Fix provided |
| Flask no authentication | 🔴 Critical | Fix provided |
| Prompt injection | 🟠 High | Sanitizer implemented |
| .env in git | 🟡 Medium | Resolved |

For vulnerability disclosure, see [SECURITY.md](SECURITY.md).

---

## Running Tests

```bash
# Spring Boot tests
mvn test

# Python safety check
pip install safety
safety check -r requirements.txt

# Security smoke test
bash scripts/security_test.sh
```

---

## Project Structure

```
compliance-calendar-tracker/
├── src/                    # Spring Boot Java source
│   └── main/
│       ├── java/           # Controllers, Services, Repositories
│       └── resources/      # application.properties, Flyway migrations
├── frontend/               # React + Vite frontend
├── ai_service/             # Flask AI service
│   ├── app.py              # Flask entry point
│   ├── routes/             # Blueprint routes
│   ├── services/           # Business logic + AI integration
│   ├── rag/                # RAG pipeline modules
│   └── prompts/            # LLM prompt templates
├── data/                   # RAG training documents
├── security/               # Security review documentation
├── scripts/                # Utility and test scripts
├── pom.xml                 # Spring Boot dependencies
├── requirements.txt        # Python dependencies
├── .env.example            # Environment template
└── SECURITY.md             # Security policy
```

---

## License

Academic capstone project — not for commercial use.
