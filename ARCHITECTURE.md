# Arquitetura de Integração - Claro TIBCO Migration

## Diagrama da Arquitetura

```mermaid
flowchart TD
    classDef consumers  fill:#E8F4FD,stroke:#2196F3,color:#0D47A1
    classDef gateway    fill:#FFF3E0,stroke:#FF9800,color:#E65100
    classDef experience fill:#E8F5E9,stroke:#4CAF50,color:#1B5E20
    classDef process    fill:#F3E5F5,stroke:#9C27B0,color:#4A148C
    classDef system     fill:#FFF8E1,stroke:#FFC107,color:#FF6F00
    classDef infra      fill:#ECEFF1,stroke:#607D8B,color:#263238
    classDef backends   fill:#FAFAFA,stroke:#9E9E9E,color:#212121
    classDef legacy     fill:#FFEBEE,stroke:#F44336,color:#B71C1C,stroke-dasharray: 5 5

    subgraph CON["  Sistemas Consumidores  "]
        C1["CRM / Siebel\n(Sales App)"]:::consumers
        C2["Portal B2B\n(Web / Mobile)"]:::consumers
        C3["Sistemas Legados\n(SOAP direto)"]:::legacy
    end

    subgraph GW["  Anypoint API Gateway  "]
        G1["OAuth 2.0 · mTLS · Rate Limiting\nSLA Policies · IP Allowlist"]:::gateway
    end

    subgraph EXP["  Experience Layer  "]
        XA["Customer Account\nExperience API\n─────────────────\nREST · TMF 629 / TMF 666\nPOST   /customerBillingAccounts\nGET    /customerBillingAccounts/{id}\nPATCH  /customerBillingAccounts/{id}\nGET    /customers/{id}\nGET    /customers/{id}/creditProfile"]:::experience
        XB["⚠️ SOAP Compatibility\nFacade\n─────────────────\nSOAP 1.1 · doc/literal\nPOST /CriarContaCliente\n(Temporário — Transição)"]:::legacy
    end

    subgraph PRC["  Process Layer  "]
        PA["Account Management\nProcess API\n─────────────\nOrquestra criação de conta\nRoteamento async → backends\nValidação + enriquecimento\nDe-Para (lookup)"]:::process
        PB["Billing Account\nProcess API\n─────────────\nQuery contas faturamento\nUpdate dados financeiros\nConsulta log por serviço\nConsulta débito"]:::process
        PC["Credit & Fraud\nProcess API\n─────────────\nAnálise de crédito\nConsulta histórico vendas\nCancelamento análise\nFraude (CRIVO)"]:::process
    end

    subgraph INF["  Plataforma Compartilhada (Cross-Cutting)  "]
        direction LR
        MQ["Anypoint MQ\n─────────\nSubstitui TIBCO EMS\nQueues + Dead Letter Queue\nAck manual · Retry policy"]:::infra
        OS["Object Store v2\n─────────\nSubstitui Shared Variables\nDe-Para cache\nToken Salesforce"]:::infra
        MON["Anypoint Monitoring\n─────────\nSubstitui Log Library\n+ Audit Library\nDashboards · Alertas"]:::infra
        SEC["Anypoint Security\n─────────\nTLS Contexts\n(migração dos .cert)\nSecrets Manager"]:::infra
    end

    subgraph SYS["  System Layer  "]
        SA["CLE\nSystem API\n─────────────\nJava Custom Connector\nCLECN600/622/680\nConta provisioning"]:::system
        SB["Parallels\nSystem API\n─────────────\nWeb Service Consumer\nSOAP HTTPS\nConfirmação CLE"]:::system
        SC["Salesforce\nSystem API\n─────────────\nSalesforce Connector\nEnterprise API\nConta + Contato SF"]:::system
        SD["Siebel CRM\nSystem API\n─────────────\nDatabase Connector\nJDBC Oracle\nDados de cliente"]:::system
        SE["ICMS\nSystem API\n─────────────\nDatabase Connector\nJDBC Oracle\nLog faturamento"]:::system
        SF2["WRH · CRIVO\nSystem API\n─────────────\nWeb Service Consumer\nSOAP HTTP\nEmployee + Crédito"]:::system
    end

    subgraph BCK["  Sistemas de Destino  "]
        direction LR
        B1[("CLE\nJava Libs\nCLECN600/622/680")]:::backends
        B2[("Parallels\nMódulo Integrador\nSOAP")]:::backends
        B3[("Salesforce\nEnterprise\nREST/SOAP")]:::backends
        B4[("Siebel CRM\nOracle DB")]:::backends
        B5[("ICMS\nOracle DB")]:::backends
        B6[("WRH · CRIVO\nBarramento SOAP")]:::backends
    end

    %% Fluxo principal
    C1 & C2 --> G1
    C3 -.->|"⚠️ legado direto\n(fase de transição)"| XB
    G1 --> XA
    G1 -.->|transição| XB
    XB -.->|"delega\npós-adaptação"| XA

    %% Experience → Process
    XA -->|"CriarConta\n(async)"| PA
    XA -->|"Query / Update\nBilling"| PB
    PA -->|"Verificação\ncrédito"| PC

    %% Process → MQ (async backbone)
    PA <-->|"async\nfire & forget"| MQ
    MQ -->|"CONTA.CLE.OUT"| SA
    MQ -->|"CONTA.PARALLELS.OUT"| SB
    MQ -->|"CONTA.SF.OUT"| SC

    %% Process → System (sync)
    PB --> SD & SE
    PC --> SF2

    %% System → Backends
    SA --> B1
    SB --> B2
    SC --> B3
    SD --> B4
    SE --> B5
    SF2 --> B6

    %% Cross-cutting
    PA & PB & PC -.->|"logs\nauditoria"| MON
    PA & PB -.->|"cache\nDe-Para\ntoken"| OS
    SA & SB & SC & SD & SE & SF2 -.->|"TLS / secrets"| SEC
```

## Legenda de Cores

| Cor | Camada | Descrição |
|-----|--------|-----------|
| 🔵 Azul | **Consumidores** | Sistemas cliente / CRM Siebel |
| 🟠 Laranja | **API Gateway** | Anypoint Gateway com políticas de segurança |
| 🟢 Verde | **Experience Layer** | APIs de experiência (REST/SOAP) |
| 🟣 Roxo | **Process Layer** | APIs de processo com orquestração |
| 🟡 Amarelo | **System Layer** | Conectores para sistemas legados |
| ⚫ Cinza | **Plataforma Compartilhada** | MQ, Object Store, Monitoring, Security |
| ⚪ Branco | **Sistemas de Destino** | Backends (CLE, Parallels, Salesforce, etc.) |
| 🔴 Vermelho | **Legacy** | Componentes em transição (linhas tracejadas) |

## Componentes Principais

### Camada de Consumidores
- **CRM / Siebel (Sales App)** - Aplicação de vendas
- **Portal B2B** - Aplicações web e mobile
- **Sistemas Legados** - Integração SOAP direta (em transição)

### Anypoint API Gateway
- Autenticação OAuth 2.0 e mTLS
- Rate Limiting e SLA Policies
- IP Allowlist

### Experience Layer
- **Customer Account Experience API** - REST com padrão TMF 629/666
- **SOAP Compatibility Facade** - Compatibilidade com sistemas legados (transição)

### Process Layer
- **Account Management Process API** - Orquestra criação de conta
- **Billing Account Process API** - Query e update de contas de faturamento
- **Credit & Fraud Process API** - Análise de crédito e fraude

### System Layer
- CLE, Parallels, Salesforce, Siebel CRM, ICMS, WRH, CRIVO

### Plataforma Compartilhada
- **Anypoint MQ** - Substitui TIBCO EMS
- **Object Store v2** - Cache de De-Para e tokens
- **Anypoint Monitoring** - Logs e auditoria
- **Anypoint Security** - TLS e Secrets Manager
