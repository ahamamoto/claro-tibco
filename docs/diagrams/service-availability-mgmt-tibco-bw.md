# SERVICE_AVAILABILITY_MGMT — Arquitetura TIBCO BW

```mermaid
flowchart TB
    subgraph Entrada["🔵 Canais de Entrada"]
        CPQ["CPQ / Portal\n(SOAP/HTTP)"]
        GAIA_SRC["GAIA\n(Oracle DB staging)"]
    end

    subgraph BW["⚙️ SERVICE_AVALABILITY_MGMT — TIBCO BW"]
        direction TB
        WS["wsAdapter*.process\n(SOAP Listener)"]
        POLL["GAIA_Requests_Polling_In\n(Timer + JDBC)"]
        EMS["TIBCO EMS\ntcp://10.2.11.10:7222\n14 filas EMBRATEL.SERVICEAVALABILITY.*"]

        subgraph FLOWS["Processos de Orquestração"]
            F_CREATE["create*Out\n(9 processos)"]
            F_CANCEL["cancel*Out\n(6 processos)"]
            F_NOTIFY["notify*Out"]
            F_QUERY["query*Out"]
        end

        subgraph INFRA["Infraestrutura Transversal"]
            LOG["Log Library\n(JDBC + JMS + DocumentDB)"]
            ERR["Erro Library\n(Hospital + Redelivery)"]
        end
    end

    subgraph Backend["🟢 Sistemas Backend"]
        SSA["SSA\n(SQL Server)\nSP: SSA_SP_PUB_1242_T\nSSA_SP_PUB_1243"]
        GAIA_DST["GAIA\n(Oracle staging)\nDB Outbox pattern"]
        SGPA["SGPA\n(SOAP/HTTP)\nhttp://10.0.197.81:8080\n/legadows/webservices/\nserviceAvailabilityMgmt"]
        SF["Salesforce\n(SOAP Partner v30)"]
        CPQ_DST["CPQ / CDIGP / MPIS\n(REST via Apigee)\nhttps://api-test.claro.com.br\n/qualification/v1/services/\navailabilitynotifications"]
    end

    CPQ -->|"SOAP\ncreate/cancel/notify/query"| WS
    GAIA_SRC -->|"SELECT ADB_L_DELIVERY_STATUS='N'"| POLL

    WS --> EMS
    POLL --> EMS

    EMS --> F_CREATE
    EMS --> F_CANCEL
    EMS --> F_NOTIFY
    EMS --> F_QUERY

    F_CREATE -->|"JDBC SP"| SSA
    F_CREATE -->|"JDBC Oracle"| GAIA_DST
    F_CREATE -->|"SOAP Basic Auth"| SGPA
    F_CREATE -->|"SOAP"| SF
    F_CREATE -->|"REST OAuth2"| CPQ_DST

    F_CANCEL -->|"JDBC SP"| SSA
    F_CANCEL -->|"JDBC Oracle"| GAIA_DST
    F_CANCEL -->|"SOAP"| SGPA
    F_CANCEL -->|"SOAP"| SF

    F_NOTIFY -->|"SOAP"| SGPA
    F_NOTIFY -->|"SOAP"| SF

    F_QUERY -->|"JDBC SP SELECT"| SSA

    FLOWS --> LOG
    FLOWS --> ERR

    style BW fill:#e8f4f8,stroke:#2196F3,stroke-width:2px
    style Entrada fill:#e8f8e8,stroke:#4CAF50,stroke-width:2px
    style Backend fill:#fff8e1,stroke:#FF9800,stroke-width:2px
    style INFRA fill:#fce4ec,stroke:#E91E63,stroke-width:1px,stroke-dasharray:5 5
```

## Legenda

| Símbolo | Descrição |
|---------|-----------|
| 🔵 | Canais de entrada (CPQ, GAIA) |
| ⚙️ | Motor de integração (TIBCO BW) |
| 🟢 | Sistemas backend |

## Componentes Principais

### Entrada
- **CPQ / Portal**: Interface SOAP para criar/cancelar/notificar/consultar ordens
- **GAIA**: Polling periódico em tabela Oracle de staging

### TIBCO BW
- **SOAP Listener** (`wsAdapter*.process`): Recebe requisições SOAP do CPQ
- **GAIA Polling** (`GAIA_Requests_Polling_In`): Timer + JDBC para monitorar staging
- **TIBCO EMS**: Message broker com 14 filas dedicadas (EMBRATEL.SERVICEAVALABILITY.*)
- **Orquestração**: 4 fluxos principais (create, cancel, notify, query)
- **Infraestrutura**: Log Library + Error Library (padrão Hospital + Redelivery)

### Backend
- **SSA**: SQL Server com stored procedures (1242 = create, 1243 = cancel)
- **GAIA**: Oracle staging com padrão Outbox
- **SGPA**: SOAP legacy com autenticação Basic Auth
- **Salesforce**: SOAP Partner API v30
- **CPQ/CDIGP/MPIS**: REST via Apigee com OAuth2

## Fluxos de Dados

| Operação | Destinos |
|----------|----------|
| `create` | SSA, GAIA, SGPA, SF, CPQ (9 processos) |
| `cancel` | SSA, GAIA, SGPA, SF (6 processos) |
| `notify` | SGPA, SF |
| `query` | SSA (SELECT via SP) |
