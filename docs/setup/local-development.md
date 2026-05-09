# Local Development

Running the project on your machine.

---

## Frontend

```bash
cd tensor-labz-website

# Install dependencies
npm install

# Copy and fill environment variables
cp .env.example .env
# edit .env — see Environment Variables page

# Start dev server
npm run dev
# → http://localhost:5173
```

The dev server proxies nothing — all API calls go directly to Supabase and the deployed Lambda. You need valid `.env` values.

### Build for production locally

```bash
npm run build    # TypeScript check + Vite build → dist/
npm run preview  # Serve dist/ at http://localhost:4173
```

---

## Lambda (local testing)

The Lambda is designed for AWS but can be invoked locally for testing:

```bash
cd tensor-labz-image-lambda
npm install

# Type-check only (no local server)
npm run typecheck

# Build the bundle
npm run build         # outputs dist/index.js
npm run build:zip     # creates function.zip ready for AWS
```

!!! note "No local HTTP server"
There is no `npm run dev` that spins up a local HTTP API. To test endpoints, deploy to Lambda and use `curl` or the AWS Console Test tab.

---

## Admin panel

The admin panel is part of the same React app, routed at `/admin/*`.

1. Start the dev server: `npm run dev`
2. Navigate to `http://localhost:5173/admin`
3. You'll be redirected to `/login`
4. Log in with your Supabase Auth credentials

!!! tip "Creating an admin user"
Go to **Supabase Dashboard → Authentication → Users → Invite user**.
Enter your email. You'll receive a link to set a password.
That email + password is what you use on the `/login` page.

---

## Branching

```
dev        ← all development work goes here
   ↓  PR
staging    ← triggers Firebase Hosting deploy
   ↓  PR
main       ← triggers AWS Amplify production deploy
```

Never push directly to `main` or `staging`.
