# K8s  Deployment

```
kubectl create ns monitoring

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

helm install tempo grafana/tempo -n monitoring -f - << 'EOF'
tempo:
 extraArgs:
   "distributor.log-received-traces": true
 receivers:
   zipkin:
   otlp:
     protocols:
       http:
       grpc:
EOF
```

Next component? Let’s create a simple deployment of Loki:
```
helm install loki grafana/loki-stack  -n monitoring -f - << 'EOF'
fluent-bit:
 enabled: false
promtail:
 enabled: true
 config:
   clients:
     - url: http://loki.monitoring.svc:3100/loki/api/v1/push 
prometheus:
 enabled: true
 alertmanager:
   persistentVolume:
     enabled: false
 server:
   persistentVolume:
     enabled: false
EOF
```

Sidecar Method
Sidecar method will deploy promtail as a container within a pod that developer/devops create.

This method will deploy promtail as a sidecar container within a pod. In a multi-tenant environment, this enables teams to aggregate logs for specific pods and deployments for example for all pods in a namespace.

```
$ helm show values grafana/promtail > promtail-values.yaml

config:
  clients:
    - url: http://loki.monitoring.svc:3100/loki/api/v1/push
```



Now, let’s deploy Opentelemetry Collector. You use this component to distribute the traces across your infrastructure:
```
kubectl apply -n monitoring -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  labels:
    app: opentelemetry
    component: otel-collector
spec:
  ports:
  - name: otlp # Default endpoint for OpenTelemetry receiver.
    port: 55680
    protocol: TCP
    targetPort: 55680
  - name: jaeger-grpc # Default endpoint for Jaeger gRPC receiver
    port: 14250
  - name: jaeger-thrift-http # Default endpoint for Jaeger HTTP receiver.
    port: 14268
  - name: zipkin # Default endpoint for Zipkin receiver.
    port: 9411
  - name: metrics # Default endpoint for querying metrics.
    port: 8888
  - name: prometheus # prometheus exporter
    port: 8889

  selector:
    component: otel-collector
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  labels:
    app: opentelemetry
    component: otel-collector
spec:
  selector:
    matchLabels:
      app: opentelemetry
      component: otel-collector
  template:
    metadata:
      labels:
        app: opentelemetry
        component: otel-collector
    spec:
      containers:
      - command:
          - "/otelcontribcol"
          - "--config=/conf/otel-collector-config.yaml"
#           Memory Ballast size should be max 1/3 to 1/2 of memory.
          - "--mem-ballast-size-mib=683"
          - "--log-level=DEBUG"
        image: otel/opentelemetry-collector-contrib:0.29.0
        name: otel-collector
        ports:
        - containerPort: 55679 # Default endpoint for ZPages.
        - containerPort: 55680 # Default endpoint for OpenTelemetry receiver.
        - containerPort: 14250 # Default endpoint for Jaeger HTTP receiver.
        - containerPort: 14268 # Default endpoint for Jaeger HTTP receiver.
        - containerPort: 9411 # Default endpoint for Zipkin receiver.
        - containerPort: 8888  # Default endpoint for querying metrics.
        - containerPort: 8889  # prometheus exporter       
        volumeMounts:
        - name: otel-collector-config-vol
          mountPath: /conf
        # livenessProbe:
        #   httpGet:
        #     path: /
        #     port: 13133 # Health Check extension default port.
        # readinessProbe:
        #   httpGet:
        #     path: /
        #     port: 13133 # Health Check extension default port.
      volumes:
        - configMap:
            name: otel-collector-conf
            items:
              - key: otel-collector-config
                path: otel-collector-config.yaml
          name: otel-collector-config-vol
EOF

kubectl apply -n monitoring -f - << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
 name: otel-collector-conf
 labels:
   app: opentelemetry
   component: otel-collector-conf
data:
 otel-collector-config: |
   receivers:
     zipkin:
       endpoint: 0.0.0.0:9411
   exporters:
     otlp:
       endpoint: tempo.monitoring.svc.cluster.local:55680
       insecure: true
   service:
     pipelines:
       traces:
         receivers: [zipkin]
         exporters: [otlp]
EOF
```
Now, the Grafana query. This component is already configured to connect to Loki and Tempo:

```
helm install grafana grafana/grafana -n monitoring --version 6.13.5  -f - << 'EOF'
datasources:
 datasources.yaml:
   apiVersion: 1
   datasources:
     - name: Tempo
       type: tempo
       access: browser
       orgId: 1
       uid: tempo
       url: http://tempo.monitoring.svc:3100
       isDefault: true
       editable: true
     - name: Loki
       type: loki
       access: browser
       orgId: 1
       uid: loki
       url: http://loki.monitoring.svc:3100
       isDefault: false
       editable: true
       jsonData:
         derivedFields:
           - datasourceName: Tempo
             matcherRegex: "traceID=(\\w+)"
             name: TraceID
             url: "$${__value.raw}"
             datasourceUid: tempo

env:
 JAEGER_AGENT_PORT: 6831

adminUser: admin
adminPassword: password

service:
 type: LoadBalancer

EOF
```

Test it

After the installations are completed, let’s open a tunnel to the Grafana query forwarding the port:

```
kubectl port-forward svc/grafana -n monitoring 8081:80
```
Access it using the credentials you have configured when you installed it:

```
user: admin
password: password
http://localhost:8081/explore
```
You are prompted to the Explore tab. There, you can select Loki to be displayed on one side and then click on split to choose Tempo to be displayed in the other side: