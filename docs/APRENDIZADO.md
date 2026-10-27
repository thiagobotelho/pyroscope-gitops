# Profiling e persistência

## O que observar

Aplicações instrumentadas enviam profiles ao backend. O StatefulSet monolítico não é HA; PVC e metastore precisam de plano de recuperação. Compare labels de serviço/namespace no profile, trace e datasource. Fixe a imagem antes de promover e teste uma amostra sintética de CPU sem carga ilimitada.

## Roteiro de leitura e validação

1. Leia o README e identifique o componente que recebe a configuração.
2. Confira namespaces, seletores, portas, identidades e dependências nos exemplos.
3. Ajuste apenas os campos necessários em um ambiente isolado; não use credenciais de produção.
4. Valide sintaxe e contratos locais antes de aplicar. Registre o resultado real separado deste guia.
5. Execute um caso permitido e um caso negado, conferindo a causa da resposta.
6. Defina reversão por recurso/arquivo e preserve dados persistentes antes de mudanças.

Os comentários dos manifests explicam escolhas; não comprovam disponibilidade do componente. Exemplos fora de entradas Kustomize não são implantados automaticamente.
