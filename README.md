# GKE gRPC Ingress LoadBalancing

## Setup

Certificates in the certs folder is created exactly as described in the double-proxy envoy example
https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/double-proxy#install-sandboxes-double-proxy

The postgres backed in that example is exchanged with an echo server from https://github.com/kalmhq/echoserver


## Docker-compose 

The docker-compose solution should be possible to start and test the secure http2 solution locally.

Test certificate validation through the envoy proxy

```
curl --request POST \
     --url    https://proxy-postgres-backend.example.com:443/envoy/tls/foo?bar=q \
     --header  'Content-Type: application/json' \
     -d       '{ "foobar": 34}' \
     --resolve 'proxy-postgres-backend.example.com:443:127.0.0.1' \
     --cert   certs/postgres-frontend.example.com.crt \
     --key    certs/example.com.key \
	   --cacert certs/ca.crt \
     --http2 \
     -v

```

Test certificate validation through the nginx proxy

```
curl --request POST \
     --url    https://proxy-postgres-backend.example.com:444/http3/tls/foo?bar=q \
     --header  'Content-Type: application/json' \
     -d       '{ "foobar": 34}' \
     --resolve 'proxy-postgres-backend.example.com:444:127.0.0.1' \
     --cert   certs/postgres-frontend.example.com.crt \
     --key    certs/example.com.key \
	   --cacert certs/ca.crt \
     --http2 \
     -v

```

## Install in gke

Create certs as secrets and envoy-proxy and nginx proxy configurations as configmaps
All artifacts are put in the echo namespace

```
kubectl create secret -n echo generic be-echo --type kubernetes.io/tls \
  --from-file=tls.crt=certs/postgres-backend.example.com.crt \
  --from-file=tls.key=certs/example.com.key \
  --from-file=ca.crt=certs/ca.crt 

kubectl create configmap -n echo envoy-echo \
  --from-file=envoy.yaml=envoy-echo-cfg.yaml 

kubectl create configmap -n echo nginx-echo --from-file=default.conf


```

Install deployment, service and ingress

```
kubectl apply -n echo -f .
```

The ingress serves different routes where the request is passed directly to the echo backend and others where it is passed via a proxy for 
certificate validation. See ingress.yaml for details 

After all is install find the ip address for the ingress and replace localhost in the `--resolve` switch with it.

Try the http2 backend (without cert validation)

```
curl --request POST \
     --url    https://proxy-postgres-backend.example.com:443/http2/tls/foo?bar=q \
     --header  'Content-Type: application/json' \
     -d       '{ "foobar": 34}' \
     --resolve 'proxy-postgres-backend.example.com:443:130.211.33.179' \
     --insecure \
     --http2 \
     -v
```

Through the envoy proxy.

```
curl --request POST \
     --url    https://proxy-postgres-backend.example.com:443/envoy/tls/foo?bar=q \
     --header  'Content-Type: application/json' \
     -d       '{ "foobar": 34}' \
     --resolve 'proxy-postgres-backend.example.com:443:130.211.33.179' \
     --cert   certs/postgres-frontend.example.com.crt \
     --key    certs/example.com.key \
	   --cacert certs/ca.crt \
     --http2 \
     -v
```

Through the nginx proxy.

```
curl --request POST \
     --url    https://proxy-postgres-backend.example.com:443/nginx/tls/foo?bar=q \
     --header  'Content-Type: application/json' \
     -d       '{ "foobar": 34}' \
     --resolve 'proxy-postgres-backend.example.com:443:130.211.33.179' \
     --cert   certs/postgres-frontend.example.com.crt \
     --key    certs/example.com.key \
	   --cacert certs/ca.crt \
     --http2 \
     -v
```

Other useful curl switches I've tried.

```
	   --keepalive-time 650 \
     -H 'Authority: "postgres-backend.example.com"' \
     -H 'Host: "postgres-backend.example.com"' \
     --trace - \
	   --no-keepalive \

```