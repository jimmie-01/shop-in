# AGENTS.md — Basic Electronics Shop MVP

## 1. Project goal

Build a small, dependable online shop for basic electronic appliances. The first release should let visitors browse products, let the shop owner manage the catalogue, let customers place orders, send an order confirmation email, and let signed-in customers return later to see their order history.

The requester is new to React, Supabase, and Google Cloud Console. Explain setup and code changes in plain language, one step at a time. Prefer a working, understandable MVP over abstractions or extra features.

## 2. Agreed stack and starting assumptions

- Frontend: React + TypeScript, preferably Vite.
- Backend: Express + TypeScript.
- Data, authentication, and product image storage: Supabase (Postgres, Supabase Auth, Supabase Storage).
- Email: Mailgun, called only from the Express server.
- Google sign-in: Google OAuth configured through Supabase Auth.
- Use npm unless the repository already uses another package manager.
- If this file is used before a repository exists, propose a simple `client/` and `server/` layout and explain each part.
- Use Supabase rather than Neon for this MVP: Supabase combines the Postgres database, Google-capable Auth, and image Storage. Do not add Neon as a second database. If the owner later chooses Neon, explain that Auth and image storage still need separate services.

## 3. MVP requirements

### Customer shop

- List active products with name, description, price, image, and stock/availability state.
- Show product details and allow adding/removing quantities in a cart.
- Keep the cart through refreshes where practical (browser local storage is sufficient for the MVP); the cart is not an order and is not trusted for final prices.
- Provide a checkout page that collects the minimum customer and delivery information needed by the owner.
- Require sign-in before placing an order, unless the product owner explicitly changes that requirement.
- At checkout, the server must load product names and prices from the database, validate quantities and availability, calculate totals, and create the order. Never trust prices or totals submitted by the browser.
- Display a clear success/failure result. Send a confirmation email after a successful order is saved.

### Shop owner

- Only the configured shop owner may create products, change product details/prices, upload/change product images, and remove or deactivate products.
- Public visitors may read active products only.
- Enforce owner permissions in the Express API and database/storage policies. Hiding admin buttons in React is not security.
- Prefer deactivating products (`is_active = false`) over deleting rows referenced by old orders. Remove an image only when it is no longer referenced.

### Accounts and orders

- Sign in with Google through Supabase Auth.
- A signed-in customer can view only their own orders and order items.
- Sign out must clear the Supabase session.
- Supabase Auth's persisted session should restore after refresh or closing and reopening the page. Load the session on app startup and handle auth-state changes; fetch orders from the persistent database after sign-in. Do not rely on React state or local storage as the order history.
- Keep historical order item name, quantity, and unit price as a snapshot so later product edits do not rewrite past purchases.

## 4. Suggested repository shape

```text
client/
  src/
    components/
    pages/
    lib/supabaseClient.ts
    App.tsx
server/
  src/
    middleware/      # authentication and owner checks
    routes/          # products, orders, health
    services/        # order creation, Mailgun
    lib/
    app.ts
supabase/
  migrations/        # schema and database policies
.env.example
AGENTS.md
README.md
```

Keep components and files small and name them by purpose. Add a dependency only when needed. Do not put secrets in the React app or commit real `.env` files.

## 5. Data model guidance

Use migrations checked into `supabase/migrations/` (or the project's established migration tool) so the schema can be recreated. A practical starting schema is:

- `products`: `id`, `name`, `description`, `price` (numeric; never floating-point money), `image_path` or `image_url`, `stock_quantity` (if tracking stock), `is_active`, timestamps.
- `orders`: `id`, `user_id` (Supabase Auth user UUID), customer/delivery fields, `status`, `total_amount`, timestamps.
- `order_items`: `id`, `order_id`, `product_id` (nullable if product is later removed), `product_name`, `unit_price`, `quantity`, `line_total`.

Use foreign keys, nonnegative price/quantity checks, and indexes for `orders.user_id` and `order_items.order_id`. Save order and item rows atomically, preferably in a Postgres function/transaction so a partial order cannot be created. Store money in a fixed-precision numeric column or integer minor units; format it only for display.

Enable Row Level Security (RLS). Policies should allow customers to select their own orders and their own order items, while preventing them from changing prices, totals, owner flags, or other users' orders. Public product reads should expose active products only. Admin writes and product image upload/delete must be limited to the owner. The Express service role key bypasses RLS, so every server route using it must authenticate the caller and authorize owner/customer access before querying or mutating data.

## 6. Beginner setup and integration path

Work in this order. At each step, explain how to run it and what success looks like.

### Step 1 — Create and run the app

1. Check whether a repository and package setup already exist; do not replace existing choices without a reason.
2. If starting fresh, create a Vite React TypeScript client and an Express TypeScript server in `client/` and `server/`.
3. Add a health endpoint such as `GET /api/health` and a simple React page that calls it. Configure local development origins/CORS explicitly.
4. Provide commands for installing dependencies and running both processes locally. Use separate terminals or a simple documented root script.

### Step 2 — Create Supabase project, schema, and image bucket

1. Create a Supabase project and save its Project URL and publishable/anon key for the frontend; obtain the service-role/secret key for the server only.
2. Create database migrations for the tables, constraints, indexes, RLS, and policies described above. Do not ask a beginner to paste an unreviewed policy that grants everyone write access.
3. Create a product-images Storage bucket. Decide public vs private deliberately: product catalogue images can use a public bucket for simple display, while write/upload permissions remain owner-only. Apply and document Storage policies.
4. Put configuration in environment variables and verify a harmless product read before wiring the whole UI.

### Step 3 — Connect React to Supabase Auth

1. Install `@supabase/supabase-js` in the client and create one reusable client module.
2. Expose only the Supabase project URL and publishable/anon key through Vite-prefixed client variables. Never expose a service-role key, Google client secret, or Mailgun key to Vite.
3. Add a sign-in button using Supabase Google OAuth, restore the existing session when the app opens, subscribe to auth-state changes, and add sign-out.
4. Verify sign-in, refresh, close/reopen, sign-out, and sign-in again before adding orders.

### Step 4 — Configure Google OAuth in Google Cloud Console

1. Create/select a Google Cloud project and configure the Google Auth Platform/OAuth consent screen with the app name and audience. While testing, add the intended test accounts when Google requires them.
2. Create an OAuth Client ID of type **Web application**.
3. Add the local frontend origin (for example `http://localhost:5173`) to **Authorized JavaScript origins**. Add the production site origin when deployed.
4. In Supabase Dashboard, open Authentication provider settings for Google, enable Google, and copy the Supabase callback URL shown there.
5. Paste that exact Supabase callback URL into Google's **Authorized redirect URIs**. It is the Supabase callback, not the React page URL.
6. Copy Google's Client ID and Client Secret into the Supabase Google provider settings. Save them in the dashboard, never in frontend source code.
7. Set the Supabase Site URL and allowed redirect URLs for local and production app URLs. Use HTTPS in production. Test the full redirect and check browser/server logs for redirect mismatch errors.

### Step 5 — Build the catalogue and owner tools

1. Read products from the API/database and show loading, empty, and error states.
2. Add an owner-only page/form for product creation and editing, including image selection and price validation.
3. Upload image files to the Storage bucket, save the resulting path/reference in `products`, and render the image in the catalogue.
4. Add server middleware that verifies the Supabase bearer token and confirms the caller is the configured owner before allowing product writes or Storage changes. Configure matching RLS/Storage rules as a second protection layer.
5. Configure the owner identity through a server-only allowlist (prefer the owner's stable Supabase Auth user UUID; email allowlisting may be used during initial setup but must be normalized and verified). Do not allow a user to make themselves owner by editing a profile in the browser.

### Step 6 — Cart and checkout

1. Build a cart with product ID and quantity; look up current product information for display.
2. At submit, send product IDs, quantities, and delivery/customer fields to the Express API. Treat client price fields as display-only and ignore them on the server.
3. Verify the session token, validate the request, re-read active products and prices, calculate totals on the server, then save the order and item snapshots atomically.
4. Return the created order ID and show confirmation to the customer. Avoid duplicate orders on retries (use an idempotency key or equivalent if practical).

### Step 7 — Mailgun confirmation email

1. Create a Mailgun account and sending domain, then follow Mailgun's domain/DNS verification steps for the chosen sending domain.
2. Create and protect a Mailgun API key. Store it only in the server environment. Configure the verified sender address/domain and Mailgun region/API endpoint as needed.
3. Send a concise confirmation containing order number, item summary, total, and customer support/contact details. Escape or safely template user-controlled values.
4. Save the order before attempting email. If email delivery fails, keep the order, log a safe error, and expose a retry/manual follow-up path; do not tell the customer the order failed if it was saved.
5. Never log secrets, access tokens, or unnecessary personal data.

### Step 8 — Persistent order history

1. Add an authenticated `GET /api/orders` route that returns only the current customer's orders and item snapshots.
2. Query the database on page load/sign-in rather than storing order data only in React state or local storage.
3. Add an orders page accessible when signed in. After sign-out, protected order data should no longer be shown. After sign-in on a new browser session, fetch and display the saved orders.
4. Verify the complete acceptance flow: place order, see confirmation, sign out, close/reopen page, sign in again, see the same order; a second account must not see it.

## 7. API boundaries

Suggested initial routes:

- `GET /api/health`
- `GET /api/products` and `GET /api/products/:id` (public read of active products)
- `POST /api/products`, `PATCH /api/products/:id`, `DELETE /api/products/:id` (owner only; delete should normally deactivate)
- `POST /api/orders` (signed-in customer; prices are resolved server-side)
- `GET /api/orders` and `GET /api/orders/:id` (signed-in customer; own records only)

Use consistent JSON responses, appropriate HTTP status codes, request validation, and centralized error handling. Do not return stack traces or provider secrets to the browser.

## 8. Security and quality rules

- Validate all input on the server, including IDs, prices, quantities, email/address lengths, and image MIME type/size.
- Authenticate every protected Express request by validating the Supabase access token; do not trust a user ID supplied in the request body.
- Use the server-only service-role key only where needed, and authorize each operation before using it.
- Keep secrets in ignored environment files locally and in the deployment platform's secret store in production. Commit `.env.example` with placeholder names only.
- Use HTTPS in production, restrictive CORS, and safe error messages. Avoid storing payment card details; payments are outside this MVP unless explicitly added later.
- Do not mark an order paid unless a real payment provider confirms it. If this version collects orders without online payment, label its status honestly (for example `pending`).
- Explain new React concepts (components, props, state, effects, forms) briefly as they appear. Avoid introducing state libraries or elaborate architecture for this MVP.

## 9. Completion checklist

- [ ] Visitor can browse active appliance products and view their images and prices.
- [ ] Only the configured owner can add/update/deactivate products and manage product images.
- [ ] Cart quantities work and survive refresh if practical.
- [ ] Checkout computes current prices and totals on the server.
- [ ] A successful checkout creates one persistent order and item snapshots.
- [ ] Confirmation email is attempted through Mailgun after saving the order, with failure handled safely.
- [ ] Google sign-in/out works through Supabase Auth.
- [ ] A returning signed-in customer sees their orders after closing and reopening the app.
- [ ] Customers cannot read one another's orders.
- [ ] No service-role, Google secret, or Mailgun key is present in client code or committed files.
- [ ] README explains prerequisites, environment variables, migrations, local startup, and the Google/Supabase/Mailgun setup in beginner-friendly steps.

## 10. Official setup references

Use these official guides when checking provider-specific UI labels or current setup details:

- Supabase React Auth quickstart: https://supabase.com/docs/guides/auth/quickstarts/react
- Supabase Google sign-in setup: https://supabase.com/docs/guides/auth/social-login/auth-google
- Supabase Storage quickstart: https://supabase.com/docs/guides/storage/quickstart
- Supabase JWT guidance: https://supabase.com/docs/guides/auth/jwts
- Mailgun documentation: https://documentation.mailgun.com/


### Users and profiles

- Supabase Auth owns user accounts and credentials in its managed `auth.users` table. Do not create a second `users` table to store passwords or duplicate Google sign-in data.
- Use the Supabase Auth user UUID as the customer's ID. Set `orders.user_id` to reference `auth.users(id)` with an appropriate delete policy.
- A public `profiles` table is optional. Add one only for app-specific data the shop needs, such as a display name or phone number. Key it by `user_id` referencing `auth.users(id)`, and apply RLS so users can read/update only their own profile.
- Do not let users assign themselves a role or owner flag in `profiles`. Keep shop-owner authorization in a server-controlled allowlist or another trusted server-side configuration.
- Create a profile row on first sign-in only if the app needs profile fields; order history should work directly from the Auth user UUID.