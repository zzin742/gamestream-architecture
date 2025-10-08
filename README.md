# 🎮 GameStream — Arquitetura de Referência (Escalabilidade & Resiliência)

## 📘 Contexto
A **TechCorp**, uma Big Tech de tecnologia, decidiu lançar um novo serviço de **streaming de jogos em nuvem** chamado **GameStream**.  
O objetivo é alcançar **1 milhão de usuários nos primeiros 6 meses**, com foco em **alta escalabilidade** e **resiliência** para garantir que o serviço nunca fique fora do ar.

Como **Arquiteto de Soluções**, o desafio é desenhar a **Arquitetura de Referência inicial** para os componentes mais críticos do sistema.

---

## 🧩 1) Divisão da Aplicação (Microsserviços e Desacoplamento)

### 🔹 Microsserviço 1 — Orquestrador de Sessões de Jogo
- **Função:** cria, gerencia e encerra sessões de jogo em servidores com GPU; emparelha o jogador com a instância mais próxima; envia eventos de status (início/fim de sessão).
- **Por que é crítico:** controla o fluxo de quem está jogando em tempo real.
- **Se falhar:** apenas a criação de **novas sessões** é afetada; **as sessões ativas continuam** normalmente nos servidores de streaming.

---

### 🔹 Microsserviço 2 — Catálogo & Recomendações
- **Função:** lista os jogos disponíveis, metadados (descrição, imagens, trailers) e recomenda títulos com base no histórico do usuário.
- **Por que é crítico:** é a porta de entrada da experiência do jogador (antes de iniciar o streaming).
- **Se falhar:** o usuário ainda pode acessar conteúdo em **cache/CDN**; **as partidas em andamento não param.**

---

### 🔹 Mecanismo de Comunicação
- **Tipo:** Comunicação **assíncrona**, desacoplada por eventos.
- **Serviços de Cloud possíveis:**  
  - **AWS:** SNS + SQS  
  - **GCP:** Pub/Sub  
  - **Azure:** Service Bus  
  - **Self-managed:** Apache Kafka  
- **Motivo:** evita dependência direta entre microsserviços; se um falhar, o outro continua processando mensagens da fila normalmente.  
- **Boas práticas:** retentativa com backoff, DLQ (Dead Letter Queue), idempotência e circuit breaker.

---

## 🌍 2) Alta Resiliência e Disponibilidade (Geografia)

### 🔹 Conceito de Distribuição
- Usar **Zonas de Disponibilidade (AZs)** dentro de cada região e replicar o serviço em **múltiplas Regiões** (modelo ativo-ativa).  
- Implementar **CDN (Content Delivery Network)** e **Edge POPs** para entregar vídeo e comandos com latência mínima.  
- Front global com **GSLB (Global Server Load Balancing)** ou **Anycast DNS**.

### 🔹 Justificativa
- Se uma **zona** ou até **uma região inteira** falhar, o tráfego é redirecionado automaticamente para outra, sem interrupção.  
- Mantém **baixa latência** global e **alta disponibilidade**, essencial para streaming em tempo real.  
- A **CDN** também ajuda a reduzir carga nos servidores principais.

---

## ⚖️ 3) Escalabilidade de Tráfego (Entrada)

### 🔹 Serviço Necessário
- **Balanceador de Carga Global (Layer 7)** com **WAF (Web Application Firewall)** e **Auto Scaling**.

Exemplos:
- **AWS:** Application Load Balancer + Global Accelerator  
- **GCP:** Cloud Load Balancing  
- **Azure:** Front Door + App Gateway

### 🔹 Função
- É o **primeiro componente** a receber as requisições dos usuários.  
- Analisa o tráfego HTTP/HTTPS e distribui as conexões de forma **inteligente e dinâmica** para os servidores disponíveis.  
- Garante que **nenhum servidor fique sobrecarregado** e aumenta/diminui o número de instâncias conforme o volume de usuários.

---

## 🔒 4) Segurança por Design (Identity and Access)

### 🔹 Pilar de Segurança (Conceito)
- **IAM (Identity and Access Management)** com modelo **Zero-Trust** entre microsserviços.

### 🔹 Por que é crucial
- Garante que **somente o Microsserviço 1 (autenticado)** possa acessar o **Microsserviço 2**.  
- Evita acessos indevidos e ataques internos.  
- Implementação via **mTLS (mutual TLS)**, **OAuth2**, **JWT Tokens** ou **Workload Identity Federation**.
- Políticas de **menor privilégio (Least Privilege)** e **RBAC (Role-Based Access Control)** asseguram que cada serviço acesse apenas o que precisa.

---

## 🧭 5) Diagrama de Arquitetura (Mermaid)

```mermaid
flowchart LR
    U[Usuário] --> DNS[Anycast DNS / GSLB]
    DNS --> WAF[WAF + Load Balancer Global]
    WAF --> EDGE[CDN / Edge POPs]
    EDGE --> STREAM[Nós de Streaming (GPU) - Multi-AZ/Região]
    WAF --> API[API Gateway (Plano de Controle)]
    
    API --> MS1[MS1: Orquestrador de Sessões]
    API --> MS2[MS2: Catálogo & Recomendações]
    
    MS1 <-- Pub/Sub --> MS2

    subgraph Observabilidade
        LOGS[Logs]
        METRICS[Métricas]
        TRACES[Traces]
    end

    STREAM --- LOGS
    MS1 --- LOGS
    MS2 --- LOGS
    STREAM --- METRICS
    MS1 --- METRICS
    MS2 --- METRICS
    STREAM --- TRACES
    MS1 --- TRACES
    MS2 --- TRACES
