# Gerador de QR Code no Kubernetes

Manifestos Kubernetes para publicar o **Gerador de QR Code SRJM** com NGINX
Gateway Fabric, entrega contínua via Argo CD e exposição pública pelo Cloudflare.

> Este repositório não provisiona o cluster nem a infraestrutura de rede. Ele
> contém os recursos Kubernetes da aplicação e de sua camada de entrada.

## Arquitetura

```text
Usuário
  │
  ▼
Cloudflare (DNS/proxy e proteção de borda)
  │
  ▼
LoadBalancer do NGINX Gateway Fabric
  │
  ├── geradorqrcode-srjm.uk ──► Service srjm-qrcode:3000
  │                                └── Deployment srjm-qrcode
  │
  └── argocd.geradorqrcode-srjm.uk ──► Service argocd-server:80

Git ──► Argo CD ──► reconciliação dos manifestos no cluster
```

As métricas seguem o fluxo abaixo. Um OpenTelemetry Collector central no
namespace `monitoring` descobre em todos os namespaces os Pods habilitados por
anotações, coleta seus endpoints Prometheus e publica as séries para o
Prometheus, que é a fonte de dados consultada pelo Grafana.

```text
Pods do cluster:/metrics ──► OpenTelemetry Collector ──► Prometheus ──► Grafana
```

O `Gateway` encerra o TLS dos dois hosts. O cert-manager solicita os
certificados ao Let's Encrypt por meio do desafio HTTP-01. A aplicação é
executada no namespace `qrcode-system`, com duas réplicas iniciais e atualização
do tipo `RollingUpdate`.

## Recursos

| Manifesto | Função |
| --- | --- |
| `deployment.yml` | Executa a imagem da aplicação na porta `3000`, com probes e limites de recursos |
| `service.yml` | Expõe os Pods internamente por um Service `ClusterIP` |
| `hpa.yml` | Escala de 2 a 5 réplicas quando CPU ou memória atingem 70% |
| `opentelemetry/configmap.yml` | Configura receivers, processors, exporters, health check e o pipeline de métricas |
| `opentelemetry/service-account.yml` | Identidade usada pelo Pod do Collector dentro do cluster |
| `opentelemetry/cluster-role.yml` | Autoriza somente leitura e descoberta de Pods em todos os namespaces |
| `opentelemetry/cluster-role-binding.yml` | Vincula a permissão cluster-wide à identidade do Collector |
| `opentelemetry/deployment.yml` | Executa o OpenTelemetry Collector com probes, recursos e segurança |
| `opentelemetry/service.yml` | Publica OTLP, métricas Prometheus, telemetria interna e health check no cluster |
| `serviceMonitor.yml` | Faz o Prometheus coletar as métricas e a telemetria interna do Collector |
| `grafana/otel-dashboard.yml` | Provisiona o dashboard `SRJM QR Code / OpenTelemetry` no Grafana |
| `grafana/datasources.yml` | Mantém o Prometheus e provisiona o Tempo como datasource do Grafana |
| `tempo/configmap.yml` | Configura recepção OTLP, retenção e armazenamento local do Tempo |
| `tempo/persistent-volume-claim.yml` | Reserva 10 Gi persistentes para os traces |
| `tempo/deployment.yml` | Executa o backend Grafana Tempo em modo monolítico |
| `tempo/service.yml` | Expõe a API de consulta e os receivers OTLP do Tempo no cluster |
| `gateway.yml` | Define os listeners HTTP/HTTPS da aplicação e do Argo CD |
| `http-route.yml` | Encaminha o domínio principal para a aplicação |
| `argocd-route/httproute-argocd.yml` | Encaminha o subdomínio do Argo CD para `argocd-server` |
| `production_issuer-Gateway.yml` | Configura o emissor de produção do Let's Encrypt |

## OpenTelemetry e Grafana

Aplique os recursos (ou inclua-os na aplicação do Argo CD):

```bash
kubectl apply -f opentelemetry/service-account.yml
kubectl apply -f opentelemetry/cluster-role.yml
kubectl apply -f opentelemetry/cluster-role-binding.yml
kubectl apply -f opentelemetry/configmap.yml
kubectl apply -f opentelemetry/deployment.yml
kubectl apply -f opentelemetry/service.yml
kubectl apply -f serviceMonitor.yml
kubectl apply -f grafana/otel-dashboard.yml
```

O `ServiceMonitor` exige o Prometheus Operator. O dashboard é descoberto
automaticamente quando o sidecar de dashboards do Grafana está habilitado para
buscar ConfigMaps com o label `grafana_dashboard=1` em todos os namespaces (ou
no namespace `monitoring`).

Para habilitar qualquer aplicação do cluster que já exponha métricas, adicione
estas anotações ao `spec.template.metadata` do Deployment (ajustando porta e
caminho):

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: /metrics
  prometheus.io/port: "3000"
```

Aplicações instrumentadas com um SDK OpenTelemetry também podem enviar métricas
por OTLP para `otel-collector.monitoring.svc.cluster.local:4317` (gRPC) ou
`http://otel-collector.monitoring.svc.cluster.local:4318` (HTTP).

Os traces seguem um pipeline separado e ficam retidos por 24 horas:

```text
Aplicação Node.js ──OTLP──► Collector ──OTLP──► Tempo ──consulta──► Grafana
```

A aplicação usa auto-instrumentação Node.js, propagação W3C e amostragem de 10%.
O datasource `tempo` pode ser consultado em **Grafana → Explore**.

Para validar o pipeline:

```bash
kubectl -n monitoring rollout status deployment/otel-collector
kubectl -n monitoring logs deployment/otel-collector
kubectl -n monitoring port-forward service/otel-collector 8889:8889
curl http://localhost:8889/metrics
```
