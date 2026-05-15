# HTTP Status Codes

HTTP status codes are grouped into five classes. The first digit indicates the class.

| Range | Class |
|-------|-------|
| 1xx | Informational |
| 2xx | Success |
| 3xx | Redirection |
| 4xx | Client Error |
| 5xx | Server Error |

---

## 1XX — Informational

### 100 Continue

The server has received the request headers and is telling the client it is safe to send the request body. Used when the client sends an `Expect: 100-continue` header before uploading a large body — it checks the server is ready before sending the data.

**Trigger:**
```bash
curl -X POST http://<LB-IP>/api/transaction \
  -H "Content-Type: application/json" \
  -H "Expect: 100-continue" \
  -d '{"amount": 500, "description": "Lunch", "category": "Food"}'
```

> **Note:** Use `curl -v` to see the `100 Continue` response in the exchange before the final `201`.

---

## 2XX — Success

### 200 OK

The request succeeded. The most common response for GET and DELETE (all) requests.

**Trigger:**
```bash
curl -X GET http://<LB-IP>/api/transaction
```

---

### 201 Created

A new resource was successfully created. Returned after a successful POST.

**Trigger:**
```bash
curl -X POST http://<LB-IP>/api/transaction \
  -H "Content-Type: application/json" \
  -d '{"amount": 500, "description": "Lunch", "category": "Food"}'
```

---

### 204 No Content

The request succeeded but there is no response body. Returned after deleting a single transaction.

**Trigger:**
```bash
curl -X DELETE http://<LB-IP>/api/transaction/1
```

---

## 3XX — Redirection

### 304 Not Modified

The resource has not changed since the last request. The browser uses its cached copy — no body is sent. Triggered when the client sends a matching `If-None-Match` header.

**Trigger:**

1. Make a GET request and note the `ETag` value in the response headers:
```bash
curl -I http://<LB-IP>/api/transaction
```

2. Repeat the request with the `If-None-Match` header:
```bash
curl -I http://<LB-IP>/api/transaction \
  -H 'If-None-Match: W/"<etag-value>"'
```

> **Note:** The data must not change between the two requests for the ETag to match.

---

## 4XX — Client Error

### 400 Bad Request

The request was malformed or failed validation. Returned when required fields are missing or invalid.

**Trigger — missing field:**
```bash
curl -X POST http://<LB-IP>/api/transaction \
  -H "Content-Type: application/json" \
  -d '{"amount": -10, "description": "", "category": "Invalid"}'
```

---

### 401 Unauthorized

The request requires authentication. The client did not provide credentials or provided invalid ones.

> **Note:** Despite the name, 401 means *unauthenticated* — the client's identity is not known. It is called "Unauthorized" for historical reasons.


---

### 403 Forbidden

The server understood the request but refuses to authorise it. Unlike 401, the client's identity may be known — they simply do not have permission.


Then:
```bash
curl http://<LB-IP>/secret
```

> **401 vs 403:** 401 means *who are you?* — send credentials. 403 means *I know who you are, but you can't come in.*

---

### 404 Not Found

The requested resource does not exist.

**Trigger — non-existent transaction:**
```bash
curl http://<LB-IP>/api/transaction/9999
```

---

### 405 Method Not Allowed

The HTTP method used is not supported for this endpoint. The response includes an `Allow` header listing the accepted methods.

**Trigger — PUT on /transaction:**
```bash
curl -X PUT http://<LB-IP>/api/transaction \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

## 5XX — Server Error

### 500 Internal Server Error

An unexpected error occurred on the server, typically a database failure.

**Trigger:**

Stop MySQL on the DB EC2, then make any transaction request:
```bash
# On DB EC2
systemctl stop mysqld
```
```bash
curl http://<LB-IP>/api/transaction
```

Restore:
```bash
systemctl start mysqld
```

---

### 502 Bad Gateway

The load balancer or reverse proxy could not reach the upstream server.

**Trigger from LB (frontend is down):**

Stop nginx on the frontend EC2, then reload the page in the browser:
```bash
# On Frontend EC2
systemctl stop nginx
```

**Trigger from Frontend (backend is down):**

Stop the backend service on the backend EC2, then make an API request:
```bash
# On Backend EC2
systemctl stop backend
```

Restore:
```bash
systemctl start nginx     # on Frontend EC2
systemctl start backend   # on Backend EC2
```

---

### 503 Service Unavailable

The server is temporarily unable to handle requests — intentional maintenance or overload.

**Trigger from LB:**

Uncomment the `return 503` line in `/etc/nginx/nginx.conf` on the LB EC2 and reload:
```bash
# return 503 "Service temporarily unavailable";   ← uncomment this
systemctl reload nginx
```

Restore:
```bash
# comment out the return 503 line, then:
systemctl reload nginx
```

---

### 504 Gateway Timeout

The load balancer gave up waiting for the upstream to respond.

**Trigger:**

Open this URL in the browser (backend takes 40 s, LB times out at 10 s):
```
http://<LB-IP>/api/slow
```

---

## Quick Reference

| Code | Name | Who sends it | How to trigger |
|------|------|-------------|----------------|
| 100 | Continue | Backend | `POST` with `Expect: 100-continue` header |
| 200 | OK | Backend | `GET /api/transaction` |
| 201 | Created | Backend | `POST /api/transaction` with valid body |
| 204 | No Content | Backend | `DELETE /api/transaction/:id` |
| 304 | Not Modified | Backend | Repeat `GET` with matching `If-None-Match` header |
| 400 | Bad Request | Backend | `POST` with invalid or missing fields |
| 401 | Unauthorized | Frontend nginx | Access page without credentials (nginx `auth_basic`) |
| 403 | Forbidden | Frontend nginx | Access a denied path (nginx `deny all`) |
| 404 | Not Found | Backend | `GET /api/transaction/9999` |
| 405 | Method Not Allowed | Backend | `PUT /api/transaction` |
| 500 | Internal Server Error | Backend | Stop MySQL, then make any API request |
| 502 | Bad Gateway | LB / Frontend nginx | Stop frontend or backend nginx/service |
| 503 | Service Unavailable | LB / Frontend nginx | Uncomment `return 503` in nginx config |
| 504 | Gateway Timeout | LB nginx | Open `/api/slow` (backend waits 40 s, LB times out at 10 s) |
