# Como instrumentar aplicações com Pyroscope

Este guia é genérico para qualquer aplicação da stack. O `pyroscope-gitops`
instala o backend Pyroscope no cluster; ele não coleta profiles sozinho de uma
aplicação sem instrumentação.

## 1. O que este repositório entrega

- `Namespace` `pyroscope`.
- `StatefulSet` monolítico do Pyroscope.
- `Service` interno `pyroscope.pyroscope.svc:4040`.
- `PVC` para armazenar profiles.
- `ServiceMonitor` para métricas do próprio Pyroscope.
- `NetworkPolicy` permitindo acesso do Grafana, Prometheus Apps, Zabbix,
  namespace `observability` e namespaces com label `observability=enabled`.

Sem uma aplicação enviando profiles, o datasource no Grafana fica criado, mas o
Profiles Drilldown não terá flamegraphs.

## 2. Preparar o namespace da aplicação

Marque o namespace que enviará profiles:

```bash
APP_NAMESPACE=minha-aplicacao

oc create namespace "${APP_NAMESPACE}" --dry-run=client -o yaml | oc apply -f -
oc label namespace "${APP_NAMESPACE}" observability=enabled --overwrite
```

Essa label é usada pela `NetworkPolicy` do Pyroscope para permitir envio para a
porta `4040` sem depender de nomes fixos como `dev`, `aceite` ou `producao`.

## 3. Escolher o método de instrumentação

Use uma das abordagens abaixo:

| Método | Quando usar | Observações |
|---|---|---|
| SDK/agente da linguagem | Melhor controle por aplicação | Requer alterar imagem, comando ou código da aplicação. |
| Java Agent | JVM/Keycloak/Spring/Quarkus | Usa `-javaagent` e variáveis `PYROSCOPE_*`. |
| Grafana Alloy/eBPF | Descoberta automática e menos alteração de app | Exige revisão de RBAC/SCC/permissões e tuning por linguagem. |
| Span profiles | Correlação exata trace → profile | Exige OpenTelemetry + suporte da linguagem; atualmente CPU-only. |

Para CRC, comece por SDK/agente da linguagem. Alloy/eBPF é poderoso, mas mexe
em permissões de nó e pode aumentar consumo em single-node.

## 4. Labels recomendadas

Padronize os labels para que o Grafana consiga correlacionar profiles, traces,
logs e métricas:

| Label Pyroscope | Valor recomendado | Por quê |
|---|---|---|
| `service_name` | nome lógico do serviço | Mapeia com `service.name` do OpenTelemetry. |
| `service_namespace` | namespace do serviço | Mapeia com `service.namespace`. |
| `namespace` | namespace Kubernetes | Facilita filtro no Grafana. |
| `deployment_environment_name` | `desenvolvimento`, `aceite`, `producao` | Separa ambientes. |

Evite labels de alta cardinalidade, como `pod`, `container_id`, `user_id`,
`request_id` ou `trace_id`, salvo quando a documentação da linguagem pedir
explicitamente para span profiles.

## 5. Exemplo Java/JVM

Inclua o `pyroscope.jar` na imagem e adicione:

```bash
JAVA_OPTS_APPEND="-javaagent:/opt/app/pyroscope.jar"
PYROSCOPE_SERVER_ADDRESS="http://pyroscope.pyroscope.svc:4040"
PYROSCOPE_APPLICATION_NAME="minha-api"
PYROSCOPE_FORMAT="jfr"
PYROSCOPE_PROFILER_EVENT="itimer"
PYROSCOPE_LABELS="service_name=minha-api,service_namespace=minha-api,namespace=minha-api,deployment_environment_name=desenvolvimento"
```

Em containers restritos, comece com CPU profiling. Perfis de alloc/lock podem
gerar snapshots maiores e devem ser habilitados depois de validar consumo.

## 6. Exemplo Go

Use o SDK da linguagem e aponte para o Service interno:

```go
pyroscope.Start(pyroscope.Config{
  ApplicationName: "minha-api",
  ServerAddress:   "http://pyroscope.pyroscope.svc:4040",
  Tags: map[string]string{
    "service_name": "minha-api",
    "service_namespace": "minha-api",
    "namespace": "minha-api",
    "deployment_environment_name": "desenvolvimento",
  },
})
```

Habilite profile types adicionais apenas se fizer sentido para o runtime e para
o custo de coleta.

## 7. Exemplo Python/Node/.NET/Ruby/PHP/Rust

Siga o SDK oficial da linguagem e mantenha o mesmo padrão:

```text
server address: http://pyroscope.pyroscope.svc:4040
application name: <servico>
labels/tags: service_name, service_namespace, namespace, deployment_environment_name
```

O nome exato das variáveis ou funções muda por linguagem, mas o destino e os
labels devem permanecer consistentes.

## 8. Correlação com Tempo e Grafana

O `grafana-gitops` configura:

- datasource `Pyroscope`;
- datasource `Tempo` com `tracesToProfiles`;
- mapeamento de labels:
  - `service.name` → `service_name`;
  - `service.namespace` → `service_namespace`;
  - `k8s.namespace.name` → `namespace`.

Isso permite abrir profiles por serviço/namespace a partir de traces. Para
correlação exata por span, a aplicação precisa emitir o identificador de profile
no span, normalmente via integração Pyroscope + OpenTelemetry da linguagem.

## 9. Validar envio de profiles

Faça port-forward temporário:

```bash
oc -n pyroscope port-forward svc/pyroscope 4040:4040
```

Em outro terminal:

```bash
START="$(date -d '1 hour ago' +%s)000"
END="$(date +%s)000"

curl -fsS -H 'Content-Type: application/json' \
  -d "{\"start\":${START},\"end\":${END},\"name\":\"service_name\"}" \
  http://127.0.0.1:4040/querier.v1.QuerierService/LabelValues
```

Você deve ver o nome do serviço instrumentado. Para listar tipos:

```bash
curl -fsS -H 'Content-Type: application/json' \
  -d "{\"start\":${START},\"end\":${END},\"labelSelector\":\"{service_name=\\\"minha-api\\\"}\"}" \
  http://127.0.0.1:4040/querier.v1.QuerierService/ProfileTypes
```

## 10. Usar no Grafana Profiles Drilldown

1. Abra o Grafana.
2. Acesse Profiles Drilldown.
3. Selecione o datasource `Pyroscope`.
4. Filtre por `service_name`, `service_namespace` e ambiente.
5. Use flamegraph, tabela e comparação temporal para identificar hot paths.

Se nada aparecer:

- confirme que o pod da aplicação está rodando;
- confirme que o namespace tem `observability=enabled`;
- teste conexão para `pyroscope.pyroscope.svc:4040`;
- veja logs do agente/SDK;
- valide os labels no endpoint `LabelValues`.

## Referências oficiais

- Grafana Pyroscope: <https://grafana.com/docs/pyroscope/latest/>
- Configurar clientes: <https://grafana.com/docs/pyroscope/latest/configure-client/>
- Profile types e métodos de instrumentação: <https://grafana.com/docs/pyroscope/latest/configure-client/profile-types/>
- Java SDK/Agent: <https://grafana.com/docs/pyroscope/latest/configure-client/language-sdks/java/>
- Span profiles Java: <https://grafana.com/docs/pyroscope/latest/configure-client/trace-span-profiles/java-span-profiles/>
- Pyroscope Server API: <https://grafana.com/docs/pyroscope/latest/reference-server-api/>
