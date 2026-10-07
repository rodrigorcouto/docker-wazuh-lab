# Relatório Técnico — Implantação do Wazuh via Docker Compose

## 1. Informações Gerais

| Item | Informação |
|---|---|
| Solução | Wazuh |
| Método de implantação | Docker Compose |
| Versão do Wazuh | 4.14.8 |
| Sistema operacional | Ubuntu Server 22.04.5 LTS |
| Arquitetura | x86_64 |
| Hostname | `wazuh` |
| Endereço IP | `192.168.1.50` |
| CPU | 4 vCPU |
| Memória RAM | 7,8 GiB |
| Armazenamento | Aproximadamente 48 GB |
| Docker Engine | 29.2.8 |
| Docker Compose | 5.6.0 |
| Data da validação final | 07/10/2026 |

---

# 2. Resumo / Escopo

Foi realizada a implantação de uma instância do **Wazuh 4.14.8** utilizando Docker Compose sobre Ubuntu Server 22.04.5 LTS.

O escopo contemplou:

- preparação do sistema operacional;
- instalação do Docker Engine e Docker Compose;
- configuração do parâmetro `vm.max_map_count`;
- obtenção e seleção da versão 4.14.8 do repositório `wazuh-docker`;
- configuração do ambiente **single-node**;
- geração dos certificados TLS;
- validação da configuração Docker Compose;
- inicialização dos containers;
- análise dos logs do Wazuh Indexer;
- análise dos logs do Wazuh Manager;
- análise dos logs do Wazuh Dashboard;
- validação do acesso ao Dashboard via HTTPS;
- autenticação no Dashboard.

O ambiente foi validado até o ponto em que o **Dashboard estava acessível e o login foi realizado com sucesso**.

A instalação de agentes Wazuh ainda não fazia parte da etapa concluída.

---

# 3. Ambiente

## 3.1 Sistema operacional

```text
Ubuntu Server 22.04.5 LTS
Arquitetura: x86_64
```

A máquina foi configurada com:

```text
CPU: 4 vCPU
RAM: 7,8 GiB
Disco: aproximadamente 48 GB
```

## 3.2 Docker

Foram utilizadas:

```text
Docker Engine 29.2.8
Docker Compose v5.6.0
```

## 3.3 Estrutura do projeto

O repositório foi clonado em:

```text
~/wazuh-docker/wazuh-docker
```

Foi utilizada a estrutura `single-node`:

```text
single-node/
├── config/
│   └── certs.yml
├── docker-compose.yml
├── generate-indexer-certs.yml
└── README.md
```

A versão utilizada foi selecionada por:

```bash
git fetch --all --tags
git checkout v4.14.8
```

Resultado:

```text
HEAD detached at v4.14.8
working tree clean
```

---

# 4. Sintoma

Durante a inicialização do ambiente foram observados alguns comportamentos que exigiram validação:

1. O gerador de certificados apresentou:

```text
/wazuh-certs-tool.sh: line 636: find: command not found
```

2. Durante o startup inicial do Dashboard, foram registradas tentativas de conexão recusadas ao Wazuh Indexer:

```text
[ConnectionError]: connect ECONNREFUSED 172.18.0.2:9200
```

3. Foram observadas mensagens de aviso relacionadas a configurações depreciadas e `agentkeepalive`.

Apesar desses eventos, era necessário determinar se eles representavam falhas impeditivas ou apenas eventos transitórios durante a inicialização.

---

# 5. Hipóteses

| Hipótese | Classificação |
|---|---|
| Indexer ainda não estava pronto quando o Dashboard iniciou | **Confirmada como condição transitória de startup** |
| Falha definitiva de comunicação entre Dashboard e Indexer | Não confirmada |
| Problema nos certificados TLS | Não confirmado |
| Falha de configuração do Docker Compose | Não confirmada |
| Falha de inicialização do Manager | Não confirmada |
| Erro impeditivo relacionado ao `find` durante geração dos certificados | Não confirmado como impeditivo |

**Importante:** não foi identificada evidência suficiente para afirmar que qualquer uma das hipóteses não confirmadas tenha sido causa raiz do problema.

---

# 6. Diagnóstico e Evidências

## 6.1 Validação dos certificados

Foi executado:

```bash
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```

O processo apresentou:

```text
/wazuh-certs-tool.sh: line 636: find: command not found
```

Apesar da mensagem, o processo continuou e os certificados foram gerados.

A validação posterior apresentou os seguintes arquivos:

```text
admin-key.pem
admin.pem
root-ca.key
root-ca-manager.key
root-ca-manager.pem
root-ca.pem
wazuh.dashboard-key.pem
wazuh.dashboard.pem
wazuh.indexer-key.pem
wazuh.indexer.pem
wazuh.manager-key.pem
wazuh.manager.pem
```

Também foi executado:

```bash
sudo docker compose config --quiet
```

Sem retorno de erro.

Isso forneceu evidência de que a configuração do Compose estava sintaticamente válida.

---

## 6.2 Inicialização dos containers

Foi executado:

```bash
sudo docker compose up -d
```

Os containers foram criados/iniciados:

```text
single-node-wazuh.indexer-1
single-node-wazuh.manager-1
single-node-wazuh.dashboard-1
```

O `docker compose ps` apresentou os serviços em estado `Up`.

Portas publicadas:

```text
Dashboard: 0.0.0.0:443->5601
Indexer:   0.0.0.0:9200->9200
Manager:   1514-1515/tcp
           514/udp
           55000/tcp
```

---

## 6.3 Wazuh Indexer

Foi executado:

```bash
sudo docker compose logs --tail=100 wazuh.indexer
```

Os logs demonstraram evolução do estado de saúde do cluster, incluindo:

```text
cluster health GREEN
```

Também foi observada a criação de índices, incluindo:

```text
wazuh-alerts-4.x-2026.10.07
```

E:

```text
.plugins-ml-config
ML configuration initialized successfully
```

### Diagnóstico

Os registros fornecem evidência de que o **Wazuh Indexer inicializou e atingiu estado GREEN**, não havendo evidência de falha persistente do serviço.

---

## 6.4 Wazuh Manager

Foi executado:

```bash
sudo docker compose logs --tail=100 wazuh.manager
```

Foram observadas inicialmente algumas mensagens:

```text
find: '/proc/400/fd/5': No such file or directory
```

Essas mensagens não impediram a inicialização.

Posteriormente foram observados processos do Manager em execução, incluindo:

```text
wazuh-apid
wazuh-csyslogd
wazuh-integratord
wazuh-agentlessd
wazuh-authd
wazuh-db
wazuh-execd
wazuh-analysisd
wazuh-syscheckd
wazuh-remoted
wazuh-logcollector
wazuh-monitord
wazuh-modulesd
```

Também houve evidência de funcionamento do Filebeat e comunicação com o Indexer:

```text
Connection to backoff(elasticsearch(https://wazuh.indexer:9200)) established
```

Foi observado ainda:

```text
IndexerConnector initialized successfully
```

e o carregamento do template e pipeline.

### Diagnóstico

As evidências demonstraram que o **Manager estava operacional e conectado ao Indexer**.

---

## 6.5 Wazuh Dashboard

Foi executado:

```bash
sudo docker compose logs --tail=100 wazuh.dashboard
```

Durante a inicialização foram observados:

```text
Waiting until all OpenSearch nodes are compatible with OpenSearch Dashboards before starting saved objects migrations...
```

Em seguida:

```text
[ConnectionError]: connect ECONNREFUSED 172.18.0.2:9200
```

Essas mensagens ocorreram no início do processo.

Posteriormente o Dashboard prosseguiu com:

```text
Starting saved objects migrations
```

Criou o índice:

```text
.kibana_1
```

E iniciou o servidor HTTPS:

```text
Server running at https://0.0.0.0:5601
```

Também:

```text
http server running at https://0.0.0.0:5601
```

O plugin Wazuh iniciou corretamente e criou o índice de monitoramento:

```text
wazuh-monitoring-2026.41w index created
```

O log também apresentou:

```text
resource_already_exists_exception: index [wazuh-statistics-2026.41w/Hy7zEr7CTMyzurGjBjtG8g] already exists
```

Esse registro não impediu a continuidade do serviço, que posteriormente registrou:

```text
Settings added to wazuh-monitoring-2026.41w index
```

---

# 7. Causa Raiz

**Não foi identificada uma causa raiz de uma falha persistente no ambiente.**

O principal evento potencialmente interpretado como falha foi:

```text
[ConnectionError]: connect ECONNREFUSED 172.18.0.2:9200
```

Entretanto, os próprios logs demonstraram que posteriormente o Dashboard conseguiu prosseguir com as migrações, inicializar os plugins e disponibilizar o servidor HTTPS.

Portanto, com base exclusivamente nas evidências disponíveis, a conclusão tecnicamente suportada é:

> **O `ECONNREFUSED` ocorreu durante a inicialização do Dashboard, antes de o Indexer estar disponível para conexão, e não caracterizou uma falha persistente do ambiente.**

Não há evidência suficiente para atribuir uma causa raiz diferente dessa condição transitória de inicialização.

---

# 8. Tentativas de Correção

## 8.1 Correção do diretório de trabalho

Inicialmente houve execução no diretório incorreto do repositório. O ambiente foi ajustado para:

```text
~/wazuh-docker/wazuh-docker
```

e posteriormente:

```text
~/wazuh-docker/wazuh-docker/single-node
```

## 8.2 Acesso aos certificados

A tentativa de listar os certificados sem privilégios apresentou:

```text
Permission denied
```

A validação foi realizada posteriormente com `sudo`.

## 8.3 Mensagem `find: command not found`

Durante a geração dos certificados:

```text
/wazuh-certs-tool.sh: line 636: find: command not found
```

Não houve necessidade de uma correção adicional registrada, pois o processo continuou e os certificados necessários foram gerados.

## 8.4 Conexão recusada do Dashboard

Não foi aplicada uma alteração específica para corrigir o `ECONNREFUSED`.

O Dashboard continuou seu processo de inicialização e posteriormente estabeleceu as condições necessárias para operar.

**Portanto, não deve ser registrado que houve uma "correção" específica do `ECONNREFUSED`, pois isso não foi demonstrado pelas evidências.**

---

# 9. Solução Implementada

A solução implementada consistiu na implantação do Wazuh em arquitetura **single-node utilizando Docker Compose**, com:

- Ubuntu Server 22.04.5 LTS;
- Docker Engine 29.2.8;
- Docker Compose 5.6.0;
- Wazuh 4.14.8;
- geração dos certificados TLS;
- configuração válida do Docker Compose;
- inicialização do Indexer;
- inicialização do Manager;
- inicialização do Dashboard.

O comando utilizado para iniciar a solução foi:

```bash
sudo docker compose up -d
```

---

# 10. Validação Pós-Correção

Foram realizados os seguintes testes:

### 10.1 Configuração Docker Compose

```bash
sudo docker compose config --quiet
```

**Resultado:** sem retorno de erro.

### 10.2 Estado dos containers

```bash
sudo docker compose ps
```

**Resultado:** containers `wazuh.indexer`, `wazuh.manager` e `wazuh.dashboard` em estado `Up`.

### 10.3 Indexer

Evidência:

```text
cluster health GREEN
```

### 10.4 Manager

Evidência:

```text
Connection to backoff(elasticsearch(https://wazuh.indexer:9200)) established
```

### 10.5 Dashboard

Evidência:

```text
Server running at https://0.0.0.0:5601
```

### 10.6 Acesso pela rede

Foi acessado pelo navegador:

```text
https://192.168.1.50/app/login
```

A tela de autenticação do Wazuh foi apresentada.

### 10.7 Autenticação

O login foi realizado com sucesso e o Dashboard apresentou a tela **Overview**.

---

# 11. Resultado Final

O ambiente encontra-se **operacional**.

A interface apresentou:

| Indicador | Resultado |
|---|---:|
| Agents registrados | 0 |
| Alertas Critical | 0 |
| Alertas High | 0 |
| Alertas Medium | 45 |
| Alertas Low | 141 |

A ausência de agentes é compatível com o estágio atual do laboratório: **o servidor Wazuh foi implantado, mas ainda não foi realizado o cadastro/instalação de um agente**.

A tela também disponibilizou os módulos de:

- Configuration Assessment;
- Malware Detection;
- Threat Hunting;
- Vulnerability Detection;
- File Integrity Monitoring;
- MITRE ATT&CK;
- IT Hygiene;
- PCI DSS;
- GDPR;
- HIPAA;
- Docker;
- Amazon Web Services;
- Google Cloud;
- GitHub.

---

# 12. Lições Aprendidas

1. **A ordem de inicialização dos componentes é relevante.**  
   O Dashboard pode iniciar antes de o Indexer estar pronto, produzindo temporariamente `ECONNREFUSED`.

2. **Logs devem ser analisados considerando o ciclo completo de inicialização.**  
   Uma mensagem de erro durante o startup não deve ser automaticamente classificada como falha definitiva.

3. **O estado final dos serviços é mais representativo que mensagens isoladas do startup.**  
   O Indexer atingiu `GREEN`, o Manager estabeleceu conexão com o Indexer e o Dashboard iniciou o servidor HTTPS.

4. **A validação do Docker Compose antes da inicialização é importante.**

```bash
sudo docker compose config --quiet
```

5. **Certificados devem ser validados após sua geração.**  
   Apesar da mensagem `find: command not found`, os certificados necessários foram efetivamente produzidos.

6. **O ambiente atual é adequado para prosseguir com o estudo de agentes, monitoramento e posteriormente IaC.**

---

# 13. Conclusão

A implantação do **Wazuh 4.14.8 via Docker Compose** foi concluída com sucesso em uma VM Ubuntu Server 22.04.5 LTS.

### Problema

Durante a inicialização foram observados avisos e uma tentativa inicial de conexão recusada do Dashboard ao Indexer:

```text
[ConnectionError]: connect ECONNREFUSED 172.18.0.2:9200
```

### Causa

Com base exclusivamente nas evidências disponíveis, o `ECONNREFUSED` correspondeu a uma **condição transitória durante o startup**, enquanto o Indexer ainda não estava disponível. **Não foi identificada uma causa raiz de falha persistente.**

### Correção / Solução

Foi concluída a implantação do stack:

```text
Wazuh Manager
       │
       ▼
Wazuh Indexer
       │
       ▼
Wazuh Dashboard
```

utilizando Docker Compose e certificados TLS gerados pelo projeto.

### Testes realizados

- validação do `docker-compose.yml`;
- geração e validação dos certificados;
- verificação dos containers;
- análise dos logs do Indexer;
- análise dos logs do Manager;
- análise dos logs do Dashboard;
- validação do servidor HTTPS;
- acesso via `https://192.168.1.50`;
- autenticação no Dashboard.

### Status final

**AMBIENTE OPERACIONAL**

```text
Indexer      → OPERACIONAL / GREEN
Manager      → OPERACIONAL / conectado ao Indexer
Dashboard    → OPERACIONAL / HTTPS
Autenticação → VALIDADA
Agents       → 0 registrados
```

O próximo estágio técnico, ainda **não realizado neste relatório**, é a implantação e registro de um **Wazuh Agent** para validar a cadeia completa de monitoramento ponta a ponta.
