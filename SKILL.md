---
name: frappe-react-fullstack
description: >
  Build full-stack internal business applications using Frappe Framework (Python backend) with React 18 + Vite + TypeScript + Tailwind CSS v4 frontend. Use this skill when the user asks to "build a Frappe app with React", "create a React SPA for Frappe", "set up Frappe with React frontend", "add a new module to a Frappe React project", "create admin panel with Frappe backend", "build customer portal with Frappe", or is working on any project that combines Frappe Framework with a React-based SPA frontend.
---

# Frappe + React Full-Stack Application Skill

## When to Use

This skill applies when building internal business tools with:
- **Backend**: Frappe Framework (Python 3.11+, MariaDB, Redis)
- **Frontend**: React 18 + Vite + TypeScript + Tailwind CSS v4 + Lucide icons
- **Architecture**: Two SPAs (admin panel + customer portal) backed by Frappe doctypes and whitelisted APIs

---

## 1. Project Structure

Always scaffold projects with this layout:

```
project-root/
├── CLAUDE.md                          # Project conventions & instructions
├── docs/                              # Requirements, API design, doctype specs
├── docker-compose.yml                 # MariaDB + Redis + Frappe + React
├── docker/                            # Dockerfiles
├── frappe-bench/
│   └── apps/<app_name>/
│       └── <app_name>/
│           ├── doctype/               # Frappe doctype definitions
│           ├── api/                   # Whitelisted Python methods
│           ├── scheduled_tasks/       # Cron/scheduler jobs
│           └── utils/                 # Helpers (PDF, email, etc.)
└── frontend/
    └── src/
        ├── main.tsx                   # Entry: BrowserRouter wrapping App
        ├── App.tsx                    # Routes + ThemeProvider + AuthProvider
        ├── index.css                  # Tailwind v4 + CSS variables + dark mode
        ├── api/client.ts              # Frappe API client with CSRF handling
        ├── components/                # Shared: AuthContext, ThemeContext, layouts, guards
        └── pages/
            ├── portal/                # Customer-facing SPA
            └── admin/                 # Admin panel SPA
```

---

## 2. Backend — Frappe Conventions

### Doctype Rules
- All field names: `snake_case`
- Status fields: Frappe `Select` type with fixed option lists (never free text)
- Dates: ISO 8601 (`YYYY-MM-DD`)
- Money: `Currency` type (specify currency code, e.g., SAR)
- Every doctype gets a `naming_series` for auto-generated IDs (e.g., `INV-YYYY-#####`)

### API Pattern — Whitelisted Methods

```python
import frappe
from frappe import _

@frappe.whitelist()
def do_something(param1, param2):
    """Always check permissions first."""
    if "Credit Admin" not in frappe.get_roles() and frappe.session.user != "Administrator":
        frappe.throw(_("Insufficient permissions."), frappe.PermissionError)

    doc = frappe.get_doc("My Doctype", param1)
    doc.field = param2
    doc.save()
    frappe.db.commit()
    return {"success": True}
```

### Hard Rules
- NEVER use `frappe.db.sql()` raw queries when ORM can do the job
- NEVER send emails synchronously — always `frappe.sendmail()` (enqueued)
- NEVER store files outside Frappe's file attachment system
- Always validate roles in every whitelisted method before accessing data
- Use `frappe.get_list()` / `frappe.get_doc()` / `frappe.get_count()` — not raw SQL

### Scheduler Tasks

```python
# scheduled_tasks/daily_notifications.py
def run_all():
    """Idempotent: use flag fields to avoid duplicate sends."""
    docs = frappe.get_list("My Doctype", filters={"alert_sent": 0, "expiry_date": ["<=", threshold]})
    for d in docs:
        frappe.sendmail(recipients=[...], subject="...", message="...")
        frappe.db.set_value("My Doctype", d.name, "alert_sent", 1)
    frappe.db.commit()
```

---

## 3. Frontend — React Patterns

### Vite Configuration

```typescript
// vite.config.ts
import tailwindcss from '@tailwindcss/vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  server: {
    port: 5173,
    proxy: {
      '/api': { target: 'http://localhost:8000', changeOrigin: true },
      '/private': { target: 'http://localhost:8000' },
      '/files': { target: 'http://localhost:8000' },
    },
  },
})
```

### Tailwind CSS v4 Setup

```css
/* index.css */
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));

:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --card: 0 0% 100%;
  --primary: 221.2 83.2% 53.3%;
  --muted: 210 40% 96%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --border: 214.3 31.8% 91.4%;
  --input: 214.3 31.8% 91.4%;
  --ring: 221.2 83.2% 53.3%;
}

.dark {
  --background: 224 71% 4%;
  --foreground: 213 31% 91%;
  --card: 224 71% 4%;
  --primary: 217.2 91.2% 59.8%;
  --muted: 223 47% 11%;
  --muted-foreground: 215.4 16.3% 56.9%;
  --border: 216 34% 17%;
  --input: 216 34% 17%;
}

/* Restore native date picker (Tailwind v4 sets appearance: none) */
input[type="date"], input[type="datetime-local"], input[type="time"] {
  appearance: auto;
  -webkit-appearance: auto;
}

/* Global dark mode overrides */
.dark .bg-white { background-color: hsl(var(--card)) !important; }
.dark input:not([type="radio"]):not([type="checkbox"]),
.dark select, .dark textarea {
  background-color: hsl(var(--background)) !important;
  color: hsl(var(--foreground)) !important;
  border-color: hsl(var(--border)) !important;
}
```

**Key Tailwind v4 Gotchas:**
- Use `@custom-variant dark` instead of Tailwind v3's `darkMode: 'class'` config
- Use `@tailwindcss/vite` plugin — no PostCSS config needed
- Global `.dark .bg-white` override avoids adding `dark:` to every component
- Date inputs need `appearance: auto` to restore native calendar picker
- Exclude `input[type="radio"]` and `input[type="checkbox"]` from dark overrides

### Frappe API Client

The API client must handle:
1. **CSRF Token** — extract from Frappe home page, auto-retry on 403/417
2. **Session cookies** — `credentials: "include"` on all fetch calls
3. **Token refresh** — `resetCsrf()` after login/logout (session changes)

```typescript
// api/client.ts — key exports:
export async function login(usr: string, pwd: string): Promise<void>
export async function logout(): Promise<void>
export async function getLoggedUser(): Promise<string | null>
export async function callMethod<T>(method: string, body?: object): Promise<T>
export async function listResource<T>(doctype: string, params?: ListParams): Promise<T[]>
export async function getResource<T>(doctype: string, name: string): Promise<T>
export async function createResource<T>(doctype: string, body: object): Promise<T>
export async function updateResource<T>(doctype: string, name: string, body: object): Promise<T>
export async function getCount(doctype: string, filters?: unknown[]): Promise<number>
export async function uploadFile(file: File): Promise<string>
```

**Error Extraction Pattern** (Frappe returns nested JSON errors):
```typescript
function extractError(err: unknown): string {
  const e = err as Record<string, unknown>;
  if (typeof e._server_messages === "string") {
    const msgs = JSON.parse(e._server_messages) as string[];
    return msgs.map(m => { try { return JSON.parse(m).message; } catch { return m; } }).join("; ");
  }
  if (typeof e.exception === "string") {
    const match = e.exception.match(/Duplicate entry '(.+?)' for key '(.+?)'/);
    if (match) return `Duplicate ${match[2]}: "${match[1]}" already exists`;
    return e.exception.split("\n")[0];
  }
  return typeof e.message === "string" ? e.message : "Operation failed";
}
```

### Authentication Context

```typescript
// components/AuthContext.tsx
interface AuthState {
  user: string | null;
  roles: string[];
  loading: boolean;
  refresh: () => Promise<RefreshResult>;  // Returns {user, roles, isAdmin} for immediate use
  isAdmin: boolean;
}
```

- `refresh()` returns data directly so login handlers can redirect without waiting for re-render
- `isAdmin` computed from: `user === "Administrator"` OR roles includes admin role
- Roles fetched via `frappe.client.get_list("Has Role", { parent: user })`

### Route Guards

```typescript
// ProtectedRoute: for customer portal
// - Not logged in → redirect to /portal/login?redirect=...
// - Is admin → redirect to /admin/dashboard (admins don't use portal)

// AdminRoute: for admin panel
// - Not logged in → redirect to /portal/login?redirect=...
// - Not admin → redirect to /portal/status
```

### Theme Context (Dark Mode)

```typescript
// components/ThemeContext.tsx
// - Reads from localStorage("cm-theme"), falls back to system preference
// - Adds/removes .dark class on document.documentElement
// - Provides { theme, toggle } via useTheme() hook
```

---

## 4. Page Patterns

### List Page Structure

Every list page follows this layout:
1. **Header**: Title + "New X" button
2. **Stat Cards**: 2-4 cards showing counts (Total, Open, Overdue, Closed)
3. **Filters**: Status dropdown + search input
4. **Table**: Sortable, clickable rows, action column with View/Edit/Close

```typescript
// Key patterns:
// - Stats loaded via getCount() on mount
// - Table rows clickable: onClick={() => navigate(`/admin/invoices/${r.name}`)}
// - Action column: stopPropagation() to prevent row click
// - View link → /admin/invoices/{name} (opens view mode)
// - Edit link → /admin/invoices/{name}?edit=1 (opens edit mode)
// - Closed items shown with opacity-50
// - Loading state shows <Loader2 className="animate-spin" />
// - Empty state shows centered text message
```

### Detail/Form Page Structure

Every form page supports **View Mode** and **Edit Mode**:

```typescript
const [editing, setEditing] = useState(isNew || searchParams.get("edit") === "1");

// View Mode: read-only display with labeled fields, status badge, Edit button
// Edit Mode: form with inputs, select fields, Save/Cancel buttons
// Activity Log: always shown below for existing records (ActivityTimeline component)
// Back link: always at top
```

**Field binding helper:**
```typescript
const set = (field: keyof FormData) => (
  e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement>
) => setForm(f => ({ ...f, [field]: e.target.value }));
```

### Admin Layout

- **Sidebar**: Fixed 56px wide, navigation links with active state highlighting
- **Header**: Theme toggle button + user profile dropdown (avatar, name, role, sign out)
- **Content**: `<Outlet />` with max-w-6xl container
- Dropdown closes on outside click via `useRef` + mousedown listener

---

## 5. Common Pitfalls & Solutions

| Pitfall | Solution |
|---------|----------|
| Tailwind v4 strips native date picker | Add `appearance: auto` for `input[type="date"]` |
| `.dark .bg-white` override affects toggle knobs | Use inline `style={{ backgroundColor: "white" }}` for elements that must stay white |
| Dark mode override affects radio/checkbox | Exclude with `:not([type="radio"]):not([type="checkbox"])` |
| Admin lands on customer portal | ProtectedRoute must check `isAdmin` and redirect to `/admin/dashboard` |
| Login redirect uses username check | Use `refresh()` return value with actual role data, not `email === "Administrator"` |
| CSRF token goes stale after login/logout | Call `resetCsrf()` after every `login()` and `logout()` |
| Frappe error messages are nested JSON | Parse `_server_messages` → JSON array → each item is JSON with `.message` |
| `bg-white` cards don't go dark | Global `.dark .bg-white { background-color: hsl(var(--card)) !important }` |
| Sidebar bg-white conflicts with global override | Use dynamic className based on theme variable instead of `bg-white` |

---

## 6. Workflow That Works

1. **Start with docs**: Write requirements, doctype specs, API design docs first
2. **Scaffold Frappe app**: `bench new-app`, create doctypes with fields + permissions + naming series
3. **Build API layer**: Whitelisted methods with role checks, before touching frontend
4. **API client**: Build `client.ts` with CSRF handling + all CRUD helpers
5. **Auth + Theme**: Set up AuthContext, ThemeContext, ProtectedRoute, AdminRoute
6. **Layout components**: AdminLayout (sidebar + header + outlet), PortalLayout
7. **List pages first**: Dashboard → List pages → they inform the detail page needs
8. **Detail pages**: View mode first, then add edit mode, then activity log
9. **Dark mode last**: CSS variable approach with global overrides, not per-component `dark:` classes
10. **Test login flows**: Admin → portal redirect, customer → admin block, logout → re-login

---

## 7. Dependencies (package.json)

```json
{
  "dependencies": {
    "@tailwindcss/vite": "^4.x",
    "tailwindcss": "^4.x",
    "react": "^19.x",
    "react-dom": "^19.x",
    "react-router-dom": "^7.x",
    "lucide-react": "latest",
    "class-variance-authority": "latest",
    "clsx": "latest",
    "tailwind-merge": "latest"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "latest",
    "typescript": "latest",
    "vite": "latest"
  }
}
```

---

## 8. Docker Deployment

```yaml
# docker-compose.yml services:
# - mariadb:10.11 (port 3307:3306, utf8mb4)
# - redis-cache:7-alpine
# - redis-queue:7-alpine
# - backend: Frappe bench (port 8000 + 9000 websocket)
# - frontend: React SPA via Nginx (port 80)
```

Backend Dockerfile: Install Frappe bench, copy app, run `bench setup`, `bench build`.
Frontend Dockerfile: `npm ci && npm run build`, serve `dist/` via Nginx with SPA fallback (`try_files $uri /index.html`).
