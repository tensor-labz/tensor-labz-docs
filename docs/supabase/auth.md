# Authentication

The admin panel is protected by **Supabase Auth** using email + password and JWT sessions.

---

## App startup auth flow

```mermaid
flowchart TD
    A([App mounts]) --> B["providers.tsx\nsupabase.auth.onAuthStateChange()"]
    B --> C{Session\nexists?}
    C -->|"Yes (returning user)"| D["dispatch setUser(user)\ninitialized = true"]
    C -->|"No (new / signed out)"| E["dispatch setUser(null)\ninitialized = true"]
    D --> F["ProtectedRoute checks"]
    E --> F
    F --> G{initialized?}
    G -->|"false"| H["render null\n(no flash / redirect)"]
    G -->|"true"| I{user?}
    I -->|"Yes"| J["Render /admin/* children"]
    I -->|"No"| K["Navigate to /login\nwith ?from= return URL"]

    style D fill:#22c55e,color:#fff
    style E fill:#f97316,color:#fff
    style J fill:#3b82f6,color:#fff
    style K fill:#ef4444,color:#fff
```

---

## Login sequence

```mermaid
sequenceDiagram
    participant User
    participant LoginPage as Login.tsx
    participant AuthSlice as authSlice (Redux)
    participant Supabase
    participant ProtectedRoute

    User->>LoginPage: enter email + password → submit
    LoginPage->>AuthSlice: dispatch login({ email, password })
    AuthSlice->>Supabase: auth.signInWithPassword({ email, password })

    alt valid credentials
        Supabase-->>AuthSlice: session + user object
        AuthSlice->>AuthSlice: setUser(user), initialized=true
        AuthSlice-->>LoginPage: fulfilled
        LoginPage->>User: navigate to /admin (or original URL)
    else invalid credentials
        Supabase-->>AuthSlice: error "Invalid login credentials"
        AuthSlice-->>LoginPage: rejected
        LoginPage->>User: show error message
    end

    Note over ProtectedRoute: onAuthStateChange also fires<br/>sets initialized=true
```

---

## Sign-out flow

```mermaid
sequenceDiagram
    participant Admin
    participant AuthSlice
    participant Supabase
    participant ProtectedRoute

    Admin->>AuthSlice: dispatch signOut()
    AuthSlice->>Supabase: auth.signOut()
    Supabase-->>AuthSlice: ok
    Note over Supabase: onAuthStateChange fires<br/>session = null
    AuthSlice->>AuthSlice: setUser(null)
    ProtectedRoute->>Admin: redirect to /login
```

---

## JWT flow to Lambda

```mermaid
flowchart LR
    subgraph Frontend
        A["Admin panel\n(browser)"]
    end
    subgraph SupabaseAuth["Supabase Auth"]
        B["JWT\n(HS256)"]
    end
    subgraph Lambda["AWS Lambda"]
        C["verifyToken()\njsonwebtoken"]
        D["SUPABASE_JWT_SECRET\n(env var)"]
    end

    A -- "auth.getSession()" --> B
    B -- "access_token" --> A
    A -- "Authorization: Bearer JWT" --> C
    D -- "secret" --> C
    C -- "verified ✓" --> Lambda
```

---

## Creating admin users

1. Go to **Supabase Dashboard → Authentication → Users**
2. Click **Invite user** and enter the email address
3. User receives a link to set their password
4. That email + password is used on the `/login` page

!!! warning "No self-registration"
There is no signup page. Admin users can only be created from the Supabase dashboard.

---

## Session persistence

Supabase stores the session in `localStorage`. On page reload:

1. `onAuthStateChange` fires with the existing session
2. `initialized` becomes `true` with the user set
3. `ProtectedRoute` renders the admin directly — no redirect
