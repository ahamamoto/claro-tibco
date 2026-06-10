# Arquitetura Proposta — Mule 4 (Migração de SERVICE_AVALABILITY_MGMT)

> Diagramas Mermaid — renderizáveis nativamente no GitHub.  
> Referência: https://docs.mulesoft.com/general/ | TM Forum TMF760 Service Qualification

---

## 1. Visão Geral — API-Led Connectivity (3 Camadas)

```mermaid
flowchart TB
    classDef ext fill:#e8f8e8,stroke:#2E7D32,color:#000
    classDef exp fill:#e3f2fd,stroke:#1565C0,color:#000
    classDef proc fill:#f3e5f5,stroke:#6A1B9A,color:#000
    classDef sys fill:#fff8e1,stroke:#E65100,color:#000
    classDef mq fill:#fce4ec,stroke:#C62828,color:#000
    classDef infra fill:#f5f5f5,stroke:#616161,color:#000

    %% External consumers
    subgraph EXT["Consumidores"]
        C1["🖥️ Portal / CPQ\n(atual: SOAP)\n(futuro: REST JSON)"]
        C2["📊 GAIA\n(DB polling → CDC/Events)"]
        C3["🔧 Sistemas internos\n(outros serviços ESB)"]
    end

    %% Experience Layer
    subgraph EXP_LAYER["━━━━━━━━ Experience Layer ━━━━━━━━"]
        EXP["🔵 service-availability-exp-api\n───────────────────────────\nPOST   /serviceAvailabilityOrders\nGET    /serviceAvailabilityOrders/{id}\nDELETE /serviceAvailabilityOrders/{id}\nPOST   /serviceAvailabilityOrders/{id}/changeEvent\nGET    /serviceAvailabilityOrders?externalOrderId={id}\n───────────────────────────\n• OAS 3.0 / RAML (TMF760-like)\n• Client ID Enforcement (policy)\n• Rate Limiting (policy)\n• Spike Control (policy)\n• Request/Response logging"]
    end

    %% Process Layer
    subgraph PROC_LAYER["━━━━━━━━ Process Layer ━━━━━━━━"]
        PROC["🟣 service-availability-proc-api\n───────────────────────────\n• Validação de negócio\n• Enriquecimento de dados\n• Idempotência (Object Store)\n• Correlação X-Correlation-Id\n• Orquestração fanout assíncrono\n• Retry policy + Dead Letter Queue\n• Transformação EIMM ↔ JSON\n• Mapeamento ServiceProvisioningAction\n• Extração CharacteristicValue[]"]
    end

    %% Messaging
    subgraph MQ_LAYER["━━━━━━━━ Anypoint MQ ━━━━━━━━"]
        MQ_SSA["📬 svc-availability-ssa-queue"]
        MQ_GAIA["📬 svc-availability-gaia-queue"]
        MQ_SGPA["📬 svc-availability-sgpa-queue"]
        MQ_SF["📬 svc-availability-sf-queue"]
        MQ_NOTIFY["📬 svc-availability-notify-queue"]
        MQ_DLQ["☠️ svc-availability-dlq\n(Dead Letter Queue)"]
    end

    %% System Layer
    subgraph SYS_LAYER["━━━━━━━━ System Layer ━━━━━━━━"]
        SYS_SSA["🟠 ssa-sys-api\n─────────────\nDatabase Connector\nSQL Server\nSP: SSA_SP_PUB_1242_T\nSP: SSA_SP_PUB_1243\nSP: SSA_SP_SEL_CONSULTA_T\nRetry + Circuit Breaker"]
        SYS_GAIA["🟠 gaia-sys-api\n─────────────\nDatabase Connector (Oracle)\nScheduler Polling\nOutbox Pattern\nADB_L_DELIVERY_STATUS\nN → C / E"]
        SYS_SGPA["🟠 sgpa-sys-api\n─────────────\nHTTP Connector\nSOAP→REST Proxy\nBasic Auth\nEndpoint: /legadows/\nwebservices/serviceAvailabilityMgmt\nTimeout + Retry"]
        SYS_SF["🟠 salesforce-sys-api\n─────────────\nSalesforce Connector\nAPI v59+\nOAuth2 JWT Bearer\nBulk API support"]
        SYS_CPQ["🟠 cpq-sys-api\n─────────────\nHTTP Connector\nREST + OAuth2\nApigee Token Cache\nhttps://api-test.claro.com.br\n/qualification/v1/services/\navailabilitynotifications"]
    end

    %% Flows
    C1 -->|"REST/JSON\nor SOAP legacy"| EXP
    C2 -->|"Scheduler\nor CDC"| PROC
    C3 -->|"REST/JSON"| EXP

    EXP -->|"process call"| PROC

    PROC -->|"create/cancel"| MQ_SSA
    PROC -->|"create/cancel"| MQ_GAIA
    PROC -->|"create/cancel/notify"| MQ_SGPA
    PROC -->|"create/cancel/notify"| MQ_SF
    PROC -->|"notify"| MQ_NOTIFY
    PROC -.->|"falha / retry esgotado"| MQ_DLQ

    MQ_SSA -->|"consume"| SYS_SSA
    MQ_GAIA -->|"consume"| SYS_GAIA
    MQ_SGPA -->|"consume"| SYS_SGPA
    MQ_SF -->|"consume"| SYS_SF
    MQ_NOTIFY -->|"consume"| SYS_CPQ

    class C1,C2,C3 ext
    class EXP exp
    class PROC proc
    class SYS_SSA,SYS_GAIA,SYS_SGPA,SYS_SF,SYS_CPQ sys
    class MQ_SSA,MQ_GAIA,MQ_SGPA,MQ_SF,MQ_NOTIFY,MQ_DLQ mq
```

---

## 2. Fluxo de Criação — `POST /serviceAvailabilityOrders`

```mermaid
sequenceDiagram
    autonumber
    actor Client as Portal / CPQ
    participant EXP as service-availability-exp-api
    participant PROC as service-availability-proc-api
    participant OS as Object Store<br/>(Idempotência)
    participant MQ as Anypoint MQ
    participant SSA as ssa-sys-api<br/>(SQL Server)
    participant GAIA as gaia-sys-api<br/>(Oracle)
    participant SGPA as sgpa-sys-api<br/>(SOAP)
    participant SF as salesforce-sys-api

    Client->>EXP: POST /serviceAvailabilityOrders<br/>X-Correlation-Id: {uuid}<br/>Content-Type: application/json<br/>{serviceAvailabilityOrder}

    EXP->>EXP: Valida Client ID (policy)
    EXP->>EXP: Valida schema JSON (APIkit)
    EXP->>PROC: HTTP POST /internal/serviceAvailabilityOrders

    PROC->>OS: GET idempotencyKey = hash(externalOrderId)
    alt Já processado
        OS-->>PROC: 200 cached response
        PROC-->>EXP: 200 OK (idempotent)
        EXP-->>Client: 200 OK (duplicate request)
    else Novo request
        OS-->>PROC: null
        PROC->>PROC: DataWeave: JSON → EIMM canonical model
        PROC->>PROC: Map ServiceProvisioningAction → SERVICO_DES
        PROC->>PROC: Extract CharacteristicValue[] → flat fields

        par Fanout Assíncrono (Anypoint MQ)
            PROC->>MQ: Publish svc-availability-ssa-queue
            MQ->>SSA: Consume message
            SSA->>SSA: DB Connector: CALL SSA_SP_PUB_1242_T<br/>(~100 params)
            SSA-->>SSA: Retry on failure (3x)
        and
            PROC->>MQ: Publish svc-availability-gaia-queue
            MQ->>GAIA: Consume message
            GAIA->>GAIA: DB INSERT staging table<br/>ADB_L_DELIVERY_STATUS='N'
        and
            PROC->>MQ: Publish svc-availability-sgpa-queue
            MQ->>SGPA: Consume message
            SGPA->>SGPA: HTTP POST SOAP serviceAvailabilityMgmt
        and
            PROC->>MQ: Publish svc-availability-sf-queue
            MQ->>SF: Consume message
            SF->>SF: Salesforce Connector upsert
        end

        PROC->>OS: SET idempotencyKey = {orderId, status, timestamp}
        PROC-->>EXP: 202 Accepted<br/>{orderId, externalOrderId, status: "inProgress"}
        EXP-->>Client: 202 Accepted<br/>Location: /serviceAvailabilityOrders/{orderId}
    end
```

---

## 3. Padrão de Idempotência e Correlação

```mermaid
flowchart LR
    subgraph REQ["Request de Entrada"]
        R1["POST /serviceAvailabilityOrders\nX-Correlation-Id: abc-123\n{externalOrderId: 'SNOA-001'}"]
    end

    subgraph EXP2["Exp API (policy)"]
        P1{"X-Correlation-Id\npresente?"}
        P2["Gera UUID\nnovo"]
        P3["Propaga header\ncorrelationId"]
    end

    subgraph PROC2["Proc API"]
        I1["idempotencyKey =\nMD5(externalOrderId +\nsourceSystem)"]
        I2{"OS.get(key)\n≠ null?"}
        I3["Retorna resposta\ncacheada"]
        I4["Processa\nrequest"]
        I5["OS.store(key, response)\nTTL: 24h"]
    end

    subgraph VAR["Vars Mule 4"]
        V1["vars.correlationId\nvars.operationCode\nvars.sourceSystem\nvars.studyId\nvars.externalOrderId"]
    end

    R1 --> P1
    P1 -->|"Sim"| P3
    P1 -->|"Não"| P2 --> P3
    P3 --> I1
    I1 --> I2
    I2 -->|"Sim (duplicata)"| I3
    I2 -->|"Não (novo)"| I4
    I4 --> I5
    I4 --> VAR

    style REQ fill:#e3f2fd,stroke:#1565C0
    style EXP2 fill:#e8f4f8,stroke:#0288D1
    style PROC2 fill:#f3e5f5,stroke:#6A1B9A
    style VAR fill:#fff9c4,stroke:#F9A825
```

---

## 4. Tratamento de Erros — Mule 4 (DLQ + Redelivery)

```mermaid
flowchart TD
    START(["📩 Anypoint MQ\nMessage Received"]) --> LOCK["MQ Ack Mode: MANUAL\nAcquire lock"]

    LOCK --> PROC_F["Processar mensagem\n(System API call)"]

    PROC_F -->|"✅ Sucesso"| ACK["MQ.ack()\nRemove da fila"]
    ACK --> LOG_OK["Log: SUCCESS\n• correlationId\n• duration\n• systemResponse"]
    LOG_OK --> END_OK(["✅ End"])

    PROC_F -->|"❌ Erro técnico\n(timeout, connection)"| RETRY{"Tentativa\n≤ maxRedelivery\n(default: 3)?"}

    RETRY -->|"Sim — tentar novamente"| WAIT["Wait backoff\n(exponential: 1s, 2s, 4s)"]
    WAIT --> PROC_F

    RETRY -->|"Não — esgotou"| NACK["MQ.nack()\nDevolve para DLQ"]
    NACK --> DLQ["☠️ Dead Letter Queue\nsvc-availability-dlq"]

    DLQ --> PERSIST["Object Store\nPersistir payload original\nKey: correlationId\nTTL: 7 dias"]
    PERSIST --> ALERT["🔔 Alerta\nNotificar equipe operação\n(email / Slack / Ops Center)"]
    ALERT --> MANUAL["♻️ Reprocessamento Manual\nvia Ops Dashboard\nou API /retry/{correlationId}"]

    PROC_F -->|"⚠️ Erro funcional\n(regra de negócio)"| FUNC_ERR["Não retentar\nErro esperado"]
    FUNC_ERR --> NACK_FUNC["MQ.ack()\n(não recolocar na fila)"]
    NACK_FUNC --> LOG_FUNC["Log: BUSINESS_ERROR\n• errorCode\n• errorDescription\n• payload"]
    LOG_FUNC --> END_FUNC(["⚠️ End (funcional)"])

    style DLQ fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    style RETRY fill:#fff3e0,stroke:#FF9800
    style MANUAL fill:#e8f5e9,stroke:#2E7D32
```

---

## 5. Componentes Anypoint Platform

```mermaid
flowchart TB
    subgraph AP["☁️ Anypoint Platform"]

        subgraph DESIGN["Design Center"]
            OAS["OAS 3.0 Spec\nservice-availability-api"]
            DW["DataWeave Modules\n• mapToSSA.dwl\n• mapFromEIMM.dwl\n• charUtils.dwl"]
        end

        subgraph EXCHANGE["Anypoint Exchange"]
            API_ASSET["API Asset\nservice-availability-api\nv1.0.0"]
            REUSE["Reusable Fragments\n• error-handling-module\n• logging-module\n• security-module"]
        end

        subgraph MANAGER["API Manager"]
            POLICIES["Policies Aplicadas\n• Client ID Enforcement\n• Rate Limiting (100 req/min)\n• Spike Control\n• JWT Validation\n• Request Logging"]
        end

        subgraph RUNTIME["CloudHub 2.0 / RTF"]
            EXP_RT["service-availability-exp-api\nMule Runtime 4.6+\n0.2 vCores / 1 replica"]
            PROC_RT["service-availability-proc-api\nMule Runtime 4.6+\n0.5 vCores / 2 replicas"]
            SSA_RT["ssa-sys-api\nMule Runtime 4.6+\n0.2 vCores / 1 replica"]
            GAIA_RT["gaia-sys-api\nMule Runtime 4.6+\n0.2 vCores / 1 replica"]
            SGPA_RT["sgpa-sys-api\n0.1 vCores / 1 replica"]
            SF_RT["salesforce-sys-api\n0.1 vCores / 1 replica"]
            CPQ_RT["cpq-sys-api\n0.1 vCores / 1 replica"]
        end

        subgraph MQ_AP["Anypoint MQ"]
            QUEUES["Filas\n• svc-availability-ssa-queue\n• svc-availability-gaia-queue\n• svc-availability-sgpa-queue\n• svc-availability-sf-queue\n• svc-availability-notify-queue\n• svc-availability-dlq"]
        end

        subgraph SECRETS["Secrets Manager"]
            SEC["Credenciais\n• db.ssa.password\n• db.gaia.password\n• jms.password\n• sf.clientSecret\n• apigee.clientSecret\n• sgpa.password"]
        end

        subgraph MONITORING["Anypoint Monitoring"]
            MON["• Dashboards por API\n• Alertas de latência\n• Error rate tracking\n• Log correlation\n• Distributed tracing"]
        end
    end

    OAS --> API_ASSET
    API_ASSET --> POLICIES
    POLICIES --> EXP_RT
    EXP_RT --> PROC_RT
    PROC_RT --> QUEUES
    QUEUES --> SSA_RT
    QUEUES --> GAIA_RT
    QUEUES --> SGPA_RT
    QUEUES --> SF_RT
    QUEUES --> CPQ_RT
    SEC -.->|"inject at deploy"| EXP_RT
    SEC -.->|"inject at deploy"| PROC_RT
    SEC -.->|"inject at deploy"| SSA_RT
    REUSE -.->|"dependency"| EXP_RT
    REUSE -.->|"dependency"| PROC_RT
    EXP_RT --> MON
    PROC_RT --> MON

    style AP fill:#f8f9fa,stroke:#455A64,stroke-width:2px
    style DESIGN fill:#e3f2fd,stroke:#1565C0
    style EXCHANGE fill:#e8f5e9,stroke:#2E7D32
    style MANAGER fill:#fff3e0,stroke:#E65100
    style RUNTIME fill:#f3e5f5,stroke:#6A1B9A
    style MQ_AP fill:#fce4ec,stroke:#C62828
    style SECRETS fill:#fff9c4,stroke:#F57F17
    style MONITORING fill:#e0f7fa,stroke:#00695C
```

---

## 6. Estrutura de Projetos Mule 4

```mermaid
flowchart LR
    subgraph REPO["📁 Repositório Git"]
        direction TB

        subgraph EXP_PROJ["service-availability-exp-api/"]
            EA1["src/main/mule/\n├── service-availability-exp-api.xml\n├── global.xml\n└── error-handler.xml"]
            EA2["src/main/resources/\n├── api/\n│   └── service-availability-api.yaml\n├── app.yaml (dev/prod)\n└── secure-app.yaml"]
            EA3["src/test/munit/\n└── service-availability-exp-test.xml"]
        end

        subgraph PROC_PROJ["service-availability-proc-api/"]
            PA1["src/main/mule/\n├── service-availability-proc-api.xml\n├── create-flow.xml\n├── cancel-flow.xml\n├── notify-flow.xml\n├── query-flow.xml\n├── gaia-polling-flow.xml\n└── global.xml"]
            PA2["src/main/resources/dwl/\n├── mapToSSA.dwl\n├── mapFromEIMM.dwl\n├── charUtils.dwl\n└── addressUtils.dwl"]
        end

        subgraph SSA_PROJ["ssa-sys-api/"]
            SA1["src/main/mule/\n├── ssa-sys-api.xml\n├── create-flow.xml\n├── cancel-flow.xml\n└── query-flow.xml"]
        end

        subgraph COMMON["common-modules/ (Exchange)"]
            CM1["error-handling-module\nlogging-module\nsecurity-module\ncorrelation-module"]
        end
    end

    style REPO fill:#f5f5f5,stroke:#616161
    style EXP_PROJ fill:#e3f2fd,stroke:#1565C0
    style PROC_PROJ fill:#f3e5f5,stroke:#6A1B9A
    style SSA_PROJ fill:#fff8e1,stroke:#E65100
    style COMMON fill:#e8f5e9,stroke:#2E7D32
```

---

## 7. Mapeamento TIBCO BW → Mule 4

```mermaid
flowchart LR
    subgraph TIBCO["TIBCO BW (legado)"]
        T1["SOAP Listener\n(wsAdapterServiceCPQNC)"]
        T2["JMS Queue Receiver\n(TIBCO EMS)"]
        T3["JDBC Call Activity\n(SQL Server SP)"]
        T4["JDBC Update Activity\n(Oracle GAIA)"]
        T5["SOAP Invoke\n(SGPA Basic Auth)"]
        T6["Timer Event Source\n(Polling GAIA)"]
        T7["Checkpoint / Confirm"]
        T8["ERR_Handle Loop\n(repeat group)"]
        T9["GlobalVariables\n(defaultVars.substvar)"]
        T10["tib:render-xml()\nXSL Mapper Activity"]
    end

    subgraph MULE4["Mule 4 (proposto)"]
        M1["HTTP Listener\n(APIkit for SOAP)\nou REST Listener"]
        M2["Anypoint MQ Subscriber\n(MANUAL ack mode)"]
        M3["Database Connector\n(Stored Procedure)"]
        M4["Database Connector\n(Oracle INSERT/UPDATE)"]
        M5["HTTP Connector\n(SOAP envelope\nBasic Auth)"]
        M6["Scheduler\n(cron expression)"]
        M7["MQ.ack() / MQ.nack()\n+ Object Store"]
        M8["Until Successful\n+ On Error Continue\n+ Dead Letter Queue"]
        M9["Configuration Properties\n+ Secrets Manager"]
        M10["DataWeave 2.0\nwrite(payload, 'application/xml')"]
    end

    T1 -->|migra para| M1
    T2 -->|migra para| M2
    T3 -->|migra para| M3
    T4 -->|migra para| M4
    T5 -->|migra para| M5
    T6 -->|migra para| M6
    T7 -->|migra para| M7
    T8 -->|migra para| M8
    T9 -->|migra para| M9
    T10 -->|migra para| M10

    style TIBCO fill:#ffecb3,stroke:#F57F17,stroke-width:2px
    style MULE4 fill:#e8f5e9,stroke:#2E7D32,stroke-width:2px
```

---

## 8. Checklist de Migração

```mermaid
flowchart TD
    subgraph PHASE1["🔵 Fase 1 — Design (2 semanas)"]
        P1A["✅ Discovery TIBCO BW\n(docs/discovery.md)"]
        P1B["⬜ Definir contrato OAS 3.0\n(TMF760-like)"]
        P1C["⬜ Mapear todos os campos\nEIMM → JSON → SSA SP params"]
        P1D["⬜ Validar estrutura tabela GAIA\n(schema, índices, SPs)"]
        P1E["⬜ Obter WSDL do SGPA\n(não presente nos artefatos)"]
    end

    subgraph PHASE2["🟣 Fase 2 — Desenvolvimento (6 semanas)"]
        P2A["⬜ Criar API Spec (OAS/RAML)\nno Design Center"]
        P2B["⬜ Implementar ssa-sys-api\n(DB Connector + ~100 params)"]
        P2C["⬜ Implementar gaia-sys-api\n(Scheduler + DB Outbox)"]
        P2D["⬜ Implementar sgpa-sys-api\n(HTTP + SOAP)"]
        P2E["⬜ Implementar salesforce-sys-api\n(SF Connector v59+)"]
        P2F["⬜ Implementar cpq-sys-api\n(HTTP + OAuth2 Apigee)"]
        P2G["⬜ Implementar proc-api\n(DataWeave + Fanout + MQ)"]
        P2H["⬜ Implementar exp-api\n(APIkit + Policies)"]
        P2I["⬜ MUnit tests (>80% coverage)"]
    end

    subgraph PHASE3["🟠 Fase 3 — Validação (2 semanas)"]
        P3A["⬜ Testes de integração\ncom sistemas reais (sandbox)"]
        P3B["⬜ Testes de carga\n(JMeter / Gatling)"]
        P3C["⬜ Validar idempotência\ne reprocessamento DLQ"]
        P3D["⬜ Configurar monitoramento\n(Anypoint Monitoring + alertas)"]
    end

    subgraph PHASE4["🟢 Fase 4 — Go-Live (1 semana)"]
        P4A["⬜ Deploy em produção\n(CloudHub 2.0 / RTF)"]
        P4B["⬜ Execução paralela\nTIBCO + Mule (shadow mode)"]
        P4C["⬜ Cutover gradual\n(canary release)"]
        P4D["⬜ Descomissionar TIBCO BW"]
    end

    P1A --> P1B --> P1C --> P1D --> P1E
    P1E --> P2A --> P2B
    P2A --> P2C
    P2A --> P2D
    P2A --> P2E
    P2A --> P2F
    P2B & P2C & P2D & P2E & P2F --> P2G --> P2H --> P2I
    P2I --> P3A --> P3B --> P3C --> P3D
    P3D --> P4A --> P4B --> P4C --> P4D

    style PHASE1 fill:#e3f2fd,stroke:#1565C0,stroke-width:2px
    style PHASE2 fill:#f3e5f5,stroke:#6A1B9A,stroke-width:2px
    style PHASE3 fill:#fff8e1,stroke:#E65100,stroke-width:2px
    style PHASE4 fill:#e8f5e9,stroke:#2E7D32,stroke-width:2px
```

---

## Referências

- [MuleSoft Documentation](https://docs.mulesoft.com/general/)
- [TM Forum TMF760 — Service Qualification API](https://www.tmforum.org/resources/specification/tmf760-service-qualification-api-rest-specification-r19-0-0/)
- [TM Forum TMF641 — Service Ordering API](https://www.tmforum.org/resources/specification/tmf641-service-ordering-management-api-rest-specification/)
- [Mermaid Docs](https://mermaid.js.org/)
