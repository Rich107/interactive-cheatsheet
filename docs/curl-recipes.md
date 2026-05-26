# curl recipes

Practical HTTP request recipes for `curl` — GET, POST, auth, file upload/download, TLS, cookies, retries, and verbose timing. Pair with [jq](https://stedolan.github.io/jq/) for JSON inspection.

## 1. Basic GET with status code

Include response headers (and status line) in the output:

```bash
curl -i https://api.example.com/v1/users
```

Silent mode but still follow redirects and print the body — good for piping into other tools:

```bash
curl -sSL https://api.example.com/v1/users
```

## 2. Custom method and JSON body

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"alice","role":"admin"}' \
  https://api.example.com/v1/users
```

Read the body from a file with `@`:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d @payload.json \
  https://api.example.com/v1/users
```

## 3. Authentication

Basic auth — username + password are base64-encoded into an `Authorization` header:

```bash
curl -u alice:secret https://api.example.com/v1/users
```

Bearer token — pass a JWT or OAuth token via the `Authorization` header:

```bash
curl -H "Authorization: Bearer eyJhbGciOi..." https://api.example.com/v1/users
```

## 4. Custom headers

Add as many `-H` flags as you need. Order is preserved:

```bash
curl -H "Accept: application/json" \
     -H "User-Agent: my-tool/1.0" \
     -H "X-Request-ID: 9b3c-2024" \
     https://api.example.com/v1/users
```

## 5. Multipart file upload

`-F` builds a multipart/form-data body. `@./photo.png` means "attach this file as the field's content":

```bash
curl -X POST \
  -F "file=@./photo.png" \
  -F "title=Avatar" \
  -F "owner=alice" \
  https://api.example.com/v1/users
```

## 6. Download to file

Write the body to a specific filename:

```bash
curl -o response.json https://api.example.com/v1/users
```

Use the remote filename (the last path segment of the URL):

```bash
curl -O https://api.example.com/v1/users
```

## 7. Resume and range requests

Resume an interrupted download — `-C -` tells curl to auto-detect the offset:

```bash
curl -C - -o response.json https://api.example.com/v1/users
```

Fetch just the first kilobyte (HTTP byte-range request):

```bash
curl --range 0-1023 -o response.json https://api.example.com/v1/users
```

## 8. Follow redirects, timing, and verbose

Follow 3xx redirects (default: don't):

```bash
curl -L https://api.example.com/v1/users
```

Print status code and total time after the body, then exit cleanly:

```bash
curl -s -o response.json -w "%{http_code} %{time_total}s\n" https://api.example.com/v1/users
```

Verbose mode — full request/response headers and TLS handshake detail:

```bash
curl -v https://api.example.com/v1/users
```

## 9. Cookies

Save a session's cookies to a jar and reuse them on subsequent requests:

```bash
curl -c cookies.txt -d "user=alice&pass=secret" https://api.example.com/v1/users
curl -b cookies.txt https://api.example.com/v1/users
```

## 10. TLS options

Use a private CA bundle:

```bash
curl --cacert ./ca.pem https://api.example.com/v1/users
```

Mutual TLS — present a client certificate and key:

```bash
curl --cert ./client.pem --key ./client.key https://api.example.com/v1/users
```

Pin the resolved address (useful for testing before DNS cutover):

```bash
curl --resolve api.example.com:443:127.0.0.1 https://api.example.com/v1/users
```

Skip TLS verification (`-k`) — INSECURE; only for throwaway debugging against self-signed certs:

```bash
curl -k https://api.example.com/v1/users
```

## 11. Retries and timeouts

Retry up to 3 times with a 2-second delay between attempts, and cap the whole transfer at 30 seconds:

```bash
curl --retry 3 --retry-delay 2 --max-time 30 https://api.example.com/v1/users
```

## 12. Pipe to jq

`-s` suppresses progress meter; pipe the JSON body straight into `jq`:

```bash
curl -s https://api.example.com/v1/users | jq .
```

Filter a specific field:

```bash
curl -s https://api.example.com/v1/users | jq '.data[].name'
```
