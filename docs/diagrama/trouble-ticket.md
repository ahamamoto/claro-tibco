
## Arquitetura Atual — TIBCO BusinessWorks 5.x

```mermaid
flowchart LR
    classDef inbound fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef bw fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef outbound fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef crosscut fill:#f3e8ff,stroke:#9333ea,color:#581c87
    classDef db fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    subgraph CALLERS["☎ Sistemas Chamadores"]
        direction TB
        EXT1["CRM / Portais\nSistemas Externos"]
        EXT2["SIATE\n(inbound)"]
        EXT3["CRM-IN\n(ADB)"]
    end

    subgraph INBOUND["⬇ Inbound"]
        direction TB
        WS["🌐 SOAP Web Service\nTroubleTicket.wsdl\n─────────────\nCreate / Update / Close\nConsultar / Delivery"]
        JMS_IN["📨 JMS TIBCO EMS\ntcp://uxeaitst:7600\n─────────────\nQUEUES/CREATE\nQUEUES/DELIVERY\nQUEUES/TROUBLETICKET"]
    end

    subgraph BW["⚙ TIBCO BW 5.x — Camada de Orquestração"]
        direction TB
        subgraph PROC["Processos de Negócio"]
            direction LR
            IN_P["*In\n(Recebe &\nValida)"]
            ORQ_P["*Orq\n(Roteia &\nTransforma)"]
            OUT_P["*Out\n(Envia ao\nDestino)"]
            IN_P --> ORQ_P --> OUT_P
        end
        DEPARA["🗂 DePara Engine\nOracle DB\nIdServicoNegocio\n+ SistemaOrigem\n→ SISTEMA_DESTINO[]"]
        ORQ_P <-.->|"lookup\nroteamento"| DEPARA
    end

    subgraph CROSSCUT["🔧 Serviços Transversais"]
        direction TB
        ERR["❌ Erro Library\nJMS + Oracle\nERR_Handle\nRetry / Suspend\nKill Engine"]
        LOG["📋 Log Library\nJMS + Oracle\nSet_Log\nNível por ServiçoNeg."]
        AUDIT["📝 Audit\nOracle\nAudit_Lista"]
    end

    subgraph TARGETS["➡ Sistemas Destino (Outbound)"]
        direction TB
        SF["☁ Salesforce\nSOAP Enterprise\n+ OAuth / SSL"]
        NETF["📡 NET Serviços\nSOAP HTTPS\nServiceProblemTicket"]
        SIR["🔧 SIR\nSOAP"]
        VSMS["📟 VSMS\nSOAP"]
        PSAC["🏢 PSAC\nSOAP + JDBC Oracle"]
        SIATE_OUT["🖥 SIATE\nSOAP\nOpCreate/Set/GetList"]
        TMX["📦 TMX\nJMS IBM MQ"]
        GRC["🔑 GRC\nSOAP"]
        CRM_DB["🗄 CRM\nJDBC Oracle"]
    end

    EXT1 -->|"HTTP\nSOAP"| WS
    EXT2 -->|"HTTP\nSOAP"| WS
    EXT3 -->|"ADB\nOracle"| JMS_IN
    WS --> IN_P
    JMS_IN --> IN_P

    OUT_P -->|"SOAP/HTTPS\ncert SF"| SF
    OUT_P -->|"SOAP/HTTPS\nNETAuth.id"| NETF
    OUT_P -->|"SOAP"| SIR
    OUT_P -->|"SOAP"| VSMS
    OUT_P -->|"SOAP+JDBC"| PSAC
    OUT_P -->|"SOAP\nSSL"| SIATE_OUT
    OUT_P -->|"JMS\nIBM MQ"| TMX
    OUT_P -->|"SOAP"| GRC
    OUT_P -->|"JDBC\nOracle"| CRM_DB

    ORQ_P -.->|"erros"| ERR
    ORQ_P -.->|"logs"| LOG
    ORQ_P -.->|"auditoria"| AUDIT

    class EXT1,EXT2,EXT3 inbound
    class WS,JMS_IN inbound
    class IN_P,ORQ_P,OUT_P,DEPARA bw
    class ERR,LOG,AUDIT crosscut
    class SF,NETF,SIR,VSMS,PSAC,SIATE_OUT,TMX,GRC,CRM_DB outbound
```

---

## Arquitetura Proposta — Mule 4 (API-led Connectivity + TMF 621)

```mermaid
flowchart TD
    classDef exp fill:#dbeafe,stroke:#2563eb,color:#1e3a8a,font-weight:bold
    classDef prc fill:#fef3c7,stroke:#d97706,color:#78350f,font-weight:bold
    classDef sys fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef infra fill:#f3e8ff,stroke:#9333ea,color:#581c87
    classDef ext fill:#f1f5f9,stroke:#64748b,color:#1e293b
    classDef mq fill:#fce7f3,stroke:#db2777,color:#831843

    CALLERS["☎ Callers\nCRM / Portais / Sistemas Externos\nLegado SOAP (via facade)"]

    subgraph EXL["🌐 Experience Layer"]
        EXP["trouble-ticket-exp-api\n─────────────────────\nREST / JSON — OAS 3.0\nTMF 621 compliant\nHTTP Listener\nAPIkit Router\n─────────────────────\nPOST   /troubleTickets\nGET    /troubleTickets/{id}\nPATCH  /troubleTickets/{id}\nDELETE /troubleTickets/{id}\nPOST   /hub (webhook)"]
    end

    subgraph PRC["⚙ Process Layer"]
        direction TB
        PROC["trouble-ticket-prc-api\nOrquestração + DePara Routing\nDataWeave Transforms\nError Handler + Until-Successful"]
        MQ_CONS["Anypoint MQ Consumer\n(async events)"]
    end

    subgraph SYS["🔌 System Layer"]
        direction LR
        SF_S["salesforce-sys-api\nSalesforce Connector"]
        NETF_S["netf-sys-api\nWSC — ServiceProblemTicket"]
        SIATE_S["siate-sys-api\nWSC — SIATE"]
        SIR_S["sir-sys-api\nWSC — SIR"]
        VSMS_S["vsms-sys-api\nWSC — VSMS"]
        PSAC_S["psac-sys-api\nWSC + DB Connector"]
        TMX_S["tmx-sys-api\nIBM MQ Connector"]
        GRC_S["grc-sys-api\nWSC — GRC"]
        CRM_S["crm-sys-api\nDB Connector — Oracle"]
        DEP_S["depara-sys-api\nDB Connector + ObjectStore"]
    end

    subgraph PLATFORM["☁ Anypoint Platform — Infraestrutura"]
        direction LR
        AMQ["Anypoint MQ\n(async messaging)"]
        OS["Object Store v2\n(cache DePara\ncontador retry)"]
        MON["Anypoint Monitoring\n+ Logging (Logger)"]
        VAULT["Secure Properties\n(credenciais)"]
        TLS_CTX["TLS Context Global\n(certificados SSL)"]
    end

    subgraph EXT_SYS["🖥 Sistemas Externos"]
        direction TB
        SF_EXT["Salesforce"]
        NETF_EXT["NET Servicos"]
        SIATE_EXT["SIATE"]
        SIR_EXT["SIR"]
        VSMS_EXT["VSMS"]
        PSAC_EXT["PSAC"]
        TMX_EXT["TMX / IBM MQ"]
        GRC_EXT["GRC"]
        CRM_EXT["CRM Oracle"]
        DEP_EXT["DePara Oracle"]
    end

    CALLERS -->|"REST/JSON\nHTTPS"| EXP
    EXP -->|"Flow Ref"| PROC
    AMQ -->|"subscribe"| MQ_CONS
    MQ_CONS -->|"Flow Ref"| PROC

    PROC -->|"Flow Ref"| SF_S
    PROC -->|"Flow Ref"| NETF_S
    PROC -->|"Flow Ref"| SIATE_S
    PROC -->|"Flow Ref"| SIR_S
    PROC -->|"Flow Ref"| VSMS_S
    PROC -->|"Flow Ref"| PSAC_S
    PROC -->|"Flow Ref"| TMX_S
    PROC -->|"Flow Ref"| GRC_S
    PROC -->|"Flow Ref"| CRM_S
    PROC <-->|"routing lookup"| DEP_S

    PROC <-.->|"cache\nretry state"| OS
    PROC -.->|"logs\nmetrics"| MON

    SF_S --> SF_EXT
    NETF_S --> NETF_EXT
    SIATE_S --> SIATE_EXT
    SIR_S --> SIR_EXT
    VSMS_S --> VSMS_EXT
    PSAC_S --> PSAC_EXT
    TMX_S --> TMX_EXT
    GRC_S --> GRC_EXT
    CRM_S --> CRM_EXT
    DEP_S --> DEP_EXT

    class CALLERS ext
    class EXP exp
    class PROC,MQ_CONS prc
    class SF_S,NETF_S,SIATE_S,SIR_S,VSMS_S,PSAC_S,TMX_S,GRC_S,CRM_S,DEP_S sys
    class AMQ,OS,MON,VAULT,TLS_CTX infra
    class SF_EXT,NETF_EXT,SIATE_EXT,SIR_EXT,VSMS_EXT,PSAC_EXT,TMX_EXT,GRC_EXT,CRM_EXT,DEP_EXT ext
```

---

## Legenda de Cores

| Cor | Camada / Tipo |
|---|---|
| 🔵 Azul | Experience Layer / Inbound |
| 🟡 Amarelo | Process Layer / Orquestração BW |
| 🟢 Verde | System Layer / Outbound |
| 🟣 Roxo | Infraestrutura / Serviços Transversais |
| ⚪ Cinza | Sistemas Externos |

---

## Comparativo de Componentes

| Componente TIBCO BW 5.x | Equivalente Mule 4 |
|---|---|
| SOAP WS + `TroubleTicket.wsdl` | HTTP Listener + APIkit (OAS 3.0 / TMF 621) |
| JMS Queue Receiver (TIBCO EMS) | Anypoint MQ Subscriber / JMS Connector |
| `*In` → `*Orq` → `*Out` processes | Flow + Flow Reference + SubFlow |
| DePara Engine (Oracle + SharedVariable) | Database Connector + ObjectStore v2 (cache) |
| `ERR_Handle` (retry/suspend/kill) | Error Handler + Until-Successful + DLQ (Anypoint MQ) |
| `Set_Log` (JMS + Oracle) | Logger + Anypoint Monitoring |
| `sharedvariable` / `jobsharedvariable` | Object Store v2 |
| `sharedLock` | Idempotency Filter |
| `.javaxpath` Java custom functions | Modulos DataWeave 2.0 |
| Certificados individuais SSL | Mule TLS Context (global config) |
| IBM MQ (TMX) | Anypoint Connector for IBM MQ |
| Salesforce Enterprise WSDL | Salesforce Connector (OAuth 2.0 JWT) |
| TIBCO ADB Adapter | Database Connector (Oracle) |
| `.substvar` files | `src/main/resources/config.yaml` + Secure Properties |
| TIBCO Hawk (monitoring) | Anypoint Monitoring + Runtime Manager |
