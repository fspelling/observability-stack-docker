# observability-stack-docker

Stack de observabilidade com Docker Compose para monitoramento de métricas, logs e alertas de uma API de exemplo. Combina **Prometheus** (métricas), **Alertmanager** (alertas), **Loki + Promtail** (logs) em um ambiente pronto para subir localmente.

## Arquitetura

| Serviço | Imagem | Porta | Função |
|---|---|---|---|
| `api` | `fspelling/vertical-slice-api:latest` | 5193 | API de exemplo que expõe métricas no formato Prometheus |
| `prometheus` | `prom/prometheus:v2.24.1` | 9090 | Coleta e armazena as métricas da `api` |
| `alertmanager` | `prom/alertmanager:v0.21.0` | 9093 | Recebe alertas do Prometheus e os roteia via webhook |
| `loki` | `grafana/loki:2.0.0` | 3100 | Armazena os logs agregados pelo Promtail |
| `promtail` | `grafana/promtail:2.0.0` | 9080 | Coleta logs dos containers Docker e do host, enviando para o Loki |

Todos os serviços compartilham a rede `observability` (bridge).

Fluxo resumido:

```
api ──(scrape)──> prometheus ──(alertas)──> alertmanager ──(webhook)──> notificação externa
containers/host ──(tail)──> promtail ──(push)──> loki
```

> **Nota:** não há um serviço de visualização (ex.: Grafana) no `compose.yaml` atual. Métricas podem ser consultadas na UI do próprio Prometheus (`:9090`) e logs via API do Loki (`:3100`). Veja [Melhorias sugeridas](#melhorias-sugeridas).

## Pré-requisitos

- Docker
- Docker Compose (plugin `docker compose` ou `docker-compose`)

## Como executar

```bash
git clone https://github.com/fspelling/observability-stack-docker.git
cd observability-stack-docker
docker compose up -d
```

Para acompanhar os logs dos serviços:

```bash
docker compose logs -f
```

Para derrubar o ambiente:

```bash
docker compose down
```

## Acessando os serviços

| Serviço | URL |
|---|---|
| API | http://localhost:5193 |
| Prometheus | http://localhost:9090 |
| Alertmanager | http://localhost:9093 |
| Loki (API) | http://localhost:3100 |
| Promtail | http://localhost:9080 |

## Métricas e alertas

O Prometheus faz scrape de dois targets, definidos em [`prometheus/prometheus.yaml`](prometheus/prometheus.yaml):

- `prometheus` (self-monitoring, `localhost:9090`)
- `api` (`api:5193`, intervalo de 5s)

A regra de alerta em [`prometheus/alert.rules.yaml`](prometheus/alert.rules.yaml) dispara `WebApiMultiploAcessosAlert` quando a taxa de requisições (`http_requests_received_total`) ultrapassa 1 req/s por 30 segundos consecutivos.

Os alertas são roteados pelo Alertmanager ([`prometheus/alertmanager.yaml`](prometheus/alertmanager.yaml)) para um webhook de teste (webhook.site). **Troque essa URL antes de usar em qualquer ambiente real** — veja [Configuração](#configuração).

## Logs

O Promtail ([`promtail/config.yaml`](promtail/config.yaml)) coleta:

- Logs do host em `/var/log/*log` (job `varlogs`)
- Logs de containers Docker em `/var/lib/docker/containers/*/*-json.log` (job `apilogs`), com o estágio `docker` para parsing automático

Os logs são enviados ao Loki via `http://loki:3100/loki/api/v1/push`.

## Configuração

Antes de usar em outro contexto que não seja teste local, ajuste:

- **Webhook do Alertmanager**: substitua a URL `https://webhook.site/...` em `prometheus/alertmanager.yaml` pelo endpoint real (Slack, e-mail, PagerDuty, etc.).
- **Imagem da API**: `fspelling/vertical-slice-api:latest` — troque pela sua própria imagem/API se for reaproveitar a stack.
- **Volumes do Promtail**: em Windows/WSL, `/var/lib/docker/containers` e `/var/log` podem exigir ajuste de caminho conforme o driver do Docker Desktop.

## Melhorias sugeridas

- Adicionar um serviço **Grafana** ao `compose.yaml` com Prometheus e Loki já provisionados como datasources, para visualização de métricas e logs.
- Fixar (pin) as imagens em versões mais recentes — `loki`/`promtail` 2.0.0 e `alertmanager` 0.21.0 são bastante antigas.
- Adicionar `healthcheck` aos serviços para facilitar orquestração e depende de `condition: service_healthy`.
- Externalizar a URL do webhook do Alertmanager via variável de ambiente.

## Licença

Não especificada.
