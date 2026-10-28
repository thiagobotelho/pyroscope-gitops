# Profiling e persistência


Aplicações instrumentadas enviam profiles ao backend. O StatefulSet monolítico não é HA; PVC e metastore precisam de plano de recuperação. Compare labels de serviço/namespace no profile, trace e datasource. Fixe a imagem antes de promover e teste uma amostra sintética de CPU sem carga ilimitada.

## Fronteira com o mesh

O proxy gera métricas, logs e spans de rede; não gera automaticamente profiles de CPU da aplicação. Para correlacionar Tempo e Pyroscope, instrumente o processo e mantenha os mesmos identificadores de serviço. Ausência de profile deve ser investigada no emissor, na ingestão e no datasource, separadamente.
