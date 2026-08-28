# Stack Health Check — 2026-08-28T11:13:38Z

```
NAME                   IMAGE                                          COMMAND                  SERVICE          CREATED         STATUS         PORTS
atlas-alertmanager     prom/alertmanager:v0.27.0                      "/bin/alertmanager -…"   alertmanager     4 minutes ago   Up 4 minutes   0.0.0.0:9093->9093/tcp, [::]:9093->9093/tcp
atlas-grafana          grafana/grafana:11.2.0                         "/run.sh"                grafana          4 minutes ago   Up 3 minutes   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
atlas-loki             grafana/loki:3.1.1                             "/usr/bin/loki -conf…"   loki             4 minutes ago   Up 4 minutes   0.0.0.0:3100->3100/tcp, [::]:3100->3100/tcp
atlas-otel-collector   otel/opentelemetry-collector-contrib:0.106.0   "/otelcol-contrib --…"   otel-collector   4 minutes ago   Up 3 minutes   0.0.0.0:4317-4318->4317-4318/tcp, [::]:4317-4318->4317-4318/tcp, 55678-55679/tcp
atlas-prometheus       prom/prometheus:v2.55.1                        "/bin/prometheus --c…"   prometheus       4 minutes ago   Up 4 minutes   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
atlas-tempo            grafana/tempo:2.6.1                            "/tempo -config.file…"   tempo            4 minutes ago   Up 4 minutes   0.0.0.0:3200->3200/tcp, [::]:3200->3200/tcp

Prometheus: 200
Grafana: 200
Loki: 200
Tempo: 200
Alertmanager: 200
OTel Collector: 405
```
