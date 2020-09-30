
## Install redis via helm

for details see [here](https://github.com/bitnami/charts/tree/master/bitnami/redis-cluster)

Preparation

```shell
oc new-project redis
export uid=$(oc get project redis -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'|sed 's/\/.*//')
```

Deploy cert-manager (skip if already present in the cluster)

```shell
oc new-project cert-manager
oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.yaml
```

Deploy local CA

```shell
oc apply -f ./redis-certs.yaml -n redis
```

Deploy redis-proxy

```shell
export default_domain=$(oc get Ingress.config.openshift.io cluster -o jsonpath='{.spec.domain}')
helm upgrade -i redis-proxy ./redis-proxy -n redis --values values-production-updated.yaml --set default_domain=${default_domain}
```

Deploy redis

```shell
export uid=$(oc get project redis -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'|sed 's/\/.*//')
helm install redis-cluster redis-cluster -n redis --values values-production-updated.yaml --set containerSecurityContext.runAsUser=${uid} --set podSecurityContext.fsGroup=${uid}
```

### Test connectivity

```shell
export REDIS_PASSWORD=$(kubectl get secret -n redis redis-cluster -o jsonpath="{.data.redis-password}" | base64 --decode)
#Openshift route not working
export REDIS_URL=$(oc get route redis-cluster-envoy-proxy -n redis -o jsonpath='{.spec.host}')
oc get secret redis-cluster-envoy-proxy-tls -n redis -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/ca.crt
docker run --entrypoint redis-cli -v /tmp/ca.crt:/tmp/ca.crt:z docker.io/bitnami/redis-cluster:6.0.8-debian-10-r11 -h ${REDIS_URL} -p 443 -a $REDIS_PASSWORD --tls --cacert /tmp/ca.crt --verbose --sni ${REDIS_URL} ping

#External Load Balancer working
export REDIS_URL=$(oc get svc redis-cluster-envoy-proxy -n redis -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
docker run --entrypoint redis-cli -v /tmp/ca.crt:/tmp/ca.crt:z docker.io/bitnami/redis-cluster:6.0.8-debian-10-r11 -h ${REDIS_URL} -p 6378 -a $REDIS_PASSWORD --tls --cacert /tmp/ca.crt --verbose ping
```
