# Link Management – Frontend (React SPA)

> **Technical README for Interview Preparation**
> Written from codebase inspection of the `ghanshyam` (frontend) repository.

---

## 1. Overview

The frontend is a **React 18 Single Page Application (SPA)** built with **Vite** as the build tool. It serves as the UI layer for all 7 user roles (Admin, Manager, Team, Writer, Blogger, Accountant, Client), each with their own dedicated routing namespace and layout.

**Tech stack (verified from `package.json`):**

| Technology | Version | Purpose |
|---|---|---|
| React | 18.2.0 | UI component framework |
| Vite | 5.0 | Build tool + dev server |
| React Router DOM | 6.8 | Client-side routing |
| Axios | 1.13 | HTTP client |
| Socket.io-client | 4.8.3 | WebSocket real-time events |
| Tailwind CSS | 3.4 | Utility-first CSS |
| Framer Motion | 10.16 | Animations |
| Chart.js + react-chartjs-2 | 4.5 | Dashboard charts |
| Quill / react-quill | 2.0 | Rich text content editor (for writers) |
| react-hot-toast | 2.6 | Toast notifications |
| lucide-react | 0.263 | Icon library |
| xlsx | 0.18 | Client-side Excel file generation/reading |

---

## 2. Project Structure

```
src/
├── App.jsx              — Root router (all role routes defined here)
├── main.jsx             — Entry point, wraps in AuthProvider
├── index.css            — Global styles / Tailwind directives
├── lib/
│   └── api.js           — Centralized Axios instance + all API functions
├── auth/
│   └── AuthContext.jsx  — Authentication context (login, logout, token, role)
├── context/             — Shared React contexts
├── hooks/               — Custom React hooks
├── components/          — Shared UI components
├── pages/               — Public pages (Landing, Login, Signup)
├── admin/               — Admin panel pages
├── manager/             — Manager panel pages
├── teams/               — Team member pages
├── writer/              — Writer pages
├── blogger/             — Blogger pages
├── accountant/          — Accountant pages
├── client/              — Client pages
└── utils/               — Frontend utility functions
```

---

## 3. Routing Architecture

**All routes are defined in `src/App.jsx`** using React Router DOM v6's declarative `<Routes>` and `<Route>` components.

**Role-based route namespacing:**

| URL Prefix | Role | Layout Component |
|---|---|---|
| `/` | Public | None |
| `/login` | Public | None |
| `/signup` | Public | None |
| `/admin/*` | Admin | `AdminLayout` |
| `/manager/*` | Manager | `ManagerRoutes` |
| `/teams/*` | Team | `TeamsApp` |
| `/writer/*` | Writer | `WriterLayout` |
| `/blogger/*` | Blogger | `BloggerLayout` |
| `/accountant/*` | Accountant | `AccountantLayout` |
| `/client/*` | Client | `ClientLayout` |

**Route protection (`RequireAuth`):**
```jsx
export function RequireAuth({ roles = [], children }) {
  const { isAuthenticated, role } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" />;
  if (roles.length && !roles.includes(role)) return <Navigate to="/" replace />;
  return children;
}
```
`RequireAuth` wraps role-specific layouts. After login, users are redirected to their role's dashboard based on `data.user.role` returned from the API.

---

## 4. Authentication (Frontend)

**`src/auth/AuthContext.jsx`** manages all auth state:

- **State:** `user`, `role`, `token` — initialized from `localStorage`
- **Persistence:** All three are synced to `localStorage` via `useEffect`
- **Token keys:** `authToken`, `authRole`, `authUser`

**Login flow:**
```jsx
const login = async (email, password) => {
  const data = await authAPI.login(email, password);
  setToken(data.token);      // Stored in localStorage.authToken
  setUser(data.user);        // Stored in localStorage.authUser (JSON)
  setRole(data.user.role);   // Stored in localStorage.authRole
}
```

**Session validation on app mount:**
```jsx
useEffect(() => {
  if (token && !user) {
    authAPI.getCurrentUser()  // GET /api/auth/me
      .then(data => setUser(data.user))
      .catch(() => logout());  // Token invalid → clear auth
  }
}, []);
```

**Logout:** Clears all three `localStorage` keys and resets state. No server call required (JWT is stateless).

**Admin impersonation:** `impersonateLogin(data)` — admin can log in as any user via `POST /api/admin/users/:id/impersonate` without needing the user's password.

---

## 5. API Layer (`src/lib/api.js`)

All HTTP communication goes through a single Axios instance configured in `src/lib/api.js`.

**Base configuration:**
```js
const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:5001/api',
  timeout: 30000,
  withCredentials: true,
});
```

**Request interceptor — automatic JWT injection:**
```js
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

**Response interceptor — global error handling:**
- HTTP 401: Clears localStorage, redirects to `/login`
- Extracts human-readable error message from various error response shapes

**API module exports (organized by role):**
- `authAPI` — login, register, getCurrentUser, changePassword, getMyPermissions
- `adminAPI` — users, sites, orders, wallets, withdrawals, link inspection
- `managerAPI` — orders, tasks, writers, bloggers, workflow actions
- `teamAPI` — tasks, websites, submissions
- `writerAPI` — tasks, content submission
- `bloggerAPI` — tasks, wallet, sites, withdrawals
- `accountantAPI` — wallet, withdrawal approvals
- `clientAPI` — orders, payments, sites, transactions

---

## 6. Real-Time Integration (Socket.io)

The frontend uses `socket.io-client` to receive live updates from the backend.

**Connection:** Established when the user is authenticated. The socket connects to the same server as the API (`VITE_API_URL` or `localhost:5001`).

**Rooms joined from frontend:**
- `orders-list` — listen for order creation/update events on list pages
- `order-{orderId}` — listen for per-order updates on the detail page
- `chat_{threadId}` — for in-platform messaging

**Events received:**
- `order-created` — append new order to list
- `order-updated` / `order-detail-updated` — update order in state
- `workflow-changed` — update status badge in real-time
- `blogger-assigned` — blogger's dashboard updates
- `url-submitted` — manager sees blogger's submission appear live
- `vendor_{userId}_new_task` — blogger receives new task notification
- `user_typing` / `user_stop_typing` — chat typing indicators

---

## 7. State Management

**No Redux or Zustand.** The application uses React's built-in primitives:

- **`AuthContext` (`useAuth` hook):** Global auth state — token, user, role, login/logout functions
- **Local component state (`useState`):** Form data, list data, loading/error states
- **`useEffect` + API calls:** Data fetching on component mount
- **React Context:** Used for auth only; other state is local

This is a simple, lightweight approach suitable for a role-scoped SPA where different role panels are largely independent of each other.

---

## 8. Key UI Features by Role

### Admin Panel (`/admin`)
- **Dashboard** with blogger stats, order counts
- **User management** — create, update, delete, impersonate users
- **Sites management** — bulk Excel upload, list, edit, delete, restore deleted sites
- **Link inspection** — view all submitted backlinks with status (Live / Not Found / Issue), trigger manual re-check, start bulk check
- **Wallet management** — view all blogger wallets, payment history, approve/reject withdrawals, download PDF invoices
- **Content management** — Careers, FAQs, Videos, Countries (CMS features)
- **Financial reports**

### Manager Panel (`/manager`)
- **Order creation** — select sites, set target URLs, anchor text, article titles, route directly to team/writer/blogger
- **Workflow control** — approve/reject team submissions, assign writers, approve content, push to bloggers
- **Task review** — review blogger submissions, finalize and credit, or send back
- **Client order management** — review client orders, push to bloggers or writers

### Blogger Panel (`/blogger`)
- **Orders dashboard** — see tasks assigned to their sites
- **Task detail** — view requirements, submit live URL
- **My Sites** — manage their registered websites, submit bulk site listings
- **Wallet** — view earnings, request withdrawal, download invoices
- **Threads** — in-platform messaging

### Writer Panel (`/writer`)
- **Assigned tasks** — view content requirements (target URL, anchor, article title)
- **Quill rich text editor** — write and submit content
- **Completed orders** — history view
- **Rejected notifications** — see why submissions were returned

### Client Panel (`/client`)
- **Create order** — select sites from catalogue, specify requirements
- **Razorpay top-up** — add credits to wallet
- **View orders** — track order status
- **Link completed** — see live delivered backlinks
- **Transactions** — wallet history

### Accountant Panel (`/accountant`)
- Same wallet views as Admin but limited to wallet/withdrawal management only

---

## 9. Build & Development

**Development:**
```bash
npm run dev     # Vite dev server with HMR
```

**Production build:**
```bash
npm run build   # Outputs to dist/
npm run start   # serve -s dist (static file server)
```

**Environment variable:**
```
VITE_API_URL=https://your-backend.railway.app/api
```
Vite exposes only `VITE_` prefixed variables to the client bundle.

---

## 10. Security Considerations (Frontend)

| Topic | Implementation |
|---|---|
| Token storage | `localStorage` — accessible to JavaScript; XSS risk exists |
| Automatic logout on 401 | Axios interceptor clears auth and redirects to `/login` |
| Route protection | `RequireAuth` component prevents unauthorized route access |
| Role enforcement | Both route-level (`RequireAuth roles=[]`) and server-side |
| CORS | Backend has an explicit allowlist; frontend must match an allowed origin |

> **XSS Risk:** Storing JWT in `localStorage` is vulnerable to XSS attacks. An alternative would be `httpOnly` cookies, which are inaccessible to JavaScript. For this platform, the trade-off favors simplicity.

---

## 11. Interview Questions (Frontend-Specific)

**Q: How is authentication state shared across components?**
Evidence: `AuthContext.jsx` — React Context + `useAuth()` hook. State initialized from `localStorage`, synced back on change.

**Q: How does the frontend handle expired JWT?**
Evidence: Axios response interceptor detects 401, clears localStorage, redirects to `/login`.

**Q: How does real-time update work when a manager assigns a task?**
Evidence: Backend emits `vendor_{id}_new_task` via Socket.io. Frontend listens and updates state/UI without a page refresh.

**Q: Why Vite over Create React App?**
Vite uses native ES modules in development — much faster hot module replacement. CRA uses Webpack which is slower for large apps.

**Q: How do you prevent a manager from accessing blogger routes?**
Evidence: `RequireAuth` checks `role` from `AuthContext`. Route mounted at `/blogger/*` only renders for users with `role = 'vendor'/'blogger'`. Additionally, the backend enforces the same check via `authorize()` middleware.

**Q: How does the bulk site upload work?**
Evidence: `adminAPI.uploadSitesExcel()` sends `multipart/form-data` with the Excel file to `POST /api/admin/sites/upload-excel`. Backend parses with `xlsx` library and validates each row.

**Q: How is the Quill editor used?**
Evidence: `react-quill` component in writer pages allows rich-text content submission. Content is serialized and sent to the API as part of the task submission payload.
