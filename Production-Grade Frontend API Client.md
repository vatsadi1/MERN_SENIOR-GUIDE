# Production-Grade Frontend API Client --- Notes & Implementation

**Stack:** React + Vite + JavaScript + native `fetch`\
**Authentication design:** JWT in `HttpOnly` cookies (access token +
refresh token)\
**Goal:** A reusable API layer that handles requests consistently,
restores sessions, handles expiry safely, and keeps UI components
focused on UI.

------------------------------------------------------------------------

## 1. What an API client is responsible for

An API client is a shared frontend service between React components and
the backend.

A component should describe **what it wants to do**:

``` js
const medicines = await apiClient.get('/medicines');
```

The API client handles the common transport concerns:

-   Base URL and endpoint construction
-   HTTP method, headers, and request body serialization
-   Cookie credentials
-   Response parsing
-   Consistent HTTP/network/timeout errors
-   Authentication recovery (refresh + one retry)
-   Request cancellation and timeouts
-   Safe logging
-   Consistent behavior across all features

It does **not** replace backend validation, authorization, database
constraints, or business rules. The backend remains the security
boundary.

### Request lifecycle

1.  Component calls `apiClient`.
2.  Client builds the URL and request configuration.
3.  Browser sends the request, including eligible cookies.
4.  Backend authenticates and processes it.
5.  Client parses the response.
6.  Client returns data or throws a structured error.
7.  UI handles loading, success, empty state, and failure.

------------------------------------------------------------------------

## 2. Recommended project structure

``` text
src/
  features/
    shared/
      services/
        apiClient.js
      errors/
        ApiError.js
  app/
    auth/
      AuthProvider.jsx
```

Keep the API client in a shared location. Feature-specific API functions
can wrap it:

``` js
// src/features/medicines/services/medicineApi.js
import apiClient from '../../shared/services/apiClient';

export const medicineApi = {
  list: (params = {}) =>
    apiClient.get('/medicines', { query: params }),

  getById: (id) =>
    apiClient.get(`/medicines/${encodeURIComponent(id)}`),

  create: (payload) =>
    apiClient.post('/medicines', payload),
};
```

Components should normally call `medicineApi.list()` rather than
repeating URL construction and fetch configuration.

------------------------------------------------------------------------

## 3. Cookie-based JWT authentication

With cookie authentication, the browser stores and sends cookies.
JavaScript should not read an `HttpOnly` token.

### Login flow

1.  Frontend sends email/password to `POST /auth/login`.
2.  Backend verifies credentials.
3.  Backend sends `Set-Cookie` headers for the access token and refresh
    token.
4.  Browser stores the cookies if their attributes and browser policies
    allow it.
5.  Frontend stores only the user/session display information in React
    state---not the token.

### Normal protected API request

The frontend uses:

``` js
fetch(url, {
  credentials: 'include',
});
```

The browser attaches eligible cookies automatically. Backend middleware
reads and verifies the access-token cookie.

### Important distinction

-   **React state / Context / Redux:** tells the UI which user is
    currently loaded and which screens to display.
-   **HttpOnly cookie:** carries the authentication credential to the
    backend.
-   **Backend authorization:** decides whether the authenticated user
    may perform the requested action.

Never treat a frontend role or `isLoggedIn` state as authorization. The
backend must check permissions on every protected request.

### Cookie and CORS checklist

Backend and frontend must be configured together:

-   CORS must allow the exact frontend origin; do not use
    `Access-Control-Allow-Origin: *` with credentials.
-   Backend must enable credentialed CORS.
-   Cookie `HttpOnly` should be enabled for authentication cookies.
-   `Secure` should be enabled in production over HTTPS.
-   Choose `SameSite` based on whether frontend and API are same-site or
    cross-site.
-   For genuinely cross-site cookies, browsers generally require
    `SameSite=None; Secure`.
-   Set an appropriate cookie `Path`, `Domain` (avoid broad domains
    unless required), and expiry.
-   Clear cookies using matching `Path` and `Domain` attributes.
-   `credentials: 'include'` does not bypass browser cookie rules or
    third-party-cookie restrictions.

### CSRF protection

Cookies are automatically attached by the browser, which creates CSRF
considerations. For state-changing operations:

-   Use an appropriate `SameSite` policy.
-   Validate `Origin` (and, where appropriate, `Referer`) on the
    backend.
-   Use a CSRF token pattern when required by the deployment and threat
    model.
-   Do not assume CORS alone is CSRF protection. CORS controls whether
    JavaScript can read a response; it is not a complete request-forgery
    defense.

------------------------------------------------------------------------

## 4. Access token and refresh token lifecycle

Use short-lived access tokens and longer-lived refresh tokens. Exact
lifetimes are a security/product decision; choose them with the backend
team rather than copying example values blindly.

### Reactive refresh

1.  A protected API returns `401`.
2.  Client calls `POST /auth/refresh`.
3.  Browser sends the refresh-token cookie.
4.  Backend validates the refresh token and issues a new access token
    (and, if using rotation, a new refresh token).
5.  Client retries the original request once.
6.  If refresh fails, client clears local auth state and asks the user
    to log in again.

### Proactive refresh

The app may refresh shortly before access-token expiry to reduce visible
interruptions. This is an optimization, not the only recovery path.

-   Backend may return `expiresIn` (seconds) as non-secret metadata, or
    the frontend may read the JWT `exp` claim if the token is
    intentionally available to JavaScript. With `HttpOnly` cookies, the
    frontend cannot decode the access token directly.
-   Schedule refresh with a safety margin; account for clock skew and
    network delay.
-   Browser timers are not reliable in background tabs, suspended
    devices, or after the browser closes.
-   Always keep reactive `401` recovery as a fallback.
-   On a fresh page load, restore the session with a session endpoint
    such as `GET /auth/me`; if access token is expired, refresh and
    retry `/auth/me`, or use a deliberately designed bootstrap refresh
    endpoint.
-   Do not assume a timer ran while the browser was closed.

### Do not expose refresh tokens

With this design, the refresh token should remain in a secure `HttpOnly`
cookie. Do not return it in JSON or store it in
`localStorage`/`sessionStorage`.

### Refresh token rotation and revocation

Backend responsibilities include:

-   Validate refresh token and its expiry.
-   Rotate refresh tokens where the chosen design requires it.
-   Detect reuse of an already-rotated token and revoke the affected
    token family/session as appropriate.
-   Support logout/revocation.
-   Store only a hash of opaque refresh tokens if using opaque tokens.
-   Define concurrent-refresh behavior (for example, a short
    grace/idempotency strategy) so legitimate simultaneous tabs do not
    cause accidental session invalidation.
-   Rate-limit sensitive authentication endpoints.

Frontend code cannot make an insecure refresh-token backend secure.

------------------------------------------------------------------------

## 5. Multiple simultaneous 401 responses

Imagine five requests receive `401` at almost the same time.

**Bad behavior:** all five independently call `/auth/refresh`.

**Better behavior in one tab:** use a single shared refresh promise
(single-flight). All failed requests wait for the same refresh
operation. If it succeeds, each original request is retried once.

A single-flight promise only coordinates requests inside the same
JavaScript context/tab.

### Multiple tabs

Each tab has a separate JavaScript context, so a module-level promise
does not coordinate tabs. Options include:

-   `BroadcastChannel` to notify other tabs that refresh is in progress
    or has completed.
-   Web Locks API to coordinate refresh work across tabs in supported
    browsers.
-   A backend refresh design that safely handles concurrent requests and
    rotation races.

Do not rely on frontend coordination alone. The backend must define safe
concurrent refresh behavior. Test two or more tabs refreshing at the
same time.

------------------------------------------------------------------------

## 6. Production-oriented `apiClient.js`

This implementation uses native `fetch`, supports JSON and `FormData`,
parses responses safely, supports timeouts and cancellation, and
performs one refresh attempt after a `401`.

**Assumed backend contract:**

-   `POST /auth/refresh` refreshes the session using the refresh-token
    cookie.
-   It returns a successful 2xx response and sets updated authentication
    cookies.
-   `POST /auth/logout` clears/revokes the session.
-   `GET /auth/me` returns the current user.
-   All cookie/CORS/CSRF settings are configured on the backend.

Adjust endpoint paths and error response parsing to match your actual
API contract.

``` js

// src/features/shared/services/apiClient.js

const RAW_API_URL = import.meta.env.VITE_API_URL || 'http://localhost:4000/api/v1';
const API_BASE_URL = RAW_API_URL.replace(/\/+$/, '');

const DEFAULT_TIMEOUT_MS = 15_000;
const REFRESH_ENDPOINT = '/auth/refresh';
const AUTH_ENDPOINTS_WITHOUT_AUTO_REFRESH = new Set([
  '/auth/login',
  '/auth/register',
  '/customers/auth/send-otp',
  '/customers/auth/verify-otp',
  REFRESH_ENDPOINT,
]);

// In-memory token storage (Fallback & Fast lookup)
let currentAccessToken = null;
let refreshPromise = null;

export const tokenStorage = {
  get: () => currentAccessToken || localStorage.getItem('access_token'),
  set: (token) => {
    currentAccessToken = token;
    if (token) localStorage.setItem('access_token', token);
    else localStorage.removeItem('access_token');
  },
  clear: () => {
    currentAccessToken = null;
    localStorage.removeItem('access_token');
  },
};

export class ApiError extends Error {
  constructor(message, {
    status = 0,
    data = null,
    url = '',
    method = 'GET',
    code = 'API_ERROR',
  } = {}) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.data = data;
    this.url = url;
    this.method = method;
    this.code = code;
  }
}

function buildUrl(endpoint, query) {
  const normalizedEndpoint = endpoint.startsWith('/') ? endpoint : `/${endpoint}`;

  // Safe URL creation for relative production paths (e.g., /api/v1)
  const base = API_BASE_URL.startsWith('http')
    ? API_BASE_URL
    : `${typeof window !== 'undefined' ? window.location.origin : 'http://localhost'}${API_BASE_URL}`;

  const url = new URL(`${base}${normalizedEndpoint}`);

  if (query && typeof query === 'object') {
    for (const [key, value] of Object.entries(query)) {
      if (value === undefined || value === null || value === '') continue;

      if (Array.isArray(value)) {
        value.forEach((item) => url.searchParams.append(key, String(item)));
      } else if (value instanceof Date) {
        url.searchParams.set(key, value.toISOString());
      } else if (typeof value === 'object') {
        url.searchParams.set(key, JSON.stringify(value));
      } else {
        url.searchParams.set(key, String(value));
      }
    }
  }

  return url;
}

async function parseResponse(response) {
  if (response.status === 204 || response.status === 205) return null;

  const contentType = response.headers.get('content-type') || '';
  if (contentType.includes('application/json')) {
    try {
      return await response.json();
    } catch {
      return null;
    }
  }

  const text = await response.text().catch(() => '');
  return text || null;
}

function getErrorMessage(data, response) {
  if (typeof data === 'string' && data.trim()) return data;
  return (
    data?.error?.message ||
    data?.message ||
    `Request failed with HTTP ${response.status}`
  );
}

function createAbortSignal(externalSignal, timeoutMs) {
  const controller = new AbortController();
  let timedOut = false;

  const timeoutId = timeoutMs > 0
    ? setTimeout(() => {
        timedOut = true;
        controller.abort();
      }, timeoutMs)
    : null;

  const abortFromCaller = () => controller.abort();

  if (externalSignal) {
    if (externalSignal.aborted) {
      controller.abort();
    } else {
      externalSignal.addEventListener('abort', abortFromCaller, { once: true });
    }
  }

  return {
    signal: controller.signal,
    didTimeout: () => timedOut,
    cleanup: () => {
      if (timeoutId) clearTimeout(timeoutId);
      externalSignal?.removeEventListener('abort', abortFromCaller);
    },
  };
}

function isFormData(value) {
  return typeof FormData !== 'undefined' && value instanceof FormData;
}

function prepareBody(body, headers) {
  if (body === undefined || body === null) return undefined;

  if (
    isFormData(body) ||
    body instanceof Blob ||
    body instanceof ArrayBuffer ||
    typeof body === 'string'
  ) {
    return body;
  }

  if (!headers.has('Content-Type')) {
    headers.set('Content-Type', 'application/json');
  }

  return JSON.stringify(body);
}

function isAuthEndpoint(endpoint) {
  const normalized = endpoint.startsWith('/') ? endpoint : `/${endpoint}`;
  return AUTH_ENDPOINTS_WITHOUT_AUTO_REFRESH.has(normalized);
}

function notifySessionExpired() {
  tokenStorage.clear();
  if (typeof window !== 'undefined') {
    window.dispatchEvent(new CustomEvent('auth:session-expired'));
  }
}

async function performRefresh() {
  if (!refreshPromise) {
    refreshPromise = (async () => {
      const url = buildUrl(REFRESH_ENDPOINT);

      const response = await fetch(url, {
        method: 'POST',
        credentials: 'include',
        headers: {
          Accept: 'application/json',
          'Content-Type': 'application/json',
        },
      });

      if (!response.ok) {
        notifySessionExpired();
        throw new ApiError('Your session has expired. Please log in again.', {
          status: response.status,
          data: await parseResponse(response),
          url: url.toString(),
          method: 'POST',
          code: 'SESSION_EXPIRED',
        });
      }

      const resData = await parseResponse(response);
      const newToken = resData?.accessToken || resData?.token || resData?.data?.token;
      if (newToken) {
        tokenStorage.set(newToken);
      }

      return newToken || true;
    })().finally(() => {
      refreshPromise = null;
    });
  }

  return refreshPromise;
}

async function executeRequest(endpoint, options, { retryAfterRefresh = true } = {}) {
  const {
    method = 'GET',
    body,
    query,
    headers: suppliedHeaders,
    signal: externalSignal,
    timeoutMs = DEFAULT_TIMEOUT_MS,
    ...fetchOptions
  } = options;

  const url = buildUrl(endpoint, query);
  const normalizedMethod = method.toUpperCase();
  const headers = new Headers(suppliedHeaders || {});

  if (!headers.has('Accept')) {
    headers.set('Accept', 'application/json');
  }

  // Inject Bearer token if available and not already passed
  const token = tokenStorage.get();
  if (token && !headers.has('Authorization')) {
    headers.set('Authorization', `Bearer ${token}`);
  }

  const requestBody = prepareBody(body, headers);
  const abort = createAbortSignal(externalSignal, timeoutMs);

  let response;

  try {
    response = await fetch(url, {
      ...fetchOptions,
      method: normalizedMethod,
      headers,
      body: requestBody,
      signal: abort.signal,
      credentials: 'include',
    });
  } catch (error) {
    if (abort.didTimeout()) {
      throw new ApiError('The request timed out. Please try again.', {
        url: url.toString(),
        method: normalizedMethod,
        code: 'TIMEOUT',
      });
    }

    if (externalSignal?.aborted) {
      throw new ApiError('The request was cancelled.', {
        url: url.toString(),
        method: normalizedMethod,
        code: 'ABORTED',
      });
    }

    throw new ApiError('Network error. Check your connection and try again.', {
      url: url.toString(),
      method: normalizedMethod,
      code: 'NETWORK_ERROR',
    });
  } finally {
    abort.cleanup();
  }

  const data = await parseResponse(response);

  if (response.status === 401 && retryAfterRefresh && !isAuthEndpoint(endpoint)) {
    try {
      await performRefresh();
      // Retry once with new token
      return executeRequest(endpoint, options, { retryAfterRefresh: false });
    } catch (refreshError) {
      notifySessionExpired();
      throw refreshError instanceof ApiError
        ? refreshError
        : new ApiError('Your session has expired. Please log in again.', {
            status: 401,
            url: url.toString(),
            method: normalizedMethod,
            code: 'SESSION_EXPIRED',
          });
    }
  }

  if (!response.ok) {
    throw new ApiError(getErrorMessage(data, response), {
      status: response.status,
      data,
      url: url.toString(),
      method: normalizedMethod,
      code: 'HTTP_ERROR',
    });
  }

  return data;
}

const apiClient = {
  request: (endpoint, options = {}) => executeRequest(endpoint, options),
  get: (endpoint, options = {}) => executeRequest(endpoint, { ...options, method: 'GET' }),
  post: (endpoint, body, options = {}) => executeRequest(endpoint, { ...options, method: 'POST', body }),
  put: (endpoint, body, options = {}) => executeRequest(endpoint, { ...options, method: 'PUT', body }),
  patch: (endpoint, body, options = {}) => executeRequest(endpoint, { ...options, method: 'PATCH', body }),
  delete: (endpoint, options = {}) => executeRequest(endpoint, { ...options, method: 'DELETE' }),
  setToken: (token) => tokenStorage.set(token),
  getToken: () => tokenStorage.get(),
  clearToken: () => tokenStorage.clear(),
};

export default apiClient;
```

### Important implementation notes

This is a strong starting point, not a universal drop-in solution. Adapt
and test it against the exact backend contract.

1.  **Refresh failure and UI state:** The client throws
    `SESSION_EXPIRED`; your AuthProvider should catch it, clear the
    in-memory user, and route to login. Avoid hard-coding React
    navigation inside this low-level service.
2.  **Refresh request timeout:** Add an explicit timeout and
    cancellation policy to the refresh request too, if required by your
    app.
3.  **Retry policy:** This client retries once after a successful
    refresh. It does not automatically retry network errors or `5xx`
    responses.
4.  **Mutation retry:** Retrying an original POST/PATCH after
    authentication recovery is only safe if the backend authentication
    middleware rejects the request before performing side effects. For
    high-impact operations (orders, payments, stock changes), use
    backend idempotency keys and carefully define retry behavior.
5.  **Request body replay:** JSON and `FormData` can generally be
    rebuilt/reused as shown when the body is passed as an object. For
    streaming bodies or one-shot streams, a retry needs a body factory
    or should be disabled.
6.  **External `AbortSignal`:** A caller cancellation should cancel the
    request, not be treated as a network failure.
7.  **Credentials:** This implementation intentionally forces
    `credentials: 'include'` for a cookie-authenticated API. If you
    later support public third-party APIs, make credentials configurable
    per request.
8.  **Headers:** Use `Headers` rather than plain object key checks
    because HTTP header names are case-insensitive.
9.  **Content-Type:** Do not set JSON Content-Type when there is no
    body. Never manually set multipart Content-Type for `FormData`.
10. **URL construction:** Configure `VITE_API_URL` as the API base (for
    example `https://api.example.com/api/v1`) and use endpoint paths
    beginning with `/`.

------------------------------------------------------------------------

## 7. Session restoration in React

The API client handles HTTP transport. An AuthProvider owns the current
user's UI state.

A common startup flow:

1.  App starts with `status = "loading"` and `user = null`.
2.  AuthProvider calls `GET /auth/me`.
3.  If it succeeds, store the returned user in React state and mark
    authenticated.
4.  If it returns `401`, the API client attempts refresh and retries
    `/auth/me` once.
5.  If session recovery fails, set user to null and mark
    unauthenticated.
6.  Only then render protected routes or redirect to login.

Pseudo-code:

``` jsx
useEffect(() => {
  let active = true;

  async function restoreSession() {
    try {
      const response = await apiClient.get('/auth/me');
      if (active) {
        setUser(response.data);
        setAuthStatus('authenticated');
      }
    } catch {
      if (active) {
        setUser(null);
        setAuthStatus('unauthenticated');
      }
    }
  }

  restoreSession();

  return () => {
    active = false;
  };
}, []);
```

Adapt `response.data` to your backend response shape. In a real
implementation, also consider Strict Mode behavior in development,
cancellation, and deduplicating bootstrap calls.

### Why not call `/auth/me` for every API?

You do not need to. Call it when restoring the app session or when you
deliberately need to revalidate the current user. Normal protected
requests carry the cookie and are authenticated by backend middleware.

------------------------------------------------------------------------

## 8. Error handling contract

The API client should give callers predictable errors.

  -----------------------------------------------------------------------
  Error type              Meaning                 Typical UI response
  ----------------------- ----------------------- -----------------------
  `400`                   Invalid request         Show validation
                                                  feedback

  `401`                   Missing/invalid         Clear session / login
                          authentication after    
                          refresh attempt         

  `403`                   Authenticated but not   Show access denied
                          permitted               

  `404`                   Resource not found      Show not-found state

  `409`                   Conflict (for example   Explain conflict; allow
                          duplicate)              correction

  `422`                   Business/input          Show field/form errors
                          validation failure (if  
                          backend uses it)        

  `429`                   Rate limited            Show a wait/retry
                                                  message

  `5xx`                   Server failure          Show generic failure
                                                  and support retry

  Network error           No usable response      Show connection message

  Timeout                 Request exceeded client Offer retry if safe
                          timeout                 

  Abort                   Caller intentionally    Usually do not show an
                          cancelled               error toast
  -----------------------------------------------------------------------

The backend should document its error response format. Do not assume
every API returns the same nested property until the contract is agreed.

### UI states to design for

For each screen/API, consider:

-   Initial loading
-   Success with data
-   Success with empty data
-   Validation error
-   Authentication expired
-   Permission denied
-   Not found
-   Network disconnected
-   Timeout
-   Server error
-   User-cancelled request

Do not show raw stack traces, database messages, secrets, or internal
server details to end users.

------------------------------------------------------------------------

## 9. Timeout, cancellation, and component lifecycle

A request can take too long or become irrelevant because the user
navigated away.

Use `AbortController` (or pass a signal) for:

-   Search-as-you-type requests
-   Component unmount/navigation
-   User-cancelled uploads or actions
-   Requests with a defined timeout

For search, abort the previous query when a newer query is entered. This
prevents stale results from overwriting newer results.

A timeout is not proof that the server did not process the request. For
example, an order may have been created even if the response was lost.
Check the order state or use an idempotency strategy before
resubmitting.

------------------------------------------------------------------------

## 10. Retry rules and idempotency

Do not blindly retry every failed request.

### Generally safer to retry

-   `GET` requests, subject to rate limits and application behavior.
-   Requests explicitly designed to be idempotent.
-   A mutation protected by a backend idempotency key.

### Dangerous to retry blindly

-   Creating an order
-   Charging a payment
-   Submitting a medicine order
-   Deducting inventory
-   Any operation that could create duplicate side effects

An idempotency key is a unique client-generated identifier sent with a
mutation. The backend stores/recognizes the key and ensures the same
logical operation is not performed twice. The backend implementation is
essential; a frontend key alone provides no guarantee.

### Debounce is not duplicate protection

Debouncing reduces how often the frontend sends requests. It does not
guarantee that requests will not be duplicated due to retries, multiple
tabs, network failures, or malicious clients. Enforce data integrity and
idempotency on the backend/database where appropriate.

------------------------------------------------------------------------

## 11. Pagination, filtering, and search

Prefer server-side pagination for large datasets.

Example:

``` js
const result = await apiClient.get('/medicines', {
  query: {
    page: 1,
    limit: 20,
    search: 'paracetamol',
    sortBy: 'name',
    order: 'asc',
  },
});
```

Good practices:

-   Keep pagination/filter state in the URL when users should be able to
    bookmark/share it.
-   Debounce search input.
-   Cancel outdated search requests.
-   Reset page when filters materially change.
-   Handle empty results separately from errors.
-   Follow the backend's pagination contract (`page`, `limit`,
    `nextCursor`, etc.).
-   Do not fetch every record just to paginate in the browser when the
    dataset is large.

------------------------------------------------------------------------

## 12. Uploads and downloads

### Uploads

Pass `FormData` directly:

``` js
const formData = new FormData();
formData.append('image', file);

await apiClient.post('/stores/logo', formData);
```

Do not JSON.stringify `FormData` and do not manually set
`Content-Type: multipart/form-data`; the browser must add the boundary.

Also consider file size/type validation in the UI, upload progress
requirements, cancellation, and server-side file validation. Frontend
validation is for user experience, not security.

### Downloads

For binary downloads, a JSON-only response parser is insufficient.
Create a dedicated download method that reads `response.blob()` (or a
stream), handles errors, and triggers the download through the UI. Do
not force every response through JSON parsing.

------------------------------------------------------------------------

## 13. Security and privacy good practices

-   Never store authentication tokens in `localStorage` when using the
    HttpOnly-cookie design.
-   Never log cookies, access tokens, refresh tokens, passwords, payment
    details, or sensitive request bodies.
-   Do not put secrets in `VITE_*` variables; Vite exposes these values
    to browser code.
-   Use HTTPS in production.
-   Keep CORS allowlists explicit.
-   Apply CSRF protections appropriate to cookie auth.
-   Do not trust client-side role checks.
-   Avoid returning unnecessary personal data from APIs.
-   Sanitize/encode untrusted data when rendering; do not use unsafe
    HTML injection.
-   Validate inputs on the backend even if the frontend validates them.
-   Rate-limit login, refresh, password reset, and other sensitive
    endpoints on the backend.

------------------------------------------------------------------------

## 14. Logging and observability

In development, useful logs may include:

-   HTTP method
-   Endpoint path (avoid sensitive query parameters)
-   Status code
-   Duration
-   A request/correlation ID, if backend provides one

In production:

-   Prefer a controlled logging/monitoring service.
-   Do not log tokens, cookies, passwords, or full sensitive payloads.
-   Avoid logging every expected `401` as a server error.
-   Distinguish network errors, timeouts, cancellations, HTTP failures,
    and refresh failures.
-   Use a correlation/request ID to help connect frontend and backend
    logs.

The example client intentionally does not `console.error` every error.
The UI or monitoring layer should decide what to report.

------------------------------------------------------------------------

## 15. Performance and maintainability

-   Reuse one shared client rather than duplicating fetch setup.
-   Keep feature API functions separate from UI components.
-   Use caching/query libraries when the app needs cache invalidation,
    deduplication, background refetching, or complex server state. A
    shared API client and a query library solve different problems and
    can work together.
-   Avoid calling `/auth/me` on every API request.
-   Avoid unnecessary refresh calls.
-   Cancel stale search requests.
-   Use pagination for large lists.
-   Keep API response/error contracts consistent with backend
    documentation.
-   Do not build a giant abstraction before the app needs it.

------------------------------------------------------------------------

## 16. Testing checklist

### API client unit/integration tests

-   Builds the correct URL and query string.
-   Uses the expected method and headers.
-   Serializes JSON correctly.
-   Sends `FormData` without manually setting multipart Content-Type.
-   Handles `204 No Content`.
-   Handles JSON and non-JSON responses.
-   Converts HTTP errors into `ApiError`.
-   Handles network errors and timeouts.
-   Handles caller cancellation.
-   Sends cookies with `credentials: 'include'`.
-   On `401`, performs one refresh and retries the original request
    once.
-   Does not refresh recursively when refresh itself fails.
-   Multiple simultaneous `401`s in one tab share one refresh promise.
-   Refresh failure produces a session-expired result.
-   Does not retry arbitrary network/5xx failures automatically.
-   Does not leak sensitive information in logs.

### Browser/backend integration tests

-   Login sets the expected cookies.
-   Cookies have correct `HttpOnly`, `Secure`, `SameSite`, `Path`, and
    expiry attributes.
-   Credentialed CORS works for the deployed frontend origin.
-   `/auth/me` restores the user after a page refresh.
-   Expired access token + valid refresh token works.
-   Expired/revoked refresh token requires login.
-   Logout clears/revokes the session.
-   Two tabs refreshing concurrently behave safely.
-   CSRF protections work for state-changing requests.
-   Protected APIs enforce authorization on the backend.

------------------------------------------------------------------------

## 17. Common interview / problem-solving questions and answers

### Q1. Why create a shared API client?

**Answer:** To centralize URL construction, headers, serialization,
cookie handling, parsing, errors, timeout, and authentication recovery.
This avoids repeating transport logic in every component.

### Q2. Why should duplicate medicine checks not be frontend-only?

**Answer:** Frontend checks improve UX, but two clients can submit
simultaneously or bypass the UI. Backend business validation and
database constraints must protect data integrity.

### Q3. Does React state authenticate the user?

**Answer:** No. React state controls what the UI displays. The backend
validates the cookie/token and checks permissions on protected requests.

### Q4. Why call `/auth/me` after a page refresh?

**Answer:** React memory state is reset on a full page refresh.
`/auth/me` asks the backend to identify the current session using the
browser's cookie and returns safe user information.

### Q5. Should `/auth/me` run on every API request?

**Answer:** No. It is normally used during session restoration or
deliberate revalidation. Protected API requests independently
authenticate through backend middleware.

### Q6. If the token is in an HttpOnly cookie, how does the browser send it?

**Answer:** The browser attaches eligible cookies automatically when
fetch uses `credentials: 'include'`. JavaScript cannot read an HttpOnly
cookie.

### Q7. Why use `credentials: 'include'`?

**Answer:** It tells fetch to include credentials such as cookies on
cross-origin requests, subject to cookie and CORS rules. It does not
itself create or validate a cookie.

### Q8. What happens when the access token expires?

**Answer:** Backend returns `401`. Client calls the refresh endpoint. If
refresh succeeds, client retries the original request once. If refresh
fails, the app clears user state and requires login.

### Q9. Why use both proactive and reactive refresh?

**Answer:** Proactive refresh can avoid an expired-token interruption
during normal use. Reactive refresh is the fallback for suspended tabs,
browser sleep, clock/network delays, and missed timers.

### Q10. Will a JavaScript timer refresh a token while the browser is closed?

**Answer:** No. Timers do not run after the page/browser is closed. On
the next app load, restore the session with `/auth/me` and the refresh
flow.

### Q11. Should the refresh token be returned in JSON?

**Answer:** Not in this HttpOnly-cookie design. Keep it in a properly
configured HttpOnly cookie and do not expose it to JavaScript.

### Q12. What if five requests return 401 at the same time?

**Answer:** In one tab, use a single-flight shared refresh promise. All
five wait for one refresh, then each original request is retried once.

### Q13. Does one shared promise coordinate five browser tabs?

**Answer:** No. Each tab has its own JavaScript context. Use cross-tab
coordination such as BroadcastChannel/Web Locks where appropriate, and
ensure backend refresh rotation safely handles concurrency.

### Q14. What if the refresh token is expired or revoked?

**Answer:** Refresh fails. Clear in-memory user state and require the
user to log in again.

### Q15. Why not retry every failed request?

**Answer:** A network failure does not prove the backend did not process
the operation. Retrying a create/payment/order request can cause
duplicate side effects. Retry only safe/idempotent operations or use
backend idempotency keys.

### Q16. Why should `FormData` not be JSON.stringify-ed?

**Answer:** It is a multipart payload. The browser must encode it and
set the multipart boundary in Content-Type.

### Q17. Why not set Content-Type on every request?

**Answer:** GET/DELETE requests often have no body. Setting Content-Type
unnecessarily can cause needless preflight behavior in cross-origin
cases. Set it when sending JSON.

### Q18. Why parse based on response Content-Type?

**Answer:** Not every response is JSON. An endpoint may return no
content, plain text, HTML from a proxy error, or binary data. Parsing
must match the response type.

### Q19. Why use AbortController?

**Answer:** To cancel stale or unnecessary requests and enforce
client-side timeouts. Cancellation does not guarantee that the backend
did not already process the request.

### Q20. Is a 403 the same as a 401?

**Answer:** No. `401` means authentication is missing or invalid. `403`
means the user is authenticated but not allowed to perform the action.
Exact API semantics should be documented.

### Q21. Is CORS a replacement for CSRF protection?

**Answer:** No. CORS controls cross-origin JavaScript access to
responses. Cookie-based state-changing requests still need an
appropriate CSRF defense.

### Q22. Does debouncing prevent duplicate database records?

**Answer:** No. Debouncing reduces frontend request frequency. Backend
validation, idempotency, and database constraints provide actual
integrity protection.

### Q23. Should the API client redirect to login itself?

**Answer:** Prefer not to tightly couple a shared transport layer to
React routing. Let it throw a clear session-expired error; AuthProvider
or the app-level auth handler updates state and navigates.

### Q24. Why is a timeout not always safe to retry?

**Answer:** The server may have completed the operation but the response
may have been lost. Check operation status or use idempotency keys
before repeating a mutation.

------------------------------------------------------------------------

## 18. Practical decision checklist before integrating any API

Before writing the integration, ask:

1.  What does this endpoint do?
2.  What HTTP method and URL does the backend contract specify?
3.  What request fields and content type are required?
4.  Does it need authentication cookies?
5.  What response shape will success return?
6.  What errors can it return, and which are field-level?
7.  What should the UI show while loading?
8.  What should the UI show for an empty result?
9.  Can the request be cancelled?
10. Is pagination/filtering needed?
11. Could a retry duplicate a side effect?
12. Does it need an idempotency key?
13. What happens if the user navigates away mid-request?
14. Does the endpoint return JSON, text, or a file?
15. How will the behavior be tested?

------------------------------------------------------------------------

## 19. Recommended implementation order for your MDP project

Do not implement every advanced feature at once. Build and verify in
layers:

1.  Agree the API contract and response/error shape with the backend.
2.  Implement base URL, methods, headers, JSON/FormData handling, and
    response parsing.
3.  Configure cookie credentials and verify login in the browser's
    Network/Application panels.
4.  Implement `/auth/me` and AuthProvider session restoration.
5.  Implement refresh endpoint contract and backend cookie behavior.
6.  Add one-time 401 refresh + original request retry.
7.  Add same-tab single-flight handling.
8.  Test logout, expired access token, expired refresh token, and
    concurrent requests.
9.  Add timeout and cancellation.
10. Add cross-tab coordination only after backend refresh rotation
    behavior is defined.
11. Add safe logging, monitoring, and automated tests.
12. Add proactive refresh only if it provides a real UX benefit; keep
    reactive recovery regardless.

**Do not call the code production-ready just because it has many
features.** It becomes production-ready only after it matches the real
backend contract, is security-reviewed, and passes integration tests in
the actual deployment environment.
