# Local Setup of Monitoring Stack (otel-loki-prometheus-grafana) 

 

 

## Introduction 

We will setup a docker compose, that will create the supporting infrastructure to test the traces. 

 

## This will deploy 

- Loki – logs agregattion 

- Promethus – metrics 

- Grafana – dashboard  

- Jaegar - tracing 

- Tempo / Otel 

 

 

## Docker compose 
----------------------------------------------- 



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

  

    promtail: 

    image: grafana/promtail:2.2.0 

    command: -config.file=/etc/promtail/promtail-local.yaml 

    volumes: 

      - ./etc/promtail-local.yaml:/etc/promtail/promtail-local.yaml 

      - ./data/logs:/app/logs 

    depends_on: 

      - loki 

 
  

  prometheus: 

    image: prom/prometheus:latest 

    volumes: 

      - ./etc/prometheus.yaml:/etc/prometheus.yaml 

    entrypoint: 

      - /bin/prometheus 

      - --config.file=/etc/prometheus.yaml 

    ports: 

      - "9090:9090" 


  

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
 

 

 

----------------------------------------------- 

 

 

In this case we deploy a simple api to check the logs 

 

View the log and trace in Grafana --> http://localhost:3000/explore

 

 

 

Also Loki and Tempo interact with the trace id 

 

Docker ps 


```
caf0427ba517   grafana/grafana:7.4.0-ubuntu                       "/run.sh"                26 minutes ago   Up 26 minutes   0.0.0.0:3000->3000/tcp                                                                                 bootstrap-opentelemetry_grafana_1 

dc37b9ec73c0   grafana/promtail:2.2.0                             "/usr/bin/promtail -…"   27 minutes ago   Up 26 minutes                                                                                                          bootstrap-opentelemetry_promtail_1 

9d9a3af1e179   prom/prometheus:latest                             "/bin/prometheus --c…"   27 minutes ago   Up 26 minutes   0.0.0.0:9090->9090/tcp                                                                                 bootstrap-opentelemetry_prometheus_1 

9ee59cc905da   grafana/tempo-query:0.7.0                          "/go/bin/query-linux…"   27 minutes ago   Up 27 minutes   0.0.0.0:16686->16686/tcp                                                                               bootstrap-opentelemetry_tempo-query_1 

fffbbba81704   grafana/loki:2.2.0                                 "/usr/bin/loki -conf…"   27 minutes ago   Up 27 minutes   0.0.0.0:3101->3100/tcp                                                                                 bootstrap-opentelemetry_loki_1 

fabiopalumbo@Fabios-MBP  ~/switfly/bootstrap-opentelemetry   main 
```
 

 

 