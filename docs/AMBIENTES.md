# Ambientes

Este repositório usa `base/` e `overlays/{desenvolvimento,aceite,producao}`.

- `desenvolvimento`: perfil CRC, Pyroscope monolítico, PVC de 10 GiB e sem
  Route pública. O bootstrap Raft usa peer estático do próprio pod para evitar
  duplicidade de peers na resolução DNS SRV do CRC.
- `aceite`: mesmo desenho, PVC de 25 GiB.
- `producao`: PVC maior e recursos mais altos, mas ainda monolítico; produção
  real deve avaliar Helm/microservices/object storage.

Validação:

```bash
oc kustomize overlays/desenvolvimento >/tmp/pyroscope-dev.yaml
oc kustomize overlays/aceite >/tmp/pyroscope-aceite.yaml
oc kustomize overlays/producao >/tmp/pyroscope-prod.yaml
oc apply --dry-run=server -k overlays/desenvolvimento
```

Profiles:

- O backend recebe profiles na API HTTP do Pyroscope.
- O datasource Grafana deve usar `http://pyroscope.pyroscope.svc:4040`.
- Para eBPF/profiling automático em OpenShift, use Grafana Alloy como componente
  opcional e revise SCC/RBAC antes de habilitar.
