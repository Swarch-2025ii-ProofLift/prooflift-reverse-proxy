# Inverse Proxy

## 1. Service Structure

```
prooflift-reverse_proxy/
├── Dockerfile             # Container 
├── nginx.conf             # Nginx configuration with security features
└── README.md              # This documentation
```

## 2. Configuration Overview

The nginx.conf file defines a secure and optimized reverse proxy for both the web frontend (React SSR) and mobile frontend (React Native) that links to the gateway API (Kong).

It includes global security headers, rate limiting, and error handling.

## 3. Key Configuration Sections

### Worker and Event Settings

```
user nginx;
worker_processes auto;

events {
    worker_connections 1024;
}
```

#### Explanation:

- Uses the default nginx user for process isolation.

- Automatically determines the number of worker processes based on available CPU cores.

- Allows up to 1024 simultaneous connections per worker.

#### Benefits:

- Efficient use of system resources.

- Scalable concurrency handling for incoming requests.

### HTTP Configuration

```
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

```

#### Explanation:

- **include /etc/nginx/mime.types:** Loads MIME type definitions so that NGINX serves files with the correct content types.  
- **default_type application/octet-stream:** Sets a fallback MIME type for unknown file extensions.  
- **sendfile on:** Enables efficient file transfer using the kernel-level `sendfile()` system call.  
- **keepalive_timeout 65:** Keeps idle connections open for 65 seconds, improving performance for clients making repeated requests.

#### Benefits:

- Reduces latency for repeated connections.

- Improves I/O performance for static assets.

### Rate Limiting

```
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
```

#### Explanation:

- Creates a **shared memory zone** called `api` (10 MB) to track request rates per client IP.  
- Limits each IP to **30 requests per second**, helping mitigate abusive or automated traffic.

#### Benefits:

- Prevents flooding from malicious users.  
- Helps protect backend services from overload.  
- Provides a fair usage policy for all clients.

---

### Server Block (Main Reverse Proxy)

```
server {
listen 80;
server_name localhost;
```
#### Explanation:

- **listen 80:** Listens for incoming HTTP traffic on port 80.  
- **server_name localhost:** Handles requests sent to the local development hostname.

#### Benefits:

- Establishes a central entry point for frontend and API routes.  
- Simplifies testing and local deployment setup.

---

### Security Headers and Server Hardening

```
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
proxy_hide_header X-Powered-By;
server_tokens off;
```

#### Explanation:

- **X-Frame-Options:** Prevents clickjacking by allowing framing only from the same origin.  
- **X-Content-Type-Options:** Disables MIME-type sniffing to prevent malicious interpretation of files.  
- **X-XSS-Protection:** Activates built-in XSS filtering in browsers.  
- **proxy_hide_header X-Powered-By:** Removes backend framework information.  
- **server_tokens off:** Hides NGINX version details from responses.

#### Benefits:

- Strengthens HTTP response security.  
- Reduces exposure to reconnaissance attacks.  
- Promotes consistent browser security policy enforcement.

---

### Frontend Routing (React SSR)
```
location / {
proxy_pass http://prooflift-fe:3000
;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
}
```

#### Explanation:

- Routes all root (`/`) requests to the **React Server-Side Rendering frontend** running at `prooflift-fe:3000`.  
- `proxy_set_header` ensures the original host and real client IP are preserved when forwarding the request.

#### Benefits:

- Provides seamless routing for web clients.  
- Maintains accurate client information for backend logs and analytics.  
- Separates presentation and proxy layers cleanly.

---

### API Gateway Routing (Kong)

```
location /api/ {
limit_req zone=api burst=50 nodelay;
# rewrite ^/api/(.*)$ /$1 break;
proxy_pass http://prooflift-api-gateway:8000/
;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
}
```

#### Explanation:

- Directs all requests with the `/api/` prefix to the **Kong API Gateway** running on port `8000`.  
- Applies the global **rate limiting** rule (`limit_req zone=api`) with a **burst capacity of 50 requests** and no delay, allowing short traffic spikes without queuing.  
- The optional rewrite (commented out) can remove the `/api/` prefix if required by backend routing.

#### Benefits:

- Protects APIs from high-frequency or abusive traffic.  
- Ensures stable and predictable response times.  
- Provides a clean separation between frontend and backend communication.

---

### Error Page Handling

```
error_page 502 503 504 /50x.html;
location = /50x.html {
root /usr/share/nginx/html;
internal;
}
```

#### Explanation:

- Defines custom error handling for upstream (backend) failures like 502, 503, or 504.  
- Serves a static `50x.html` page stored within the container.  
- `internal` prevents direct external access to the error page path.

#### Benefits:

- Provides a consistent user-facing message during downtime.  
- Hides sensitive internal error details.  
- Improves UX and professionalism during outages.

---

### Optional Monitoring Endpoint

```
location /nginx_status {
stub_status on;
access_log off;
allow 127.0.0.1;
deny all;
}
```


#### Explanation:

- Enables NGINX’s built-in `stub_status` for lightweight internal monitoring.  
- Shows active connections, handled requests, and current performance metrics.  
- Access restricted to localhost for security.

#### Benefits:

- Offers real-time visibility into server load and health.  
- Aids diagnostics and performance tuning.  
- Does not expose sensitive information externally.

---

## Summary

This NGINX reverse proxy configuration:

- Routes **web and API traffic** through a single, secure gateway.  
- Implements **rate limiting** to prevent abuse.  
- Enforces **security headers** for protection against common attacks.  
- Provides **error handling** and **local monitoring** for resilience and observability.

Together, these directives create a **hardened, production-ready proxy layer** that safely manages communication between the frontend, API gateway, and backend services.


## 4. Rate Limiting Test Scenario

This section demonstrates how the reverse proxy enforces rate limiting to prevent overload and mitigate DoS attacks.

### Test Setup

A local stress **test** was executed from WSL, sending ***multiple concurrent requests** to the login endpoint:

```
cat > login.json <<'EOF'
{"email":"usuario@example.com","password":"MiClave"}
EOF

# Launch 200 parallel requests and log status codes
rm -f results.txt
for i in $(seq 1 200); do
  curl -s -w "%{http_code}\n" -o /dev/null \
  -X POST http://localhost/api/auth/login \
  -H "Content-Type: application/json" \
  -d @login.json >> results.txt &
done
wait
```

### Request Flow

1. **Client:** The script sends N concurrent requests to ```http://localhost/api/auth/login```.
2. **Reverse Proxy**:
- Applies the rule limit_req_zone ```$binary_remote_addr zone=api:10m rate=30r/s with a burst=50```.

- When the burst queue fills up, new requests exceed the limit and are **rejected immediately**.

- Logged as:

```
2025/11/09 20:02:21 [error] 24#24: *291 limiting requests, excess: 50.520 by zone "api", client: 222.2.222.2, server: localhost, request: "POST /api/auth/login HTTP/1.1", host: "localhost"
203.0.113.1 - - [09/Nov/2025:20:02:21 +0000] "POST /api/auth/login HTTP/1.1" 503 497 "-" "curl/8.5.0"
```
These rejected requests return HTTP 503 (Service Temporarily Unavailable).

3. **Forwarded Requests:**

- Requests that stay within the rate limit are forwarded via proxy_pass ```http://prooflift-api-gateway:8000/;```

4. **API Gateway (Kong):**

- Receives valid forwarded traffic and routes it to the Auth service.

5. **Auth Service:**

- Processes the login request.
- Since the provided credentials are invalid, it returns:

```
HTTP/1.1 404 Not Found
Server: kong/3.9.1
{"message": "User not found / Bad credentials"}
```
## Log Summary
During the test, Nginx logs showed two distinct outcomes:

- Multiple 503 entries — blocked requests due to rate limiting.

- Subsequent 404 entries — valid requests that reached Kong and the Auth service.

| Code | Source | Meaning |
|------|---------|----------|
| **503** | Nginx (Reverse Proxy) | Infrastructure-level protection. Request exceeded rate limit and was dropped before reaching the backend. |
| **404** | Kong / Auth Service | Application-level response. The request passed through Nginx and Gateway but failed authentication (user not found). |










