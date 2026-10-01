# Expense Backend API

The Expense backend is a Node.js (Express) REST API. It stores expenses in MySQL and lets you create, read, update and delete them. A few extra endpoints exist only to demonstrate HTTP status codes.

---

## Base URL

The same API can be reached in two ways:

| From | Base URL | Example |
|------|----------|---------|
| Browser / HTTPie / Postman (through frontend nginx) | `http://<FRONTEND-PUBLIC-IP>/api` | `http://<FRONTEND-PUBLIC-IP>/api/transaction` |
| Backend server itself (direct to Node.js) | `http://localhost:8080` | `http://localhost:8080/transaction` |

Frontend nginx strips the `/api` prefix before passing the request to the backend, so `/api/transaction` and `:8080/transaction` hit the same code. All URLs below use the frontend form. Once a load balancer is added in front, swap `<FRONTEND-PUBLIC-IP>` for the LB IP; everything else stays the same.

---

## All Endpoints

| # | Method | URL | What it does | Success |
|---|--------|-----|--------------|---------|
| 1 | GET | `/api/health` | Is the backend up, and can it reach MySQL? | 200 |
| 2 | GET | `/api/transaction` | List all expenses | 200 |
| 3 | GET | `/api/transaction/{id}` | Get one expense | 200 |
| 4 | POST | `/api/transaction` | Add an expense | 201 |
| 5 | PUT | `/api/transaction/{id}` | Update an expense | 200 |
| 6 | DELETE | `/api/transaction/{id}` | Delete one expense | 204 |
| 7 | DELETE | `/api/transaction` | Delete **all** expenses (admin token) | 200 |
| 8 | GET | `/api/transactions` | Old URL, redirects to `/api/transaction` | 301 |
| 9 | POST | `/api/transactions` | Old URL, redirects and keeps the POST | 308 |
| 10 | GET | `/api/latest` | Redirects to the newest expense | 302 |
| 11 | GET | `/api/error` | Demo: crashes on purpose | 500 |
| 12 | GET | `/api/maintenance` | Demo: pretends to be under maintenance | 503 |
| 13 | GET | `/api/slow?seconds=N` | Demo: answers after N seconds | 200 / 504 |

These URLs have no `/api`, so they are answered by **frontend nginx**, not the backend. They are covered in [Web Server (nginx) URLs](#web-server-nginx-urls):

| # | Method | URL | What it does | Status |
|---|--------|-----|--------------|--------|
| 14 | GET | `/home` | Old page, moved to `/` for good | 301 |
| 15 | GET | `/docs` | Temporary redirect to an external page | 302 |
| 16 | GET | `/static/css/style.css` (any static file) | Re-check a file you already have | 200 / 304 |

---

## The Expense Object

```json
{
  "id": 7,
  "amount": 750,
  "description": "Electricity bill",
  "category": "Utilities"
}
```

| Field | Type | Rules |
|-------|------|-------|
| `id` | integer | Created by MySQL. You never send it in the body |
| `amount` | integer | Required. Whole number, greater than 0, at most 1,000,000 |
| `description` | string | Required. Not blank, at most 255 characters |
| `category` | string | Required. Exactly one of `Food`, `Travel`, `Entertainment`, `Shopping`, `Health`, `Utilities`, `Other` (case-sensitive) |

---

## Errors Any Endpoint Can Return

These are not repeated under every endpoint. Each endpoint below lists only its own extra errors.

| Status | When it happens | Response body |
|--------|-----------------|---------------|
| **400** Bad Request | The body is not valid JSON, e.g. `{"amount": 100,` | `{ "message": "malformed JSON body" }` |
| **404** Not Found | The URL does not exist, e.g. `GET /api/xyz` | `{ "message": "route GET /xyz not found" }` |
| **405** Method Not Allowed | The URL exists but not with that method, e.g. `PATCH /api/transaction`. The `Allow` header lists the methods that work | `{ "message": "method not allowed" }` |
| **413** Content Too Large | The body is bigger than 10 KB | `{ "message": "request body too large (limit 10kb)" }` |
| **415** Unsupported Media Type | A POST or PUT without the `Content-Type: application/json` header | `{ "message": "Content-Type must be application/json" }` |
| **429** Too Many Requests | More than 10 write requests (POST, PUT, DELETE) per minute from the same IP. The `Retry-After` header says how many seconds to wait | `{ "message": "too many requests, try again in 42s" }` |
| **500** Internal Server Error | MySQL is reachable but the query fails (table missing, no permission, wrong password) | `{ "message": "could not retrieve transactions", "error": "Table 'transactions.transactions' doesn't exist" }` |
| **502** Bad Gateway | The backend is stopped. Comes from nginx as an HTML page, not JSON | HTML error page |
| **503** Service Unavailable | MySQL cannot be reached (stopped, wrong IP, port 3306 blocked). Header `Retry-After: 30` | `{ "message": "database unavailable", "error": "connect ECONNREFUSED 10.0.1.5:3306" }` |

Write requests to `/api/transaction` (POST, PUT, DELETE) also return these headers, so you can watch the limit count down:

```
RateLimit-Limit: 10
RateLimit-Remaining: 7
```

---

## 1. Health Check

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/health` |
| **Auth** | None |
| **Input** | None |

Checks that the backend is running **and** can run a query on MySQL.

**Success: `200 OK`**

```json
{ "status": "ok", "db": "up" }
```

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 503 | MySQL is not reachable, or the query fails | `{ "status": "degraded", "db": "down", "error": "connect ECONNREFUSED 10.0.1.5:3306" }` |

---

## 2. List All Expenses

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transaction` |
| **Auth** | None |
| **Input** | None |

Returns every expense, newest first.

**Success: `200 OK`**

```json
{
  "result": [
    { "id": 3, "amount": 500,  "description": "Groceries",     "category": "Food" },
    { "id": 2, "amount": 1200, "description": "Flight ticket", "category": "Travel" }
  ]
}
```

With no expenses, `result` is an empty list: `{ "result": [] }`.

**Also possible: `304 Not Modified`**

The response has an `ETag` header, a fingerprint of the data:

```
ETag: W/"87-7W1JHFtg/3IKNdaGSNH/ZUpH0Lw"
```

Send the request again with these two headers:

```
If-None-Match: W/"87-7W1JHFtg/3IKNdaGSNH/ZUpH0Lw"
Cache-Control: max-age=0
```

If nothing changed, the answer is `304` with an **empty body**, meaning "you already have the latest copy". `Cache-Control: max-age=0` is needed because some tools add `Cache-Control: no-cache` on their own, which forces a full `200` every time.

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 500 | Query failed | `{ "message": "could not retrieve transactions", "error": "..." }` |
| 503 | MySQL not reachable | `{ "message": "database unavailable", "error": "..." }` |

---

## 3. Get One Expense

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transaction/{id}` |
| **Example** | `http://<FRONTEND-PUBLIC-IP>/api/transaction/7` |
| **Auth** | None |
| **Input** | `id` in the URL |

**Success: `200 OK`**

```json
{ "id": 7, "amount": 750, "description": "Electricity bill", "category": "Utilities" }
```

`304 Not Modified` also works here, the same way as in [List All Expenses](#2-list-all-expenses).

**Errors**

| Status | When | Example | Response body |
|--------|------|---------|---------------|
| 400 | `id` is not a positive whole number | `/api/transaction/abc`, `/api/transaction/0` | `{ "message": "invalid id" }` |
| 404 | No expense with that `id` | `/api/transaction/999999` | `{ "message": "transaction 999999 not found" }` |
| 500 | Query failed | | `{ "message": "could not retrieve transaction", "error": "..." }` |
| 503 | MySQL not reachable | | `{ "message": "database unavailable", "error": "..." }` |

---

## 4. Add an Expense

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transaction` |
| **Auth** | None |
| **Headers** | `Content-Type: application/json` |

**Sample input**

```json
{
  "amount": 750,
  "description": "Electricity bill",
  "category": "Utilities"
}
```

**Success: `201 Created`**

```json
{ "message": "transaction added successfully", "id": 7 }
```

Response header pointing at the new expense:

```
Location: transaction/7
```

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 400 | Body is broken JSON | `{ "message": "malformed JSON body" }` |
| 413 | Body larger than 10 KB | `{ "message": "request body too large (limit 10kb)" }` |
| 415 | `Content-Type` is missing or not JSON | `{ "message": "Content-Type must be application/json" }` |
| 422 | JSON is fine but the values break the rules | see below |
| 429 | Too many write requests | `{ "message": "too many requests, try again in 42s" }` |
| 500 | Insert failed | `{ "message": "could not add transaction", "error": "..." }` |
| 503 | MySQL not reachable | `{ "message": "database unavailable", "error": "..." }` |

**422 example.** Input:

```json
{ "amount": -5, "description": "", "category": "Crypto" }
```

Response:

```json
{
  "message": "validation failed",
  "errors": [
    "amount must be a positive integer",
    "description is required",
    "category must be one of: Food, Travel, Entertainment, Shopping, Health, Utilities, Other"
  ]
}
```

All the validation messages you can get:

| Field | Message |
|-------|---------|
| amount | `amount is required` |
| amount | `amount must be a positive integer` |
| amount | `amount cannot exceed 1,000,000` |
| description | `description is required` |
| description | `description cannot exceed 255 characters` |
| category | `category is required` |
| category | `category must be one of: Food, Travel, Entertainment, Shopping, Health, Utilities, Other` |

**400 vs 422:** with `400` the server cannot read the body at all. With `422` it can read it, but the values are not acceptable.

---

## 5. Update an Expense

| | |
|---|---|
| **Method** | `PUT` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transaction/{id}` |
| **Example** | `http://<FRONTEND-PUBLIC-IP>/api/transaction/7` |
| **Auth** | None |
| **Headers** | `Content-Type: application/json` |

`PUT` replaces the whole expense, so send **all three fields**, even the ones that did not change.

**Sample input**

```json
{
  "amount": 800,
  "description": "Electricity bill (revised)",
  "category": "Utilities"
}
```

**Success: `200 OK`**

```json
{
  "message": "transaction updated successfully",
  "transaction": { "id": 7, "amount": 800, "description": "Electricity bill (revised)", "category": "Utilities" }
}
```

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 400 | `id` is not a positive whole number | `{ "message": "invalid id" }` |
| 400 | Body is broken JSON | `{ "message": "malformed JSON body" }` |
| 404 | No expense with that `id` | `{ "message": "transaction 999999 not found" }` |
| 413 | Body larger than 10 KB | `{ "message": "request body too large (limit 10kb)" }` |
| 415 | `Content-Type` is missing or not JSON | `{ "message": "Content-Type must be application/json" }` |
| 422 | A field is missing or breaks the rules (same messages as [Add an Expense](#4-add-an-expense)) | `{ "message": "validation failed", "errors": [ ... ] }` |
| 429 | Too many write requests | `{ "message": "too many requests, try again in 42s" }` |
| 500 | Update failed | `{ "message": "could not update transaction", "error": "..." }` |
| 503 | MySQL not reachable | `{ "message": "database unavailable", "error": "..." }` |

---

## 6. Delete One Expense

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transaction/{id}` |
| **Example** | `http://<FRONTEND-PUBLIC-IP>/api/transaction/7` |
| **Auth** | None, no token needed |
| **Input** | `id` in the URL. No body |

**Success: `204 No Content`**

The response has **no body at all**. The status code alone says it worked.

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 400 | `id` is not a positive whole number | `{ "message": "invalid id" }` |
| 404 | No expense with that `id` (or it was already deleted) | `{ "message": "transaction 7 not found" }` |
| 429 | Too many write requests | `{ "message": "too many requests, try again in 42s" }` |
| 500 | Delete failed | `{ "message": "could not delete transaction", "error": "..." }` |
| 503 | MySQL not reachable | `{ "message": "database unavailable", "error": "..." }` |

---

## 7. Delete All Expenses

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transaction` |
| **Auth** | **Admin token required** |
| **Headers** | `Authorization: Bearer admin123` |
| **Input** | No body |

This is the only endpoint that needs a token, because it wipes every row. The token is whatever `ADMIN_TOKEN` is set to in `backend.service` (default `admin123`). In HTTPie or Postman you can use the **Auth → Bearer** option instead of typing the header.

**Success: `200 OK`**

```json
{ "message": "all transactions deleted", "deleted": 5 }
```

`deleted` is how many rows were removed.

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 401 | No `Authorization` header sent. Response also has the header `WWW-Authenticate: Bearer realm="expense"` | `{ "message": "admin token required" }` |
| 403 | Token sent, but it is wrong | `{ "message": "invalid admin token" }` |
| 429 | Too many write requests | `{ "message": "too many requests, try again in 42s" }` |
| 500 | Delete failed | `{ "message": "could not delete transactions", "error": "..." }` |
| 503 | MySQL not reachable | `{ "message": "database unavailable", "error": "..." }` |

**401 vs 403:** `401` means "who are you?", no credentials were sent. `403` means "I know who you are, and you are not allowed", the credentials were wrong.

---

## 8. Old URL: GET `/transactions`

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transactions` (note the **s**) |
| **Auth** | None |
| **Input** | None |

Imagine the API used to live at `/transactions` and was renamed. Old links still work: the server answers with a redirect.

**Success: `301 Moved Permanently`**

```
Location: transaction
```

Body (plain text): `Moved Permanently. Redirecting to transaction`

A client that follows redirects (browsers do this by default) then requests `/api/transaction` and gets the expense list. To see the `301` itself, switch off "follow redirects" in your HTTP tool.

---

## 9. Old URL: POST `/transactions`

| | |
|---|---|
| **Method** | `POST` (any method other than GET works the same) |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/transactions` |
| **Auth** | None |
| **Headers** | `Content-Type: application/json` |

**Sample input**

```json
{ "amount": 10, "description": "via 308", "category": "Other" }
```

**Success: `308 Permanent Redirect`**

```
Location: transaction
```

Body (plain text): `Permanent Redirect. Redirecting to transaction`

With "follow redirects" on, the client sends the **same POST with the same body** to `/api/transaction`, and you end up with `201 Created`.

**301 vs 308:** after a `301`, clients may switch a POST to a GET and drop the body. A `308` forbids that, so the method and body are always kept.

---

## 10. Newest Expense

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/latest` |
| **Auth** | None |
| **Input** | None |

**Success: `302 Found`**

```
Location: transaction/7
```

Body (plain text): `Found. Redirecting to transaction/7`

Following the redirect gives you expense #7.

**301 vs 302:** a `301` says "this moved forever, update your bookmarks". A `302` says "go there for now". This endpoint is a `302` because the newest expense changes every time one is added.

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 404 | There are no expenses yet | `{ "message": "no transactions yet" }` |
| 500 | Query failed | `{ "message": "could not retrieve latest transaction", "error": "..." }` |
| 503 | MySQL not reachable | `{ "message": "database unavailable", "error": "..." }` |

---

## 11. Demo: Server Error

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/error` |
| **Auth** | None |
| **Input** | None |

The code throws an exception on purpose. This is what a bug in the code looks like from the outside.

**Response: `500 Internal Server Error`**

```json
{ "message": "internal server error", "error": "simulated application failure" }
```

---

## 12. Demo: Maintenance

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/maintenance` |
| **Auth** | None |
| **Input** | None |

**Response: `503 Service Unavailable`**

```
Retry-After: 60
```

```json
{ "message": "service temporarily unavailable" }
```

`Retry-After` tells the client to try again in 60 seconds.

---

## 13. Demo: Slow Response

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/api/slow?seconds=N` |
| **Example** | `http://<FRONTEND-PUBLIC-IP>/api/slow?seconds=5` |
| **Auth** | None |
| **Input** | `seconds` in the query string. Optional, default `40`, maximum `120` |

The backend waits `N` seconds before answering. nginx only waits so long, so the result depends on `N` and on which way you call it:

| Called through | nginx waits up to | `seconds=5` | `seconds=40` |
|----------------|-------------------|-------------|--------------|
| Frontend: `http://<FRONTEND-PUBLIC-IP>/api/slow` | 30s (`proxy_read_timeout` in `expense.conf`) | `200` after 5s | `504` after 30s |
| Backend only: `http://localhost:8080/slow` (run on the backend server) | no limit | `200` after 5s | `200` after 40s |

**Success: `200 OK`**

```json
{ "message": "finally responded after 5s" }
```

**Errors**

| Status | When | Response body |
|--------|------|---------------|
| 504 Gateway Timeout | nginx gave up waiting. Comes from nginx, not the backend | HTML error page |

**502 vs 504:** `502` means the backend did not answer at all (stopped or crashed). `504` means it accepted the request but took too long to reply.

---

## Web Server (nginx) URLs

These URLs have no `/api` in them, so the request never reaches Node.js. Frontend nginx answers by itself, using rules in `/etc/nginx/default.d/expense.conf`:

```
Client ──► Frontend nginx ──✋ answers here (rule matched)
                                          Backend is never called
```

**How to tell nginx answered:** the body is a small HTML page ending in `nginx/1.20.1`, not JSON. The response headers also include `Server: nginx/1.20.1`.

**To see the 3XX itself:** switch off "follow redirects" in your HTTP tool. Otherwise the tool quietly follows the `Location` header and shows only the final `200`.

---

## 14. Old Page: `/home`

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/home` |
| **Answered by** | Frontend nginx |
| **Auth** | None |
| **Input** | None |

nginx rule:

```nginx
location = /home {
    return 301 /;
}
```

**Response: `301 Moved Permanently`**

```
HTTP/1.1 301 Moved Permanently
Server: nginx/1.20.1
Content-Type: text/html
Location: http://<FRONTEND-PUBLIC-IP>/
```

Body (HTML, shown only by very old clients that don't follow redirects):

```html
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.20.1</center>
</body>
</html>
```

**What happens next:** with "follow redirects" on, the client requests `/` and gets the Expense Tracker page with `200 OK`. That is two requests, but you only see the last one. Browsers also **remember** a 301, so next time they go straight to `/` without asking `/home` first.

**Compare:** `GET /api/transactions` ([#8](#8-old-url-get-transactions)) is also a 301, but it comes from Node.js. Its body is plain text (`Moved Permanently. Redirecting to transaction`) instead of an nginx HTML page. Same code, different layer.

---

## 15. Temporary Redirect: `/docs`

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/docs` |
| **Answered by** | Frontend nginx |
| **Auth** | None |
| **Input** | None |

nginx rule:

```nginx
location = /docs {
    return 302 https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status;
}
```

**Response: `302 Found`** (older nginx versions show it as `302 Moved Temporarily`)

```
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.20.1
Content-Type: text/html
Location: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
```

Body: the same kind of small nginx HTML page, titled `302 Found`.

**What happens next:** with "follow redirects" on, the client leaves your server and opens the MDN page on HTTP status codes. A redirect can point to **any** site, not just your own.

**301 vs 302:** browsers cache a `301` and skip the old URL next time. A `302` is never cached; the browser asks `/docs` again every time. Use `302` when the target might change later.

---

## 16. Static File Re-check

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://<FRONTEND-PUBLIC-IP>/static/css/style.css` (works for any file: `/static/js/app.js`, `/index.html`, …) |
| **Answered by** | Frontend nginx |
| **Auth** | None |

**First request: `200 OK`**

The full file comes back, with headers describing the version you got:

```
HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/css
Last-Modified: Wed, 30 Sep 2026 10:15:42 GMT
ETag: "66fa7b2e-2c9a"
```

**Second request: `304 Not Modified`**

Send the request again and add **one** of these headers, copying the value from the first response:

| Header to send | Value copied from |
|----------------|-------------------|
| `If-Modified-Since: Wed, 30 Sep 2026 10:15:42 GMT` | `Last-Modified` |
| `If-None-Match: "66fa7b2e-2c9a"` | `ETag` |

Response:

```
HTTP/1.1 304 Not Modified
Server: nginx/1.20.1
Last-Modified: Wed, 30 Sep 2026 10:15:42 GMT
ETag: "66fa7b2e-2c9a"
```

**No body.** nginx is saying "the file hasn't changed since the version you have, so use your copy". This saves sending the whole file again.

**Why this matters:** browsers do this automatically. Reload the Expense Tracker with DevTools → Network open and you'll see `style.css` and `app.js` come back as `304`. Your values for `Last-Modified` and `ETag` will be different from the examples above; always copy them from your own first response.

**Compare:** `GET /api/transaction` ([#2](#2-list-all-expenses)) also gives `304`, but there Node.js builds the `ETag` from the JSON data, not nginx from the file.

---

## All 3XX Codes in One Place

| Code | Name | URL | Answered by | Cached by browser? | Keeps POST + body? |
|------|------|-----|-------------|--------------------|---------------------|
| 301 | Moved Permanently | `/home` | nginx | Yes | No |
| 301 | Moved Permanently | `/api/transactions` (GET) | Node.js | Yes | No |
| 302 | Found | `/docs` | nginx | No | No |
| 302 | Found | `/api/latest` | Node.js | No | No |
| 304 | Not Modified | `/static/...` | nginx | n/a, means "use your cached copy" | n/a |
| 304 | Not Modified | `/api/transaction`, `/api/transaction/{id}` | Node.js | n/a, means "use your cached copy" | n/a |
| 308 | Permanent Redirect | `/api/transactions` (POST) | Node.js | Yes | **Yes** |

---

## Settings

The backend reads these from `Environment=` lines in `/etc/systemd/system/backend.service`.

| Variable | Default | Purpose |
|----------|---------|---------|
| `APP_PORT` | `8080` | Port the backend listens on |
| `DB_HOST` | *(empty)* | MySQL server IP |
| `DB_USER` | `expense` | MySQL user |
| `DB_PWD` | `ExpenseApp@1` | MySQL password |
| `DB_DATABASE` | `transactions` | MySQL database |
| `ADMIN_TOKEN` | `admin123` | Token for [Delete All Expenses](#7-delete-all-expenses) |
| `RATE_LIMIT_MAX` | `10` | Write requests allowed per IP per window |
| `RATE_LIMIT_WINDOW_SEC` | `60` | Length of the rate-limit window, in seconds |

After changing any of them:

```bash
systemctl daemon-reload
systemctl restart backend
```

---

## Database Table

```sql
CREATE TABLE transactions (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    amount      INT NOT NULL,
    description VARCHAR(255) NOT NULL,
    category    VARCHAR(50) NOT NULL
);
```

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| express | ^4.18.2 | Web framework |
| body-parser | ^1.20.2 | Reads JSON request bodies |
| cors | ^2.8.5 | Allows calls from other origins |
| mysql2 | ^3.6.0 | MySQL driver with connection pooling |

Runs on Node.js 24.
