
# Shop App - Agent Instructions

## Project Goal

Build a small full-stack shop application.

The application has two types of users:

- Shopper
- Shop owner

Shop owners can manage products.

Shoppers can browse products, add products to a cart,
checkout, and view their order history.

Keep the application simple and production-minded.
Do not introduce unnecessary architecture or dependencies.

---

# Technology Stack

## Frontend

- React
- Vite
- React Router
- TypeScript

## Backend

- Node.js
- Express
- TypeScript
- Zod

## Database

- Supabase PostgreSQL

## File Storage

- Supabase Storage

## Authentication

- Google Identity Services
- Google Cloud OAuth
- Backend-managed authentication/session

## Email

- Mailgun

## Testing

- Vitest
- Supertest

## Development

- Docker
- Git
- GitHub

---

# Architecture

Use a simple layered backend architecture:

Routes
  ↓
Middleware
  ↓
Controllers
  ↓
Services
  ↓
Repositories
  ↓
Database

Controllers handle HTTP concerns.

Services contain business logic.

Repositories handle database access.

Do not access the database directly from controllers.

Do not put business logic inside routes.

---

# Frontend Architecture

Keep the React application simple.

Use:

- components for reusable UI
- pages for route-level screens
- services/api modules for backend requests
- hooks for reusable React logic

Do not introduce Redux or another global state-management
library unless the existing application genuinely requires it.

For the initial version, React state/context is sufficient.

---

# Authentication

Users authenticate using Google.

The backend must verify Google ID tokens.

Never trust:

- userId from the request body
- email from the request body
- role from the request body

The authenticated user's identity must come from
verified authentication data.

The backend determines the application user from the
verified Google identity.

Use the Google `sub` claim as the stable Google identity.

Authentication credentials must not be stored in localStorage.

Use secure HTTP-only cookies for the application's
authentication/session mechanism.

---

# Authorization

Users have one of two roles:

- shopper
- shop_owner

Shopper:

- browse products
- manage cart
- checkout
- view own orders

Shop owner:

- create products
- update products
- delete products
- view/manage products

A shopper must never be able to perform shop-owner operations.

Authorization must be enforced by the backend.

Do not rely on hiding frontend buttons for authorization.

---

# Products

A product contains at minimum:

- id
- name
- description
- price
- image URL
- stock quantity
- createdAt
- updatedAt

Product images are stored in Supabase Storage.

Do not store binary image data directly in PostgreSQL.

---

# Cart

The initial cart may be managed on the client.

Do not create persistent cart database tables unless
the requirement actually needs carts to survive across
devices or sessions.

The backend must recalculate product prices during checkout.

Never trust a total supplied by the frontend.

---

# Orders

Orders must be persistent in PostgreSQL.

Use:

- users
- products
- orders
- order_items

An order belongs to exactly one user.

Users can only retrieve their own orders.

Shop owners must not automatically receive access to
all customer orders unless a specific requirement adds it.

Store the product price used at checkout in order_items.

Never calculate historical order totals using the
current product price.

---

# Checkout

The frontend sends product IDs and quantities.

The backend:

1. authenticates the user
2. validates the request
3. retrieves products from the database
4. verifies availability
5. calculates the total
6. creates the order
7. creates order items
8. commits the database transaction
9. sends the confirmation email

The frontend must never determine the authoritative
order total.

---

# Email

Use Mailgun for order confirmation emails.

Do not expose Mailgun credentials to React.

Mailgun credentials belong only in the backend environment.

Email sending should not corrupt or roll back a successful
order if Mailgun temporarily fails.

If email delivery fails, the order should remain stored.

Log the email failure appropriately.

Do not expose secrets in logs.

---

# Database

Use Supabase PostgreSQL.

Database access must happen from the backend.

Never expose the database connection string to the frontend.

Keep database credentials in environment variables.

Use migrations for schema changes.

Do not modify production data manually during normal development.

---

# Environment Variables

Never commit:

- .env
- database passwords
- Google client secrets
- Mailgun API keys
- Supabase service-role keys
- JWT/session secrets

Provide `.env.example` containing variable names
but not real credentials.

Example:

DATABASE_URL=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
MAILGUN_API_KEY=
MAILGUN_DOMAIN=
MAIL_FROM=
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
SESSION_SECRET=

---

# Security

Never trust client-provided:

- user ID
- role
- price
- order total
- ownership information

Validate external input with Zod.

Use parameterized database queries.

Do not expose stack traces to clients in production.

Do not log passwords, tokens, API keys, cookies,
or database credentials.

Use secure HTTP-only cookies for authentication.

Use appropriate CORS configuration.

---

# Error Handling

Use a centralized Express error-handling middleware.

Controllers should pass errors to the error handler.

Return appropriate HTTP status codes.

Do not expose internal implementation details
in production responses.

---

# Testing

Every meaningful backend feature should include tests.

Unit tests:

- services
- business logic
- validation
- authorization logic where appropriate

Integration tests:

- HTTP endpoints
- authentication flows
- database interactions
- order creation
- authorization boundaries

Tests must not use the production database.

---

# Development Rules

Before making changes:

1. Inspect the existing code.
2. Understand the current architecture.
3. Identify the smallest change needed.
4. Preserve existing conventions.

After making changes:

1. Run relevant tests.
2. Run TypeScript checks.
3. Run linting if configured.
4. Review the git diff.

Do not modify unrelated files.

Do not rewrite working architecture without a clear reason.

---

# Dependency Rules

Do not add a dependency when the requirement can
reasonably be implemented with the existing stack.

Before adding a dependency:

1. Explain why it is needed.
2. Check whether an existing dependency already solves the problem.
3. Prefer well-maintained, focused packages.

Do not introduce:

- Redux
- GraphQL
- microservices
- Redis
- message queues
- Elasticsearch
- Kubernetes
- unnecessary ORM layers
- unnecessary abstraction frameworks

unless a concrete requirement requires them.

---

# Scope Control

The initial application should contain only:

1. Product browsing
2. Product management for the shop owner
3. Google authentication
4. Shopping cart
5. Checkout
6. Persistent orders
7. Order history
8. Order confirmation emails

Do not add:

- coupons
- reviews
- ratings
- wishlists
- product recommendations
- multi-vendor support
- inventory forecasting
- payment gateways
- analytics dashboards
- notifications
- subscriptions

unless explicitly requested.

---

# Agent Behaviour

Do not assume requirements.

If a requirement is ambiguous and affects architecture,
ask before implementing.

For small changes, implement directly.

For significant changes:

1. Explain the proposed approach.
2. Identify affected files.
3. Implement the smallest maintainable solution.
4. Run tests.
5. Report what changed.

Do not claim that a feature works unless it has been
tested appropriately.

Prefer simple, readable code over clever abstractions.