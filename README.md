# Introduction 
We need a Project demonstrating Complete Observability Stack utilizing Prometheus, Loki (For distributed logging), Tempo (For Distributed monitoring, this basically uses Jaeger Internally), Grafana for Java/spring based applications (With OpenTelemetry auto / manual Instrumentation) involving multiple microservices with DB interactions

![alt text](/images/getting-started.png "Getting Started")

## Index

* [Instructions](#instructions)
* [Proposed Architecture](#proposed-architecture)
* [Local Env](#local-env)
* [CICD - Automation](#cicd-automation-bonus)
* [Observability](#observability-bonus)
* [Permissions](#permissions-bonus)
* [Best Practices](#best-practices-bonus)
* [Disaster Recovery Plan](#disaster-recovery-plan-bonus)
* [Compliance](#compliance-bonus)
* [Budget](#budget-bonus)


## Instructions

This project demonstrates Observability using:

* Prometheus for monitoring and alerting
* Loki for Distributed Logging
* Tempo for Distributed monitoring
* Grafana for visualization
* Otel Collector for sending traces

And basically integrates the following

* Opentelemetry
* Grafana Tempo Which internally uses Jaeger
* Spring Boot Project

Its divided into two flavours

<summary><b>Local Env</b></summary>
<details>

---
Running

Option 1: Azure Registry
We have added the base images in the docker compose in the azure registry

```
boot-otel-tempo-api
boot-otel-tempo-docker
boot-otel-tempo-provider1
volume_exporter
```
Az login

```
az account set --subscription 9b8e09f0-efe4-417a-81c3-f90402519522
az acr login --name Testsandbox
```

Docker Compose
```
docker-compose up
```

Option 2: Build locally

If not you have the testing application to review and you can execute the following on root folder

```

mvn clean package docker:build
```

Images
```

docker image ls
```

```
REPOSITORY                                                      TAG                 IMAGE ID            CREATED              SIZE
boot-otel-tempo-provider1                               0.0.1-SNAPSHOT      7ddceebcc722        About a minute ago   169MB
boot-otel-tempo-api                                     0.0.1-SNAPSHOT      a301242388a1        2 minutes ago        147MB
boot-otel-tempo-docker                                  0.0.1-SNAPSHOT      061a20db744b        4 minutes ago        130MB
```

And then either docker compose or docker stack

Docker Compose
```
docker-compose up
```

Docker ps
```

caf0427ba517   grafana/grafana:7.4.0-ubuntu                       "/run.sh"                26 minutes ago   Up 26 minutes   0.0.0.0:3000->3000/tcp                                                                                 bootstrap-opentelemetry_grafana_1
dc37b9ec73c0   grafana/promtail:2.2.0                             "/usr/bin/promtail -…"   27 minutes ago   Up 26 minutes                                                                                                          bootstrap-opentelemetry_promtail_1
9d9a3af1e179   prom/prometheus:latest                             "/bin/prometheus --c…"   27 minutes ago   Up 26 minutes   0.0.0.0:9090->9090/tcp                                                                                 bootstrap-opentelemetry_prometheus_1
9ee59cc905da   grafana/tempo-query:0.7.0                          "/go/bin/query-linux…"   27 minutes ago   Up 27 minutes   0.0.0.0:16686->16686/tcp                                                                               bootstrap-opentelemetry_tempo-query_1
fffbbba81704   grafana/loki:2.2.0                                 "/usr/bin/loki -conf…"   27 minutes ago   Up 27 minutes   0.0.0.0:3101->3100/tcp                                                                                 bootstrap-opentelemetry_loki_1
```


</details>

<summary><b>K8s </b></summary>
<details>

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


</details>

<summary><b>Flux </b></summary>
<details>



source.yaml
```
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  gitImplementation: libgit2
  interval: 5m0s
  ref:
    branch: main
  secretRef:
    name: azdo-ro-pat
  timeout: 20s
  url: https://dev.azure.com/prestoq/Infrastructure/_git/infra-deployment
  ignore: |
    /*
    !/flux/clusters/swdev
    !/flux/components
    !/flux/resources
```

helmrelease.yaml
```
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: otel-collector
spec:
  chart:
    spec:
      chart: opentelemetry-collector
      sourceRef:
        kind: HelmRepository
        name: Test
        namespace: monitoring
      version: 0.40.2
  install: {}
  interval: 60m0s

  values:
    mode: deployment
    replicaCount: 1
    presets:
      clusterMetrics:
        enabled: true
      kubernetesAttributes:
        enabled: true
    config:
      exporters:
        otlphttp:
          endpoint: https://otlp.nr-data.net:4318
        logging:
          logLevel: DEBUG
      processors:
        batch:
          send_batch_size: 10000
          send_batch_max_size: 11000
          timeout: 10s
        resource:
            attributes:
              - key: host.id
                from_attribute: host.name
                action: upsert
        resourcedetection:
          detectors: [ azure ]
          timeout: 2s
        k8sattributes:
          auth_type: "serviceAccount"
          passthrough: false
          filter:
            node_from_env_var: KUBE_NODE_NAME
          extract:
            metadata:
              - k8s.pod.name
              - k8s.pod.uid
              - k8s.deployment.name
              - k8s.namespace.name
              - k8s.node.name
              - k8s.pod.start_time
          pod_association:
            - sources:
              - from: resource_attribute
                name: k8s.pod.uid
      service:
        pipelines:
          metrics:
            exporters: [ logging, otlphttp ]
            processors: [ resourcedetection, k8sattributes, resource, batch ]
          traces:
            exporters: [ logging, otlphttp ]
            processors: [ resourcedetection, k8sattributes, resource, batch ]

  valuesFrom:
  - kind: Secret
    name: newrelic-otlp-key-helmvalues
```

</details>


## Questions / FAQ

<details>
<summary><b>How would the observabiliy stack deploy flow be for someone who is not a devops and does not have access to k8s, how would you automate it?</b></summary>

---
WIP
</details>

<details>
<summary><b>How long does the Log retention last?</b></summary>

---
WIP
<details>
<summary><b>How long will it take to set up the required dashboards, alerts, and other analytics I need to analyze my data?
</b></summary>

---
WIP
</details>
</details>

<details>
<summary><b>What values ​​of the resources would you monitor?</b></summary>

---
WIP
<details>
<summary><b>How long will it take to set up the required dashboards, alerts, and other analytics I need to analyze my data?
</b></summary>

---
WIP
</details>
</details>

<details>
<summary><b>How long will it take to get my telemetry data pipeline up and running?</b></summary>

---
WIP

</details>

<details>
<summary><b>How long will it take to set up the required dashboards, alerts, and other analytics I need to analyze my data?
</b></summary>

---
WIP
</details>

<details>
<summary><b>How are we going to secure the access to the grafana dashboard?
</b></summary>

---
WIP
</details>

<details>
<summary><b>
</b></summary>

---
WIP
</details>


## Proposed Architecture

The following is ***a proposal*** for local env and Cloud

### Local Env
![alt text](/images/grafana_loki_tempo_promtail.png "Proposed diagram")

### K8s

![alt text](/images/k8s_tempo_grafana_architecture.jpg "Proposed diagram")

### Flux 
![alt text](/images/flux.png "Proposed diagram")


The proposed solution performs the following acctions. 
```
* Prometheus for monitoring and alerting
* Loki for Distributed Logging
* Tempo for Distributed monitoring
* Grafana for visualization
* Otel Collector for sending traces
```

![alt text](/images/structure.png "Proposed diagram")

Distributor: The distributor handles the logs written by the log processors and forwarding clients (Otel Collector). You can find the list of all supported clients here. When the distributor receives the streams from the clients, they are first validated for correctness, split into batches, and then sent to multiple ingesters.
Example clients:
a) Promtail
b) Fluentd
c) Fluent Bit
b) Logstash etc …..

Ingester: The Ingester service is responsible for sending the chunks of data to long-term storage backends like S3, GCS, etc. The ingesters validate timestamps for each log line received maintains a strict ordering. When a flush occurs to a persistent storage provider, the chunk is hashed based on its tenant, labels, and contents. This means that multiple ingesters with the same copy of data will not write the same data to the backing store twice.

Installing Loki, Promtail, Prometheus Operator
We will use the official helm chart to install Loki. The Loki stack helm chart supports the installation of various components like promtail, fluentd, Prometheus and Grafana. Well however you might already have been using Prometheus Operator or you might already have Grafana installed. You might just want to append your Grafana data source. For the log processors and forwarding clients, we will use Promtail in the scope of this article. Promtail is an agent which ships the contents of local logs to a Loki instance.

The Loki Operator provides Kubernetes native deployment and management of Loki and related logging components. The purpose of this project is to simplify and automate the configuration of a Loki based logging stack for Kubernetes clusters.
a) Loki Instance
b) Prometheus Alert Manager
c) Prometheus Node Exporter
d) Prometheus Push Gateway
e) Prometheus Server
f) Promtail

Installing Tempo
Tempo is used to correlate the metrics, traces, and logs. There are situations where a user is getting the same kind of error multiple times. If I want to understand what is happening, I will need to look at the exact traces. But because of downsampling, some valuable information which I might be looking for would have got lost. With Tempo, now we need not downsample distributed tracing data. We can store the complete trace in object storage like S3 or GCS, making Tempo very cost-efficient.
Also, Tempo enables you for faster debugging/troubleshooting by quickly allowing you to move from metrics to the relevant traces of the specific logs which have recorded some issues.



## Requirements

* An active Azure account
* Access to Test Sandbox Account
* Azure credentials 
* Flux => https://fluxcd.io/flux/installation/
* AZ CLI => https://github.com/Azure/azure-cli

## Constrains for Local/ Cloud Deployment

* Flux should be configured in the cluster with the external secrets (cloud)
* A blob to store the backend monitoring logs (cloud)
* Supporting infrastructure for Cloud deployment (Databases, blob storage)
* Local environment (.env) vars for test deployment (local)


# Process for local testing

1. Use the `env.template` file to create the `.env` file.
2. Populate the `.env` file with your AWS access KEYs and selected Region.
3. Execute `source .env`.
4. Execute `docker-compose up`

## Docker Compose
<summary><b>YAML </b></summary>
<details>

```
version: "3.8"

services:

  loki:
    image: grafana/loki:2.2.0
    command: -config.file=/etc/loki/loki-local.yaml
    user: "0"
    ports:
      - "3101:3100"                                   # loki needs to be exposed so it receives logs
    environment:
      - JAEGER_AGENT_HOST=tempo
      - JAEGER_ENDPOINT=http://tempo:14268/api/traces # send traces to Tempo
      - JAEGER_SAMPLER_TYPE=const
      - JAEGER_SAMPLER_PARAM=1
    volumes:
      - ./etc/loki-local.yaml:/etc/loki/loki-local.yaml
      - ./data/loki-data:/tmp/loki

  provider1-db:
    image: postgres
    restart: always
    environment:
      - POSTGRES_DB=provider1
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgrespassword
      - PGDATA=/var/lib/postgresql/data/pgdata
    ports:
      - 5432:5432
    volumes:
      - ./boot-otel-tempo-provider1/db/data:/var/lib/postgresql
      - ./boot-otel-tempo-provider1/db/scripts/init.sql:/docker-entrypoint-initdb.d/init.sql

  pgadmin:
    image: dpage/pgadmin4
    ports:
      - 7070:80
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: pgadmin4@pgadmin.org
      PGADMIN_DEFAULT_PASSWORD: admin
    depends_on:
      - provider1-db

  tempo:
    image: grafana/tempo:0.7.0
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./etc/tempo-local.yaml:/etc/tempo.yaml
      - ./data/tempo-data:/tmp/tempo
    restart: unless-stopped  
    ports:
      - "14268:14268"  # jaeger ingest, Jaeger - Thrift HTTP
      - "14250:14250"  # Jaeger - GRPC
      - "55680:55680"  # OpenTelemetry
      - "3102:3100"   # tempo

  tempo-query:
    image: grafana/tempo-query:0.7.0
    command: ["--grpc-storage-plugin.configuration-file=/etc/tempo-query.yaml"]
    volumes:
      - ./etc/tempo-query.yaml:/etc/tempo-query.yaml
    ports:
      - "16686:16686"  # jaeger-ui
    depends_on:
      - tempo

  boot-otel-tempo-provider1:
    image: "bootstrap-opentelemetry/boot-otel-tempo-provider1:0.0.1-SNAPSHOT"
    ports:
      - "8090:8090"
    environment:
      PROVIDER1_DB_URL: jdbc:postgresql://provider1-db:5432/provider1
      PROVIDER1_DB_USER: postgres
      PROVIDER1_DB_PASS: postgrespassword
    volumes:
      - ./data/logs:/app/logs
    depends_on:
      - tempo
      - provider1-db

  boot-otel-tempo-api:
    image: "bootstrap-opentelemetry/boot-otel-tempo-api:0.0.1-SNAPSHOT"
    ports:
      - "8080:8080"
    environment:
      PROVIDER1_URL_BASE: "http://boot-otel-tempo-provider1:8090"
    volumes:
      - ./data/logs:/app/logs
    depends_on:
      - boot-otel-tempo-provider1

  promtail:
    image: grafana/promtail:2.2.0
    command: -config.file=/etc/promtail/promtail-local.yaml
    volumes:
      - ./etc/promtail-local.yaml:/etc/promtail/promtail-local.yaml
      - ./data/logs:/app/logs
    depends_on:
      - boot-otel-tempo-api
      - loki

  volume_exporter:
    image: bootstrap-opentelemetry/volume_exporter
    command: ["--volume-dir=tempo:/tmp/tempo", "--volume-dir=logs:/app/logs", "--volume-dir=loki:/tmp/loki"]
    volumes:
      - ./data/logs:/app/logs
      - ./data/tempo-data:/tmp/tempo
      - ./data/loki-data:/tmp/loki
    ports:
      - 9889:9888

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./etc/prometheus.yaml:/etc/prometheus.yaml
    entrypoint:
      - /bin/prometheus
      - --config.file=/etc/prometheus.yaml
    ports:
      - "9090:9090"
    depends_on:
      - boot-otel-tempo-api
      - volume_exporter

  grafana:
    image: grafana/grafana:7.4.0-ubuntu
    volumes:
      - ./data/grafana-data/datasources:/etc/grafana/provisioning/datasources
      - ./data/grafana-data/dashboards-provisioning:/etc/grafana/provisioning/dashboards
      - ./data/grafana-data/dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_AUTH_DISABLE_LOGIN_FORM=true
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - tempo-query
      - loki

```
</details>

# Testing Observability Stack (Shift left) - Local Stack

[Access the endpoint](http://localhost:8080/flights)

![](/images/access-flights.png)

View the log and trace in [Grafana](http://localhost:3000/explore)

![](/images/grafana-loki-trace.png)


Get the trace information Using **[Jaeger](http://localhost:16686/search)** as well

**Basic Trace**

![](/images/jaeger-trace.png)


## Prometheus Metrics

View the metrics in [prometheus](http://localhost:9090/graph?g0.expr=&g0.tab=1&g0.stacked=0&g0.range_input=1h)

![](/images/prometheus-metrics.png)

You can view it in [Grafana](http://localhost:3000/explore?orgId=1&left=%5B%22now-1h%22,%22now%22,%22Prometheus%22,%7B%22expr%22:%22http_server_requests_seconds_count%22,%22requestId%22:%22Q-0a6b4a46-2eeb-428a-b98d-0170a5fe4900-0A%22%7D%5D) as well

![](/images/grafana-prom-metrics.png)

# Configuring your service to send traces

## K8s Services example

![](/images/k8s_services.png)

VM Arguments:
```

-javaagent:opentelemetry-javaagent.jar
-Dotel.exporter.otlp.traces.protocol=http/protobuf
-Dotel.traces.exporter=otlp
-Dotel.exporter.otlp.traces.endpoint=http://otel-collector.monitoring.svc:55680/v1/traces
-Dotel.resource.attributes=service.name="serviceName"
-Dotel.metrics.exporter=prometheus
-Dotel.exporter.prometheus.port=8889
```

# Testing Observability Stack (Shift left) - K8s
```
Sample OTLP JSON Trace Span
Here’s a minimal example trace in JSON, with all the fields required by the OTLP/HTTP protocol schema (as of August 2022). 

{
 "resourceSpans": [
   {
     "resource": {
       "attributes": [
         {
           "key": "service.name",
           "value": {
             "stringValue": "test-with-curl"
           }
         }
       ]
     },
     "scopeSpans": [
       {
         "scope": {
           "name": "manual-test"
         },
         "spans": [
           {
             "traceId": "71699b6fe85982c7c8995ea3d9c95df2",
             "spanId": "3c191d03fa8be065",
             "name": "spanitron",
             "kind": 2,
             "droppedAttributesCount": 0,
             "events": [],
             "droppedEventsCount": 0,
             "status": {
               "code": 1
             }
           }
         ]
       }
     ]
   }
 ]
}
```

If you’re going to send the span with curl, put this into a file called `span.json`.

If you’re going to send it with Postman, put this into the request body.

Here are the values you might want to replace:

Service name: “test-with-curl” in this JSON is going to show up in the `service.name` field in your trace span. In Honeycomb, you’ll look for the span in that service dataset. You might want a different service name for this test.
Library name: “manual-test” in this JSON is going to show up in the `library.name` field in your trace span. That describes what instrumentation code created this.
Name: “spanitron” in this JSON is going to show up in the `name` field in your trace span. You might want to make this more descriptive.
Span kind: the value of “kind” in this JSON is 2, which corresponds to a `span.kind` of `server`. That’s usually the first kind of span any app creates when it receives a request. The other options are 1 for `client` or 3 for `internal` instead. (For reference: the schema.)
Send a span to the collector
Here’s the command. Check the URL and then run this at the command line, or paste it into Postman.

`curl -i http://localhost:4318/v1/traces -X POST -H "Content-Type: application/json" -d @span.json`

Here,

 `-i` means “show me what happens”
`http://localhost` is the URL to the collector—yours might be different
`4318` is the port standard port for OTLP/HTTP
`/v1/traces` is the standard endpoint for OTLP traces
`-X POST` sends an HTTP POST request
`-H "Content-Type: application/json"` says “here comes some JSON”
`-d @span.json` says “read the body of the request from a file called span.json”
If you’re using Postman, you’ll probably paste in the body separately, so you won’t need the `-d span.json`.

The collector should respond happily, and say very little:
```
HTTP/1.1 200 OK
Content-Type: application/json
Vary: Origin
Date: Tue, 12 Jul 2022 18:44:50 GMT
Content-Length: 2

{}
```

## Observability - WIP

We will consider the following metrics
```
* Scalability
* Reliability
* Availability
* Latency
* Fexibility 
```

### Grafana Dashboards Automation - WIP
We can use grafana provisioning to specify a folder fo dashboard json files to go and place the json exports in that folder. We can also use it to setup datasources as well

https://grafana.com/docs/grafana/latest/administration/provisioning/

## Monitoring and Alerting - WIP

We will use the observability stack to monitor the following

Key metrics for monitoring
```
- WIP
```

## CICD Automation - WIP


Using a CI/CD tool (i.e. Azure Devops) CD (Flux)
```
1. The CICD will create the K8s cluster
2. Using the IaC tooling of choise will run the fmt/ validate/ plan.
3. Deploy the flux operator
4. Flux configuration wil enable the installation of the observability stack

```

## Permissions - WIP

All infrastructure authentication is controlled by Active Directory 

We will use the principle of Least Priviledge using the Azure AD authentication.

It will allow to use an Azure Active Directory tenant as an identity provider for Grafana. You can use Azure AD Application Roles to assign users and groups to Grafana roles from the Azure Portal. 

Steps
```
Azure AD OAuth2 authentication
Create the Azure AD application
Enable Azure AD OAuth in Grafana
Configure allowed groups
Configure allowed domains

```
## Disaster Recovery Plan - WIP

## Calculation Report - WIP

The volatility and scale of telemetry data generated by modern cloud environments can quickly rack up costs before teams are even aware of what’s happening. These costs can take different forms.


![alt text](/images/estimate.png "Azure price estimation")

The above was generated using https://azure.microsoft.com/en-us/pricing/calculator/.  Is an approximation for heavy usage on the 100 million requests per month. 

The table compares the up front price tag, infrastructure costs, and the time needed to set up and maintain a logging pipeline that can handle 1TB/day

In order to make a cost effective use, we will implement process filtering in the opentelemetry collector.

Filtering out telemetry data
Once you’ve identified data doesn’t help you monitor a critical signal, the next step is to filter out that data before incurring the costs for it.

Filtering out metrics --> https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md

<summary><b>YAML </b></summary>
<details>

```

processors:
  filter/1:
    metrics:
      include:
        match_type: regexp
        metric_names:
          - prefix/.*
          - prefix_.*
        resource_attributes:
          - Key: container.name
            Value: app_container_1
      exclude:
        match_type: strict
        metric_names:
          - hello_world
          - hello/world
  filter/2:
    logs:
      include:
        match_type: strict
        resource_attributes:
          - Key: host.name
            Value: just_this_one_hostname
  filter/regexp:
    logs:
      include:
        match_type: regexp
        resource_attributes:
          - Key: host.name
            Value: prefix.*
  filter/regexp_record:
    logs:
      include:
        match_type: regexp
        record_attributes:
          - Key: record_attr
            Value: prefix_.*
  # Filter on severity text field
  filter/severity_text:
    logs:
      include:
        match_type: regexp
        severity_texts:
        - INFO[2-4]?
        - WARN[2-4]?
        - ERROR[2-4]?
  # Filter out logs below INFO (no DEBUG or TRACE level logs),
  # retaining logs with undefined severity
  filter/severity_number:
    logs:
      include:
        severity_number:
          min: "INFO"
          match_undefined: true
  filter/bodies:
    logs:
      include:
        match_type: regexp
        bodies:
        - ^IMPORTANT RECORD

```
</details>