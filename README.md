# hub-multitenancy

Test hub apim multitenancy

## Gen certs

### Generate wildcard for domain

```bash
LEGO_DISABLE_CNAME_SUPPORT=true lego -a --path .lego/. --email mageekbox@gmail.com --dns gandiv5 -d "tenant1.${CLUSTERNAME}.${DOMAINNAME}" -d "*.tenant1.${CLUSTERNAME}.${DOMAINNAME}" run
LEGO_DISABLE_CNAME_SUPPORT=true lego -a --path .lego/. --email mageekbox@gmail.com --dns gandiv5 -d "tenant2.${CLUSTERNAME}.${DOMAINNAME}" -d "*.tenant2.${CLUSTERNAME}.${DOMAINNAME}" run
LEGO_DISABLE_CNAME_SUPPORT=true lego -a --path .lego/. --email mageekbox@gmail.com --dns gandiv5 -d "tenant3.${CLUSTERNAME}.${DOMAINNAME}" -d "*.tenant3.${CLUSTERNAME}.${DOMAINNAME}" run
```

## deploy Redis

### For distributed features

```bash
helm upgrade --install redis bitnami/redis --namespace redis --values redis/values.yaml --create-namespace
```

## Deploy Hub tenant1

### Create tenant1 namespace

```bash
kubectl create ns tenant1
```

## Create tenant1 secret from cert file

```bash
kubectl create secret tls tenant1-wildcard-mageekbox --namespace tenant1 --cert=.lego/certificates/tenant1.${CLUSTERNAME}.${DOMAINNAME}.crt --key=.lego/certificates/tenant1.${CLUSTERNAME}.${DOMAINNAME}.key
```

### Create tenant1 Hub token secret

```bash
kubectl create secret generic hub-license --from-literal=token="${TENANT1_HUB_TOKEN}" -n tenant1
```

### deploy tenant1 Traefik

```bash
helm upgrade --install traefik traefik/traefik --create-namespace --namespace tenant1 --values tenant1/hub/hub-values.yaml
```

### Set DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.tenant1.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik -n tenant1 --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### Add tenant1 Hub dashboard ingress if needed

```bash
envsubst < tenant1/hub/dashboard.yaml | kubectl apply -f -
```

### deploy tenant1 api

### use tenant1 file to deploy

```bash
kubectl apply -f tenant1/apis
kubectl apply -f tenant1/apim-objs
```

## Deploy Hub tenant2

### Create tenant2 namespace

```bash
kubectl create ns tenant2
```

## Create tenant2 secret from cert file

```bash
kubectl create secret tls tenant2-wildcard-mageekbox --namespace tenant2 --cert=.lego/certificates/tenant2.${CLUSTERNAME}.${DOMAINNAME}.crt --key=.lego/certificates/tenant2.${CLUSTERNAME}.${DOMAINNAME}.key
```

### Create tenant2 Hub token secret

```bash
kubectl create secret generic hub-license --from-literal=token="${TENANT2_HUB_TOKEN}" -n tenant2
```

### deploy tenant2 Traefik

```bash
helm upgrade --install traefik traefik/traefik --create-namespace --namespace tenant2 --values tenant2/hub/hub-values.yaml
```

### Set tenant2 DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.tenant2.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik -n tenant2 --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### Add tenant2 Hub dashboard ingress if needed

```bash
envsubst < tenant2/hub/dashboard.yaml | kubectl apply -f -
```

## deploy tenant2 api

### use tenant2 file to deploy

```bash
kubectl apply -f tenant2/apis
kubectl apply -f tenant2/apim-objs
```


## Deploy Hub tenant3

### Create tenant3 namespace

```bash
kubectl create ns tenant3
```

## Create tenant3 secret from cert file

```bash
kubectl create secret tls tenant3-wildcard-mageekbox --namespace tenant3 --cert=.lego/certificates/tenant3.${CLUSTERNAME}.${DOMAINNAME}.crt --key=.lego/certificates/tenant3.${CLUSTERNAME}.${DOMAINNAME}.key
```

### Create tenant3 Hub token secret

```bash
kubectl create secret generic hub-license --from-literal=token="${TENANT3_HUB_TOKEN}" -n tenant3
```

### deploy tenant3 Traefik

```bash
helm upgrade --install traefik traefik/traefik --create-namespace --namespace tenant3 --values tenant3/hub/hub-values.yaml
```

### Set tenant3 DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.tenant3.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik -n tenant3 --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### Add tenant3 Hub dashboard ingress if needed

```bash
envsubst < tenant3/hub/dashboard.yaml | kubectl apply -f -
```

## deploy tenant3 api

### use tenant3 file to deploy

```bash
kubectl apply -f tenant3/apis
kubectl apply -f tenant3/apim-objs
```

## Deploy Monitoring

```bash
helm upgrade --install loki grafana/loki --create-namespace --namespace observability --values observability/loki/values.yaml
kubectl apply -f observability/jaeger
helm upgrade --install promtail grafana/promtail --create-namespace --namespace observability --values observability/promtail/values.yaml
kubectl create configmap grafana-traefik-dashboards --from-file=observability/prometheus-stack/traefik.json --from-file=observability/prometheus-stack/api.json --from-file=observability/prometheus-stack/ai.json -o yaml --dry-run=client -n observability | kubectl apply -f -
helm upgrade -i prometheus-stack prometheus-community/kube-prometheus-stack -f observability/prometheus-stack/values.yaml --namespace=observability
envsubst < observability/prometheus-stack/ingress.yaml | kubectl apply -f -
```
