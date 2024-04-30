# APISIX

## HTTPS

### prepare

- generate test certificate

```bash
# Generate the certificate authority (CA) key and certificate
openssl genrsa -out ca.key 2048
openssl req -new -sha256 -key ca.key -out ca.csr -subj "/CN=ROOTCA"
openssl x509 -req -days 36500 -sha256 -extensions v3_ca -signkey ca.key -in ca.csr -out ca.crt

# Generate the key and certificate for server, sign with CA certificate
openssl genrsa -out server.key 2048
openssl req -new -sha256 -key server.key -out server.csr -subj "/CN=example.com"
openssl x509 -req -days 36500 -sha256 -extensions v3_req  -CA  ca.crt -CAkey ca.key  -CAserial ca.srl  -CAcreateserial -in server.csr -out server.crt

# Generate the key and certificate for client, sign with CA certificate
openssl genrsa -out client.key 2048
openssl req -new -sha256 -key client.key  -out client.csr -subj "/CN=TEST-CLIENT"
openssl x509 -req -days 36500 -sha256 -extensions v3_req  -CA  ca.crt -CAkey ca.key  -CAserial ca.srl  -CAcreateserial -in client.csr -out client.crt

# to pkcs12 format only for Wins
#openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
```

### add certificate

```bash
curl -i "http://127.0.0.1:9180/apisix/admin/ssls" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "id": "tls-client-ssl",
  "sni": "example.com",
  "cert": "'"$(cat ./server.crt)"'",
  "key": "'"$(cat ./server.key)"'"
  }'
```

### Add Route

```bash
curl -i "http://127.0.0.1:9180/apisix/admin/routes" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "id": "route-client-ip",
  "uri": "/ip",
  "upstream": {
    "nodes": {
      "httpbin.org:80":1
    },
    "type": "roundrobin"
  }
}'
```

### Test https

```sh
curl -ikv --resolve "example.com:9443:127.0.0.1" https://example.com:9443/ip
```

## mTLS

```bash
curl -X PUT 'http: //127.0.0.1:9180/apisix/admin/ssls' \
--header 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \
--header 'Content-Type: application/json' \
--data-raw '{
  "id": "mTLS-client-with-skip",
  "sni": "example.com",
  "cert": "'"$(cat ./server.crt)"'",
  "key": "'"$(cat ./server.key)"'",
  "client": {
    "ca": "'"$(cat ./ca.crt)"'",
    "depth": 10,
    "skip_mtls_uri_regex": [
      "/anything.*"
    ]
  }
}'
```

### add route

```bash
curl "http://127.0.0.1:9180/apisix/admin/routes" \
-H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "id": "route-mtls",
  "methods": [
    "GET"
  ],
  "host": "example.com",
  "uri": "/anything",
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "httpbin.org:80":1
    }
  }
}'
```

### Test mTLS

```sh
curl -ikv --resolve "example.com:9443:127.0.0.1" https://example.com:9443/uuid
curl -ikv --resolve "example.com:9443:127.0.0.1" https://example.com:9443/uuid --cert ./client.crt --key ./client.key
# skip uri
curl -ikv --resolve "example.com:9443:127.0.0.1" https://example.com:9443/anything
```

## Serverless

```bash
curl -X PUT 'http: //127.0.0.1:9180/apisix/admin/routes/1' \
 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \
 -H 'Content-Type: application/json' \
 -d '{
  "uri": "/post",
  "plugins": {
    "serverless-pre-function": {
      "phase": "rewrite",
      "functions": ["
          return function(conf, ctx)
          -- Import neccessary libraries
          local http = require(\"resty.http\")
          local core = require(\"apisix.core\")
          local redis = require(\"resty.redis\")
          local red = redis:new()
          -- red:set_timeouts(1000,1000,1000)
          local ok, err = red:connect(\"172.20.0.7\", 6379)
          if not ok then
              ngx.say(\"1: failed to connect: \", err)
          return
          end
          local method = core.request.get_method()
          local res, err = red:set(method, \"ksjdlfjasdlkfjlksdf\")
          -- Send the webhook notification only if the request method is POST, otherwise skip and send it to the upstream as usual
          if core.request.get_method() == \"POST\" then
              -- Send the webhook notification to the specified URL
              local httpc = http.new()
              local res, err = httpc:request_uri(\"https://webhook.site/b73131f2-0e63-4246-a533-5929bc588ab1\", {
                  method = \"POST\",
                  headers = {
          [\"Content-Type\"] = \"application/json\"
          },
                  body = core.request.get_body(),
          })
              -- Check the response from the webhook
              if not res then
                  core.log.error(\"Failed to send webhook: \", err)
                  return 500, err
              end
          end

          -- Return the response from the upstream service
          return conf.status, conf.body
          end"
      ]
    }
  },
  "upstream": {
    "nodes": {
      "httpbin.org:80": 1
    },
    "type": "roundrobin"
  }
}'
```
