# RinFlow

RinFlow is a full-stack loan application and review prototype. Applicants can submit financial information and track their applications, while loan officers can review applications, inspect AI-assisted risk summaries, and record approval or rejection decisions.

The application uses a React frontend, an Express/MongoDB backend, and Google Gemini for structured decision-support analysis. Gemini does **not** make the authoritative lending decision; the final action remains with a human loan officer.

[View the live frontend](https://rin-flow.vercel.app) · [Source repository](https://github.com/Kanishk-blip/rin-flow)

## Features

### Applicants

- Create an applicant account and sign in.
- Submit a loan application with income, debt, CIBIL score, loan, and asset information.
- View previous applications and their current status.
- Receive immediate confirmation while risk analysis continues in the backend.

### Loan officers

- View all submitted loan applications.
- Search by loan or applicant ID.
- Filter by status and loan amount.
- Sort by date, amount, or CIBIL score.
- Review application details and stored AI risk analysis.
- Approve, reject, or reset an application to pending.

### Risk analysis

- Calculates deterministic financial metrics before contacting Gemini:
  - Debt-to-income ratio
  - Loan-to-income ratio
  - Total asset value
  - Asset coverage ratio
  - Estimated EMI
- Requests a structured Gemini assessment containing risk level, recommendation, reasoning, strengths, concerns, key risk factors, and suggested conditions.
- Stores the generated analysis with the loan so it can be reviewed without calling Gemini every time.
- Retries analysis when an officer opens a loan whose analysis is missing.

## Architecture

```text
Applicant / Loan Officer
          |
          v
React + Vite single-page application
          |
          | REST API with credentialed cookies
          v
Express modular monolith
  |-- Routes
  |-- JWT authentication middleware
  |-- Controllers
  |-- RinflowEngine
  |     |-- Financial metric calculation
  |     `-- Gemini risk analysis
  |
  `-- Mongoose models
          |
          v
       MongoDB
```

The backend is a single deployable service with separated routes, controllers, models, middleware, utilities, and external-service adapters.

## Main application flow

```text
Applicant submits the loan form
  -> frontend sends POST /api/v1/applicant/apply
  -> JWT middleware authenticates the applicant
  -> backend creates a Pending loan in MongoDB
  -> API returns 201 immediately
  -> backend calculates financial metrics
  -> Gemini generates a structured risk assessment
  -> assessment is saved in the loan document
  -> loan officer opens the application
  -> officer reviews the data and AI summary
  -> officer manually approves or rejects the loan
```

The Gemini operation currently runs as an in-process Promise after the HTTP response. It is not backed by a durable job queue.

## Technology stack

### Frontend

- React 19
- Vite
- React Router
- React Hook Form
- Tailwind CSS
- Material UI
- Axios and the Fetch API
- Spline for the animated background

### Backend

- Node.js with ECMAScript modules
- Express 5
- MongoDB and Mongoose
- JSON Web Tokens and HTTP-only cookies
- Google Generative AI SDK

## Project structure

```text
rin-flow/
|-- frontend/
|   |-- public/
|   |-- src/
|   |   |-- App.jsx                 # Routes and global Spline background
|   |   |-- LandPage.jsx            # Public landing page
|   |   |-- signup.jsx              # Applicant registration
|   |   |-- Login.jsx               # Login form
|   |   |-- Dashboard.jsx           # Applicant/officer dashboards
|   |   |-- LoanApplicationForm.jsx # Loan submission form
|   |   |-- Loan.jsx                # Officer review and decision page
|   |   |-- Navbar.jsx              # Shared navigation
|   |   |-- main.jsx                # React entry point
|   |   `-- index.css               # Tailwind import
|   |-- package.json
|   `-- vite.config.js
|
|-- backend/
|   |-- scripts/
|   |   `-- seed.js                  # Sample applicants and loans
|   |-- src/
|   |   |-- controllers/             # Authentication and loan workflows
|   |   |-- core/                    # RinflowEngine integration facade
|   |   |-- db/                      # MongoDB connection
|   |   |-- helpers/                 # Configured engine instance
|   |   |-- middlewares/             # JWT authentication middleware
|   |   |-- models/                  # Applicant, officer, and loan schemas
|   |   |-- routes/                  # REST route definitions
|   |   |-- services/
|   |   |   |-- gemini-api/          # Gemini risk assessment
|   |   |   `-- summarize/           # Financial calculations
|   |   |-- utils/                   # API response/error helpers
|   |   |-- app.js                   # Express configuration
|   |   `-- index.js                 # Backend entry point
|   `-- package.json
|
`-- README.md
```

## API overview

All paths below are prefixed with `/api/v1`.

### Authentication

| Method | Endpoint | Authentication | Description |
| --- | --- | --- | --- |
| `POST` | `/register` | Public | Register an applicant or loan officer |
| `POST` | `/login` | Public | Authenticate and set access/refresh cookies |
| `POST` | `/logout` | Required | Remove the stored refresh token and clear cookies |
| `GET` | `/getUser` | Required | Return the authenticated user and role |

### Applicant

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/applicant/loans` | Get the authenticated applicant's loans |
| `POST` | `/applicant/apply` | Create a loan application and start risk analysis |

### Loan officer

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/loanOfficer/getAllLoans` | Get every loan application |
| `GET` | `/loanOfficer/:loanId` | Get one loan and its stored AI analysis |
| `POST` | `/loanOfficer/accept/:loanId` | Mark a loan as approved |
| `POST` | `/loanOfficer/reject/:loanId` | Mark a loan as rejected |
| `POST` | `/loanOfficer/reset/:loanId` | Reset a loan to pending |

## Local development

### Prerequisites

- A recent Node.js LTS release and npm
- A MongoDB instance or MongoDB Atlas connection string
- A Google Gemini API key for risk analysis

### 1. Clone the repository

```bash
git clone https://github.com/Kanishk-blip/rin-flow.git
cd rin-flow
```

### 2. Configure the backend

Install dependencies:

```bash
cd backend
npm install
```

Create `backend/.env`:

```dotenv
PORT=8000
MONGODB_URI=mongodb://127.0.0.1:27017

ACCESS_TOKEN_SECRET=replace-with-a-long-random-secret
ACCESS_TOKEN_EXPIRY=15m
REFRESH_TOKEN_SECRET=replace-with-another-long-random-secret
REFRESH_TOKEN_EXPIRY=7d

GEMINI_API=your-gemini-api-key
```

Start the backend:

```bash
npm run dev
```

### 3. Configure the frontend

In another terminal:

```bash
cd frontend
npm install
```

Create `frontend/.env`:

```dotenv
VITE_API_URL=http://localhost:8000
```

Start the frontend:

```bash
npm run dev
```

Open the URL printed by Vite, normally `http://localhost:5173`.

### Local CORS note

The current backend CORS configuration allows only `https://rin-flow.vercel.app`; the localhost configuration in `backend/src/app.js` is commented out. Enable the localhost origin or make the allowed origin environment-based before testing the frontend locally.

## Seed sample data

The seed script creates five sample applicants and loan applications, then requests and stores Gemini analysis for each one.

From the `backend` directory:

```bash
node scripts/seed.js
```

The script loads its environment file using `../.env`, so verify the expected environment-file location before running it. It waits 15 seconds between Gemini calls to reduce quota errors.

## Integration status

| Integration | Status | Current behavior |
| --- | --- | --- |
| MongoDB | Active | Stores users, loans, statuses, and AI analysis |
| Gemini | Active | Produces structured risk summaries |
| Spline | Active | Renders the global animated frontend background |