# Integrating UMD CAS Authentication into a Web App

This guide documents how the IFAD Portal integrates UMD's Central Authentication Service (CAS) for student Single Sign-On. Use it as a reference for wiring CAS into any new UMD web application.

The IFAD Portal is a React + AWS Lambda app, but the CAS concepts apply to any frontend/backend stack.

---

## How UMD CAS Works

CAS is a redirect-based SSO protocol. The flow is:

```
User clicks "Login"
    ↓
Your app redirects to UMD CAS with ?service=<your-url>
    ↓
UMD authenticates the user (Shibboleth/UMD credentials)
    ↓
CAS redirects back to your app with ?ticket=ST-xxxxxx
    ↓
Your BACKEND calls CAS /serviceValidate with the ticket
    ↓
CAS returns an XML response with the user's identity
    ↓
Your backend issues a session/JWT to the frontend
```

**Critical rule:** ticket validation must happen server-side. Never call CAS from the browser directly — the ticket is single-use and the validation must come from a trusted server.

---

## Step 1 — Register Your Service URL with UMD IT

UMD's CAS server only accepts tickets for pre-approved `service=` URLs. Before any code will work:

1. Submit a ticket to UMD IT (`help.umd.edu`) requesting CAS service registration
2. Provide every URL you need: production domain, staging domain, and local dev URL
3. For local development, request registration of `http://localtest.dev.umd.edu:5173` — this hostname resolves to `127.0.0.1` but satisfies CAS's domain requirements

> Plain `http://localhost` cannot be registered as a CAS service URL. You must use `localtest.dev.umd.edu` locally.

The CAS base URL for UMD is:
```
https://shib.idm.umd.edu/shibboleth-idp/profile/cas
```

---

## Step 2 — Configure Your Dev Server

Because the dev server must run on `localtest.dev.umd.edu`, configure it to bind to that hostname.

**Vite (`vite.config.ts`):**
```ts
export default defineConfig({
  server: {
    host: 'localtest.dev.umd.edu',
    port: 5173,
    strictPort: true,
    allowedHosts: ['localhost', '127.0.0.1', 'localtest.dev.umd.edu'],
  },
});
```

Access your dev server at `http://localtest.dev.umd.edu:5173` instead of `http://localhost:5173`.

---

## Step 3 — Frontend: Redirect to CAS Login

When the user clicks "Login", redirect their browser to the CAS login page with your app's URL as the `service` parameter. CAS will redirect back to this URL after authentication.

```ts
const CAS_BASE_URL = 'https://shib.idm.umd.edu/shibboleth-idp/profile/cas';

function getServiceURL(): string {
  const hostname = window.location.hostname;
  const port = window.location.port;

  // CAS can't use bare localhost — swap it for the registered dev alias
  if (hostname === 'localhost' || hostname === '127.0.0.1') {
    return `http://localtest.dev.umd.edu:${port}`;
  }

  return window.location.origin;
}

function loginWithCAS(): void {
  const serviceUrl = getServiceURL();
  const loginUrl = `${CAS_BASE_URL}/login?service=${encodeURIComponent(serviceUrl)}`;
  window.location.href = loginUrl;
}
```

---

## Step 4 — Frontend: Detect the Ticket on Return

After CAS authenticates the user it redirects back to your app with `?ticket=ST-xxxxx` appended to the URL. Detect this on app load.

In React, the cleanest place is a component that wraps your routes and runs an effect on every navigation:

```tsx
// Runs on every render; detects the CAS ticket and kicks off validation
const CASCallbackHandler: React.FC = () => {
  const navigate = useNavigate();
  const { user, isLoading } = useAuth();

  useEffect(() => {
    if (isLoading || user) return; // already authenticated

    const params = new URLSearchParams(window.location.search);
    const ticket = params.get('ticket');
    if (!ticket) return;

    // Remove ticket from URL immediately — it's single-use
    const url = new URL(window.location.href);
    url.searchParams.delete('ticket');
    window.history.replaceState({}, document.title, url.pathname + url.hash);

    // Send ticket to your backend for validation
    validateTicket(ticket).then((user) => {
      if (user) {
        navigate('/dashboard', { replace: true });
      } else {
        navigate('/login', { replace: true });
      }
    });
  }, [navigate, user, isLoading]);

  return null; // renders nothing; just handles the redirect
};
```

Place this inside your router so it has access to `useNavigate`:

```tsx
function App() {
  return (
    <BrowserRouter>
      <CASCallbackHandler />
      <Routes>
        {/* your routes */}
      </Routes>
    </BrowserRouter>
  );
}
```

---

## Step 5 — Frontend: Send the Ticket to Your Backend

```ts
async function validateTicket(ticket: string): Promise<User | null> {
  const serviceUrl = getServiceURL();

  const resp = await fetch('/auth/cas/validate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ticket, service: serviceUrl }),
  });

  if (!resp.ok) return null;

  const json = await resp.json();
  const { token, user } = json.data;

  // Store the JWT your backend issued
  localStorage.setItem('auth_token', token);
  localStorage.setItem('auth_user', JSON.stringify(user));

  return user;
}
```

**Important:** the `service` value sent to your backend must exactly match the one originally passed to CAS. Even a trailing slash difference will cause validation to fail.

---

## Step 6 — Backend: Validate the Ticket with CAS

This is the critical server-side step. Your backend calls CAS's `serviceValidate` endpoint and parses the XML response.

**Node.js / TypeScript example (AWS Lambda, but works anywhere):**

Install the XML parser:
```bash
npm install fast-xml-parser jsonwebtoken
```

```ts
import { XMLParser } from 'fast-xml-parser';
import jwt from 'jsonwebtoken';

const CAS_BASE_URL = process.env.CAS_BASE_URL!; // https://shib.idm.umd.edu/shibboleth-idp/profile/cas
const JWT_SECRET = process.env.JWT_SECRET!;

async function handleCasValidate(body: { ticket: string; service: string }) {
  const { ticket, service } = body;
  if (!ticket || !service) {
    throw new Error('ticket and service are required');
  }

  // Call CAS serviceValidate
  const url = `${CAS_BASE_URL}/serviceValidate` +
    `?ticket=${encodeURIComponent(ticket)}` +
    `&service=${encodeURIComponent(service)}`;

  const res = await fetch(url, {
    method: 'GET',
    headers: { Accept: 'application/xml,text/xml' },
  });
  const xml = await res.text();

  if (!res.ok) {
    throw new Error(`CAS HTTP error ${res.status}`);
  }

  // Parse XML response
  const parser = new XMLParser({ ignoreAttributes: false, removeNSPrefix: true });
  const parsed = parser.parse(xml);
  const serviceResponse = parsed?.serviceResponse || parsed;
  const success = serviceResponse?.authenticationSuccess;
  const failure = serviceResponse?.authenticationFailure;

  if (!success) {
    const code = typeof failure === 'object' ? failure['@_code'] : 'UNKNOWN';
    throw new Error(`CAS rejected ticket: ${code}`);
  }

  // Extract user identity from XML attributes
  const directoryId = String(success.user || '').trim();
  const attrs = success.attributes || {};
  const firstName = String(attrs.givenName || '').trim();
  const lastName = String(attrs.sn || '').trim();
  const email = String(attrs.mail || `${directoryId}@umd.edu`).trim();

  if (!directoryId) {
    throw new Error('CAS response missing user identifier');
  }

  // Issue a JWT for your app
  const userPayload = { directoryId, email, firstName, lastName, role: 'student' };
  const token = jwt.sign(userPayload, JWT_SECRET, { expiresIn: '6h' });

  return { token, user: userPayload };
}
```

**What the CAS XML response looks like on success:**
```xml
<cas:serviceResponse xmlns:cas="http://www.yale.edu/tp/cas">
  <cas:authenticationSuccess>
    <cas:user>jsmith</cas:user>
    <cas:attributes>
      <cas:givenName>John</cas:givenName>
      <cas:sn>Smith</cas:sn>
      <cas:mail>jsmith@umd.edu</cas:mail>
    </cas:attributes>
  </cas:authenticationSuccess>
</cas:serviceResponse>
```

**On failure:**
```xml
<cas:serviceResponse xmlns:cas="http://www.yale.edu/tp/cas">
  <cas:authenticationFailure code="INVALID_TICKET">
    Ticket ST-xxx not recognized
  </cas:authenticationFailure>
</cas:serviceResponse>
```

---

## Step 7 — Logout

To fully log the user out of UMD SSO (not just your app), redirect to the CAS logout endpoint:

```ts
function logout(): void {
  // Clear your app's local session first
  localStorage.removeItem('auth_token');
  localStorage.removeItem('auth_user');

  // Redirect to CAS logout; the url param sends the user back to your app after
  const returnUrl = encodeURIComponent(window.location.origin);
  window.location.href =
    `https://shib.idm.umd.edu/shibboleth-idp/profile/cas/logout?url=${returnUrl}`;
}
```

If you only clear local state without hitting the CAS logout URL, the user's UMD session remains active and they won't be prompted to log in again when they return.

---

## Environment Variables

| Variable | Where | Value |
|---|---|---|
| `CAS_BASE_URL` | Backend | `https://shib.idm.umd.edu/shibboleth-idp/profile/cas` |
| `JWT_SECRET` | Backend | A long, random secret string |
| `VITE_API_URL` | Frontend `.env` | Your backend API base URL |

---

## Common Gotchas

**Ticket mismatch error**
The `service` URL sent during validation must byte-for-byte match what was sent during the initial login redirect. Store the service URL or re-derive it the same way in both places.

**Ticket already used**
CAS tickets are single-use. If your callback handler fires twice (e.g. React Strict Mode double-invokes effects in dev), the second validation will fail. Guard with a ref or state flag so validation only runs once per ticket.

**`localhost` not working**
UMD IT cannot register bare `localhost`. Use `localtest.dev.umd.edu` and configure your dev server's `host` accordingly.

**XML namespace issues**
The CAS XML uses a `cas:` namespace prefix. Some XML parsers require you to handle both `cas:user` and `user` selectors, or use a library option like `removeNSPrefix: true` (fast-xml-parser) to strip namespaces automatically.

**Redirect loop after login**
If your app redirects to the login page on load before checking for a `?ticket=` in the URL, the ticket gets discarded. Always check for the ticket before deciding whether to redirect.

---

## Summary Checklist

- [ ] Service URL(s) registered with UMD IT (production + staging + `localtest.dev.umd.edu`)
- [ ] Dev server configured to bind to `localtest.dev.umd.edu`
- [ ] Login button redirects to `{CAS_BASE_URL}/login?service={encodedServiceUrl}`
- [ ] App detects `?ticket=` on load and removes it from the URL immediately
- [ ] Ticket sent to backend; backend calls CAS `/serviceValidate`
- [ ] Backend parses XML, extracts `directoryId` / `givenName` / `sn` / `mail`
- [ ] Backend issues a JWT or session and returns it to the frontend
- [ ] Logout redirects to `{CAS_BASE_URL}/logout`
- [ ] `JWT_SECRET` and `CAS_BASE_URL` set as environment variables on the server
