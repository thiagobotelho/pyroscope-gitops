# Pyroscope GitOps

Backend GitOps do Grafana Pyroscope para OpenShift Local/CRC, integrado ao
Grafana Profiles Drilldown e preparado para coleta de continuous profiling.

Este repositório publica o Pyroscope em modo monolítico, sem Route pública e sem
versionar credenciais.

## Arquitetura

```mermaid
flowchart LR
    App[Aplicações com profiling] --> Pyro[Pyroscope]
    Alloy[Grafana Alloy opcional] --> Pyro
    Pyro --> PVC[(PVC profiles)]
    Grafana[Grafana] --> Pyro
    Tempo[Tempo] -. tracesToProfiles .-> Grafana
    Pyro --> Prom[Prometheus Apps]
```

O perfil CRC usa `-target=all`, conforme o modo monolítico documentado pelo
Grafana Pyroscope. Esse modo é simples e adequado para laboratório; para HA ou
retenção formal, use o Helm chart oficial em modo microservices/object storage.

## Deploy

```bash
oc apply -k overlays/desenvolvimento
oc -n pyroscope get statefulset,pods,svc,pvc,servicemonitor
```

Validação declarativa:

```bash
oc kustomize overlays/desenvolvimento >/tmp/pyroscope-dev.yaml
oc apply --dry-run=server -k overlays/desenvolvimento
```

## Integração com Grafana

O `grafana-gitops` provisiona:

- datasource `pyroscope` com URL `http://pyroscope.pyroscope.svc:4040`;
- `tracesToProfiles` no datasource Tempo;
- pasta `Profiles` para dashboards futuros.

Sem aplicações instrumentadas, o datasource fica saudável, mas Profiles
Drilldown não mostrará flamegraphs. Para correlação trace → profile, as spans
precisam carregar labels compatíveis e, idealmente, `pyroscope.profile.id`.

## Segurança

- Sem Route pública.
- NetworkPolicy libera somente Grafana, Prometheus Apps, Zabbix, namespace
  `observability` e tráfego interno do próprio Pyroscope.
- ServiceAccount com Role local apenas para descoberta de endpoints/pods usada
  pelo backend.
- Sem Secrets obrigatórios neste perfil.

## Ambientes

Veja [docs/AMBIENTES.md](docs/AMBIENTES.md).

## Referências

- [Grafana Pyroscope](https://grafana.com/docs/pyroscope/latest/)
- [Deploy com Helm](https://grafana.com/docs/pyroscope/latest/deploy-kubernetes/helm/)
- [Deployment modes](https://grafana.com/docs/pyroscope/latest/reference-pyroscope-architecture/deployment-modes/)
- [Grafana Pyroscope datasource](https://grafana.com/docs/grafana/latest/datasources/pyroscope/configure-pyroscope-data-source/)
