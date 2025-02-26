Необходим уже запущенный пустой новый кластер k8s и предустановленный локально istioctl с примерами
для kind
```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 8080
    hostPort: 8080
    protocol: TCP
  - containerPort: 8443
    hostPort: 8443
    protocol: TCP
EOF
```
Проверим что все запустилось
kubectl cluster-info --context kind-kind

kubectl cluster-info

Создадим файл для конфигурации istio
```
cat <<EOF > ./tracing.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing: {} # disable legacy MeshConfig tracing options
    extensionProviders:
    - name: jaeger
      opentelemetry:
        port: 4317
        service: jaeger-collector.istio-system.svc.cluster.local
EOF
```

istioctl install -f ./tracing.yaml --skip-confirmation

установим все сервисы мониторинга
Предполагается что вы находитесь в директории `istio-1.24.3`

`kubectl apply -f samples/addons`

`kubectl patch deploy -n istio-system istio-ingressgateway -p '{"spec":{"template":{"spec":{"dnsPolicy":"ClusterFirstWithHostNet","hostNetwork":true}}}}'`


После этого запустим наше демо приложение
```
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```


Затем необходимо добавить телеметрию
```
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: default
spec:
  tracing:
  - providers:
    - name: jaeger
EOF
```

Проверим что у нас что-то собирается
`for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done`

открыть jaeger у себя на хосте
kubectl -n istio-system port-forward svc/tracing 8888:80 --address 0.0.0.0

http://ip:8888
---

