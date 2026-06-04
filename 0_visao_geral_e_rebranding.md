# 📖 Visão Geral do Ecossistema AMZX e Rebranding (Scala-First)

Este documento apresenta uma visão detalhada e exaustiva da arquitetura do ecossistema **AMZX** (Amazonic One), dissecando o núcleo dos dois componentes nativos desenvolvidos em **Scala**: o **AMZX Scala Node** (`amzx`) e o **AMZX Matcher DEX** (`matcher`).

---

## 🔍 1. Arquitetura de Alto Nível do Ecossistema

O ecossistema AMZX opera uma blockchain de alta performance integrada nativamente a um motor de correspondência descentralizada (DEX) ultrarrápido, robusto e sem custódia.

```mermaid
graph TD
    User([Cliente / Wallet / MetaMask]) -->|API REST (Porta 6869)| Node[AMZX Scala Node]
    User -->|API REST & WebSocket (Porta 6886)| Matcher[AMZX Matcher DEX]
    
    subgraph Processo do Node [Processo Java - AMZX Node]
        Node
        DEXExt[DEXExtension gRPC Server]
        UtxPool[UTX Pool - Mempool]
        Miner[Miner - Forjador de Blocos]
        DB[(RocksDB - Estado & Cadeia)]
    end
    
    subgraph Processo do Matcher [Processo Java - AMZX Matcher]
        Matcher
        OrderBook[Livre de Ofertas em Memória TreeMap]
        AddressActor[AddressActor - Saldos & Alocações]
        LocalQueue[Fila de Eventos Local RocksDB / Kafka]
    end
    
    Matcher -->|Conexão Segura gRPC (Porta 6887)| DEXExt
    DEXExt -->|Validação de Saldos / Estados| Node
    Matcher -->|Injeção de ExchangeTransaction| Node
    UtxPool --> Miner
    Miner --> DB
```

### Componentes de Software:
1. **AMZX Scala Node (`amzx`)**:
   - **Mecanismo Core**: Valida as transações recebidas da rede ou de APIs, gerencia a piscina de transações não confirmadas (UTX Pool) e coordena a mineração de novos blocos/micro-blocos via protocolo **Waves-NG** e algoritmo **LPoS** (Leased Proof of Stake).
   - **Armazenamento**: Grava o estado do ledger e a cadeia física de blocos de forma embarcada no **RocksDB**.
   - **Interface Externa**: Expõe uma API REST completa para transações, contratos RIDE, oráculos e consultas de saldos, além do endpoint de compatibilidade Ethereum JSON-RPC (`/eth`).
   - **Integração DEX**: Fornece um servidor de streaming gRPC através da extensão `DEXExtension` para prover dados de saldos e transações com latência na casa dos microssegundos.

2. **AMZX Matcher DEX (`matcher`)**:
   - **Mecanismo Core**: Um motor financeiro que cruza ordens de compra e venda assinadas digitalmente pelos usuários. 
   - **Não-Custodial**: Mantém e emparelha as ordens em memória RAM, gerando uma transação do tipo `ExchangeTransaction` (ID 7) assinada pela DEX ao encontrar uma correspondência. A transferência real de fundos só ocorre de fato quando a transação é gravada na blockchain, impedindo roubos ou hacks de custódia.
   - **Armazenamento Local**: Utiliza uma instância isolada do **RocksDB** para gerenciar sua fila de mensagens local, estado operacional das ordens e logs do livro.

---

## 🎭 2. Arquitetura de Atores Scala (Akka/Pekko)

Tanto o Nó quanto o Matcher são construídos utilizando o modelo de atores sobre **Akka / Pekko**, o que confere tolerância a falhas, concorrência segura e altíssimo desempenho de processamento orientado a mensagens.

### A. Hierarquia de Atores no AMZX Node
*   **`NetworkActor`**: Responsável pelo gerenciamento de soquetes TCP e canais Netty, controlando as conexões P2P ativas com outros nós validadores da rede.
*   **`CoordinatorActor`**: Atua como o maestro da sincronização de blocos. Recebe blocos gerados por outros nós e coordena a validação cronológica das assinaturas e transações antes de gravá-los no RocksDB.
*   **`UtxPoolActor`**: Gerencia a memória volátil de transações enviadas que aguardam mineração. Garante que transações duplicadas ou inválidas (por falta de saldo) sejam rejeitadas antes de entrarem na fila.
*   **`MinerActor`**: Ator periódico que calcula se o nó atingiu o hit matemático de geração do próximo bloco através do cálculo do FairPoS e provas VRF. Ao ganhar o direito de minerar, ele monta o Key Block e passa a transmitir micro-blocos contínuos.

### B. Hierarquia de Atores no AMZX Matcher DEX
*   **`MatcherActor`**: Ator supervisor mestre do processo da DEX. Inicializa e coordena os componentes secundários, além de monitorar o desligamento seguro (graceful shutdown) das conexões.
*   **`OrderBookDirectoryActor`**: Atua como um catálogo dinâmico de todos os mercados ativos (pares de moedas). Quando uma nova ordem chega, ele a direciona para o ator correspondente ao par de ativos.
*   **`OrderBookActor`**: Gerencia o livro de ofertas de um par de ativos específico. Mantém o TreeMap de ordens de bids e asks em memória. Processa as mensagens de adição, cancelamento e cruzamento de ordens em sequência estrita e síncrona.
*   **`AddressActor`**: Gerencia as permissões, saldos e ordens abertas de um endereço de usuário específico. Ele evita que o usuário gaste o mesmo saldo em múltiplas ordens simultâneas (Double Spending preventivo em memória).
*   **`WsExternalClientHandlerActor`**: Gerencia a conexão WebSocket individual de um cliente externo, encaminhando eventos de livro ou alterações de saldo privados em tempo real.

---

## 🔌 3. A Extensão Integrada `DEXExtension`

A **`DEXExtension`** é um componente dinâmico implementado em Scala que atua como uma ponte de comunicação de baixa latência acoplada diretamente ao processo do nó.

```
+-------------------------------------------------------------+
|                     AMZX Scala Node JVM                     |
|                                                             |
|  +--------------------+             +--------------------+  |
|  |     Blockchain     |             |    DEXExtension    |  |
|  |   (Core Engine)    |<=== Event ==|  - gRPC Server     |  |
|  |                    |   Listener  |  - Port 6887       |  |
|  +--------------------+             +--------------------+  |
+-------------------------------------------------------------+
                                                 |
                                                 | gRPC Protocol Buffer Stream
                                                 v
                               +-------------------------------+
                               |    AMZX Matcher Process JVM   |
                               +-------------------------------+
```

*   **Ciclo de Vida Acoplado**: É inicializada pelo nó durante a leitura do arquivo `application.conf` na diretiva `amzx.extensions`.
*   **Injeção de Eventos**: Ela se inscreve nos barramentos de eventos internos da blockchain. Sempre que um bloco é minerado, uma transação é revertida, ou um saldo é atualizado, a `DEXExtension` intercepta o evento na memória JVM do nó.
*   **Canal gRPC de Alta Velocidade**: Ela formata e serializa esses eventos utilizando Protocol Buffers (Protobuf) e os transmite por um fluxo gRPC ativo (porta `6887`) diretamente para o Matcher DEX. Isso elimina a latência das consultas tradicionais HTTP REST.

---

## 🛠️ 4. O Desafio Técnico do Rebranding (Compatibilidade Binária)

Durante o rebranding completo do sistema de `"Waves"` para `"AMZX"` (Amazonic One), um desafio técnico crítico surgiu devido a restrições de compatibilidade binária de bibliotecas pré-compiladas de terceiros.

### A Restrição do JAR consolidado (`amzx-all-1.4.13.jar`)
O projeto utiliza um pacote binário pré-compilado consolidado denominado `amzx-all-1.4.13.jar` (localizado em `amzx-ext/lib/`). Este JAR contém as estruturas, modelos matemáticos de transações e a lógica de consenso compiladas sob o namespace original do pacote: **`com.wavesplatform.*`**.

*   **O Problema**: Se alterássemos agressivamente todas as ocorrências de pacotes de código-fonte de `com.wavesplatform` para `com.amzblockchain` ou renomeássemos variáveis de dados internas como `wavesAmount` para `amzxAmount` em classes do Scala, o compilador quebraria a herança de Traits seladas e causaria falhas catastróficas de ligação em tempo de execução (`LinkageError`, `ClassNotFoundException` ou `NoSuchMethodError`).
*   **A Solução de Engenharia**: Implementamos uma arquitetura híbrida de compatibilidade cirúrgica:
    1.  **Camada de Ligação Interna (Compatibilidade)**: O código-fonte Scala mantém as referências de pacotes internos (`com.wavesplatform.*`) e nomes de atributos binários exigidos pelo JAR pré-compilado para garantir integridade estrutural e compilação do SBT.
    2.  **Camada de Exibição e Consumo Externo (Branding)**: Todas as APIs REST, esquemas JSON, canais WebSocket, arquivos de logs, saídas gRPC e arquivos de configuração HOCON foram adaptados para expor e consumir dados utilizando a marca **`AMZX`**.
    3.  **Mapeadores de Tradução**: Arquivos como `AmzxToPbConversions.scala` e `PbToAmzxConversions.scala` traduzem em tempo real as referências do ativo padrão blockchain (`Asset.Waves`) para o rótulo público `"AMZX"`, de modo que para o cliente final, desenvolvedores de dApps e APIs, a rede opera de forma homogênea sob o nome **AMZX**.

---

## ☕ 5. Ambiente de Execução Homologado

Toda a infraestrutura do ecossistema AMZX roda de forma otimizada sobre as seguintes definições de ambiente oficiais (conforme especificado no `README.md` original da Blockchain):

*   **JVM de Compilação e Runtime**: **OpenJDK 17** (Versão 17.0.x LTS)
    *   *Nota Crítica*: O uso de JDKs mais recentes, como Java 21, não é homologado ou recomendado pelo repositório oficial da blockchain, podendo introduzir quebras silenciosas de linkage de classes, avisos de reflexão ilegal profundos ou incompatibilidades com bibliotecas nativas e classloaders dinâmicos da `DEXExtension`.
*   **Linguagens (Scala)**:
    *   **AMZX Node (Blockchain Node & waves-ext)**: Compilado nativamente com **Scala 3.8.3**
    *   **Matcher DEX**: Compilado nativamente com **Scala 2.13.14** (mantendo retrocompatibilidade segura através de canais gRPC/Protobuf desacoplados)
*   **Ferramenta de Build**: SBT 1.9.x+ (ou versão compatível)
*   **Sistema Operacional**: Linux (Ubuntu/Debian homologados)

### Por que o Java 17?
O Java 17 traz suporte nativo a coletas de lixo modernas de ultra-baixa latência (ZGC e G1GC aperfeiçoados), essenciais para evitar pausas de "stop-the-world" de compilação da JVM, garantindo que o Matcher cruze ordens e transmita eventos sem travamentos de thread. Além disso, o compilador Scala e os plug-ins SBT utilizados no projeto possuem estabilidade e conformidade estritas com o JDK 17, prevenindo avisos de reflexão ilegal e falhas de runtime associadas a versões mais antigas (Java 8) ou mais recentes (Java 21). A compilação e a execução de ambos os componentes (Blockchain e Matcher) são realizadas utilizando o OpenJDK 17 (localizado em `/usr/lib/jvm/java-1.17.0-openjdk-amd64/`).
