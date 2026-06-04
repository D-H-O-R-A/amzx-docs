# 🛠️ Referência Técnica e Desenvolvimento do Ecossistema AMZX (Guia do Desenvolvedor)

Este documento atua como a enciclopédia técnica de baixo nível definitiva para engenheiros que desenvolvem, depuram, otimizam ou integram novas soluções ao ecossistema **AMZX** (Node Scala e Matcher DEX).

---

## 📂 1. Mapeamento Profundo do Código-Fonte (Onde Alterar?)

Para realizar melhorias de código ou alterar o comportamento nativo da rede, localize os respectivos arquivos e módulos listados abaixo:

### A. Módulo AMZX Scala Node (`/amzx`)
*   **Regras de Consenso e Mineração LPoS**:
    *   `node/src/main/scala/com/wavesplatform/consensus/`: Contém os algoritmos matemáticos que determinam quem gera o próximo bloco com base no arrendamento de stake e criptografia VRF (Verifiable Random Function).
*   **Contratos Inteligentes e Compilador RIDE**:
    *   `lang/shared/src/main/scala/com/wavesplatform/lang/v1/`: Contém o parser, analisador sintático (AST), compilador, tipos de dados e funções internas da linguagem **RIDE** (da versão v3 à v6).
*   **Regras e Validação de Transações**:
    *   `node/src/main/scala/com/wavesplatform/transaction/`: Contém as definições binárias, serializadores, verificadores de assinatura e modificadores de estado de cada um dos tipos de transação da rede (Transfer, Issue, InvokeScript, etc.).
*   **Endpoints e Roteamento de APIs REST**:
    *   `node/src/main/scala/com/wavesplatform/api/http/`: Contém os servidores HTTP Akka-HTTP/Pekko-HTTP e as rotas que expõem os serviços JSON REST públicos e administrativos (como `/blocks`, `/transactions`, `/addresses`, `/node/status`).
*   **Extensão gRPC DEXExtension**:
    *   `grpc-server/src/main/scala/com/wavesplatform/api/grpc/`: Contém as assinaturas gRPC geradas via Protobuf que alimentam o Matcher DEX.

### B. Módulo AMZX Matcher DEX (`/matcher`)
*   **Core do Motor de Correspondência (Matching Engine)**:
    *   `dex/src/main/scala/com/wavesplatform/dex/market/`: Contém as classes que gerenciam a memória do Livro de Ofertas (`OrderBook.scala`), o cruzamento matemático de ordens e a manutenção das ordens parciais em memória.
*   **Serviços e Atores Akka/Pekko**:
    *   `dex/src/main/scala/com/wavesplatform/dex/Matcher.scala`: O ator supervisor Akka principal que inicializa todo o ecossistema da DEX, gerencia o ciclo de vida do RocksDB local e inicia os canais gRPC.
*   **Conexão de Integração gRPC (Cliente)**:
    *   `amzx-ext/src/main/scala/com/wavesplatform/dex/grpc/integration/`: Contém a lógica de tratamento das mensagens recebidas do Node validador por canais de fluxo gRPC e conversão entre as estruturas internas e o gRPC do Node.
*   **APIs WebSocket de Alta Velocidade**:
    *   `dex/src/main/scala/com/wavesplatform/dex/api/ws/`: Endpoints de conexões bidirecionais via WebSocket que fornecem aos clientes atualizações instantâneas de livros de ofertas, tickers e estados de ordens privadas em tempo real.

---

## 🔌 2. Tabela de Portas de Rede do Ecossistema

Esta tabela mapeia todas as portas de rede necessárias para a operação da blockchain e da DEX:

| Porta | Protocolo | Escopo de Rede | Função e Descrição |
| :---: | :---: | :--- | :--- |
| **`6868`** | TCP | Público (Mainnet/Custom) | Porta de comunicação **P2P** para sincronização de cadeia de blocos entre nós validadores. |
| **`6863`** | TCP | Público (Testnet) | Porta de sincronização **P2P** específica da Testnet de homologação. |
| **`6862`** | TCP | Público (Stagenet) | Porta de sincronização **P2P** específica da Stagenet de testes. |
| **`6869`** | HTTP | Interno / Proxy Reverso | Endpoint da **REST API** pública do Nó e barramento RPC compatível com a **MetaMask** (`/eth`). |
| **`6870`** | TCP | Interno | Porta do servidor **gRPC nativo** da extensão de streaming de dados do Nó. |
| **`6887`** | TCP | Interno | Porta do servidor **gRPC da `DEXExtension`** no Nó, consumido pelo Matcher. |
| **`6886`** | HTTP/WS | Interno / Proxy Reverso | Endpoint unificado da **REST API e do Canal WebSocket** do Matcher DEX. |

---

## 🛠️ 3. Interfaces de Serviços gRPC (Protobuf APIs)

As interfaces gRPC do Nó são definidas em arquivos `.proto` e compiladas de forma estática em classes Scala no subprojeto `grpc-server`. Elas dividem-se em 4 APIs fundamentais:

```
+---------------------------------------------------------------------------------+
|                                gRPC API SERVER                                  |
+---------------------------------------+-----------------------------------------+
                                        |
     +----------------------------------+----------------------------------+
     |                                  |                                  |
     v                                  v                                  v
+-------------------+          +------------------+          +--------------------+
|  TransactionsApi  |          |    BlocksApi     |          |    AccountsApi     |
+-------------------+          +------------------+          +--------------------+
  - Broadcast()                  - GetBlockRange()             - GetBalances()
  - GetTxStatus()                - GetHeaders()                - ResolveAlias()
  - GetUtxStream()               - GetBlock()                  - GetAccountStream()
```

### 1. `TransactionsApi`
Fornece streaming e injeção de transações no nó de forma assíncrona.
*   **`rpc Broadcast (SignedTransaction) returns (Transaction)`**: Envia uma transação assinada para validação e propagação na UTX pool e rede P2P.
*   **`rpc GetTransactionStatus (TransactionsByIdRequest) returns (stream TransactionStatus)`**: Retorna o status de confirmação (altura e se foi minerada) de transações em tempo real.
*   **`rpc GetUtxStream (Empty) returns (stream UtxEvent)`**: Fornece um fluxo de streaming de todos os eventos de transações adicionadas ou removidas da UTX Pool (mempool).

### 2. `BlocksApi`
Fornece dados brutos de blocos e cabeçalhos em formato unificado gRPC.
*   **`rpc GetBlockRange (BlockRangeRequest) returns (stream BlockWithStatus)`**: Faz o streaming de blocos serializados em uma janela de alturas informada.
*   **`rpc GetBlock (BlockRequest) returns (BlockWithStatus)`**: Retorna os detalhes de um bloco específico por assinatura ou altura.

### 3. `AccountsApi`
Responsável pelo monitoramento ativo de alterações de saldo e oráculos.
*   **`rpc GetBalances (BalancesRequest) returns (stream BalanceResponse)`**: Retorna e atualiza de forma incremental o saldo de uma lista de contas e ativos.
*   **`rpc GetAccountData (AccountDataRequest) returns (stream AccountDataResponse)`**: Monitora escritas e alterações de pares chave-valor (Data Entry) no RocksDB da conta.

---

## 🦊 4. Integração Criptográfica com a MetaMask (Ethereum RPC)

O AMZX Node possui compatibilidade integrada e de baixo nível com a máquina de assinaturas da rede Ethereum e com o aplicativo de carteira **MetaMask**, orquestrada pelo controlador `EthRpcRoute.scala`.

### A. Derivação do Chain ID Decimal
Ao contrário do Ethereum que possui IDs numéricos estáticos e registrados, o AMZX calcula o Chain ID dinamicamente em tempo de execução utilizando o byte do caractere ASCII configurado no parâmetro `address-scheme-character` (Magic Byte) da rede:
$$\text{Chain ID (Decimal)} = \text{ASCII}(\text{Magic Byte})$$

*   Se `address-scheme-character = "S"` (Stagenet): Código ASCII `83` (Hex `0x53`).
*   Se `address-scheme-character = "T"` (Testnet): Código ASCII `84` (Hex `0x54`).
*   Se `address-scheme-character = "C"` (Custom / Devnet): Código ASCII `67` (Hex `0x43`).
*   Na MetaMask, cadastre o RPC apontando para `http://localhost:6869/eth` e informe o respectivo ID decimal obtido da fórmula acima.

### B. Tradução e Multiplicador de Balanço (Satoshis vs Wei)
*   **O Desafio**: O token nativo blockchain possui **8 decimais** (onde $10^8$ Satoshis equivalem a 1 AMZX). No entanto, a MetaMask opera de forma fixa sobre o padrão do Ethereum de **18 decimais** (Wei).
*   **A Solução**: No endpoint de consulta `eth_getBalance`, o nó multiplica o saldo real retornado pelo RocksDB por um fator multiplicador estático de **$10^{10}$**:
    $$\text{Balance (Wei)} = \text{Balance (Satoshis)} \times 10^{10}$$
    Desta forma, a MetaMask exibe o saldo de forma legível e sem perda de precisão ou descompasso matemático.

### C. Mapeamento de Endereços Ethereum para AMZX
A MetaMask utiliza chaves elípticas **ECDSA secp256k1**, gerando um endereço público de 20 bytes (ex: `0x90F8bf...`). Para mapear esses fundos deterministicamente no ledger AMZX, o nó executa a rotina de hash Blake2b e Keccak256 sobre a chave pública e concatena o byte de rede (Chain ID).
*   Quando a MetaMask assina e envia uma transação via `eth_sendRawTransaction`, o nó intercepta o payload, extrai as provas `r`, `s` e `v`, recupera a chave pública ECDSA e reconstrói o endereço AMZX de destino para efetuar os débitos e execuções. Essa transação especial de compatibilidade é gravada on-chain como uma **`EthereumTransaction` (ID 18)**.

### D. Geração Dinâmica de ABI para Smart Contracts
Para invocar métodos de contratos inteligentes dApp escritos em RIDE utilizando a MetaMask, o nó disponibiliza um endpoint gerador de ABI automático:
*   **Endpoint**: `GET http://127.0.0.1:6869/eth/abi/{dAppAddress}`
*   **Funcionamento**: O nó lê o script atachado à conta dApp no RocksDB, inspeciona sua estrutura AST (Abstract Syntax Tree), identifica todas as funções anotadas com `@Callable` e seus respectivos parâmetros e cospe um JSON de ABI compatível com Solidity e Ethereum, pronto para ser lido pela biblioteca Web3.js de frontends.

---

## ✍️ 5. Mecânica de Smart Contracts em RIDE (Desenvolvimento)

A blockchain executa contratos inteligentes imutáveis e sem estado escritos em **RIDE**. 

```
                                  +---------------------------+
                                  |     RIDE Source Code      |
                                  +---------------------------+
                                                |
                                                | sbt compile
                                                v
                                  +---------------------------+
                                  |   AST Static Analysis     |
                                  | (Calcula complexidade max)|
                                  +---------------------------+
                                                |
                                                | Base64 Compile
                                                v
+-----------------------------+   SetScriptTx   +---------------------------+
|  RocksDB dApp Deployment    |<================|    SetScript (Type 13)    |
+-----------------------------+                 +---------------------------+
```

### A. Análise Estática de Complexidade (Sem Gás Dinâmico)
Diferente da EVM (Ethereum) que calcula gás consumido instrução por instrução de forma dinâmica durante o processamento do bloco, o compilador RIDE executa uma **Análise Estática de Complexidade** no código antes da compilação.
*   Cada função básica e operação matemática do RIDE possui uma pontuação de complexidade de execução estática predefinida.
*   A complexidade de uma função `@Callable` ou `@Verifier` é o somatório do caminho mais longo possível de execução de suas ramificações condicionais.
*   A rede impõe um teto de complexidade por transação de chamada (padrão: **4.000 pontos**). Se o código compilado exceder esse limite na análise de caminhos estáticos, ele é rejeitado já no envio da transação `SetScriptTransaction` (ID 13), impedindo que códigos infinitos ou ineficientes entrem on-chain e causem negação de serviço.

### B. Transação de Implantação (`SetScriptTransaction` - ID 13)
Utilizada para atachar um script RIDE em uma conta blockchain, transformando-a em uma Smart Account ou dApp ativo.
*   **Taxa Base**: `0.01 AMZX` (acrescido de `0.004 AMZX` por verificador ativo).
*   **Payload JSON**:
    ```json
    {
      "type": 13,
      "sender": "3MvYourAddressHere...",
      "script": "base64CompiledBinaryStringHere...",
      "fee": 1000000,
      "timestamp": 1780578324000,
      "proofs": ["signatureString..."]
    }
    ```

### C. Transação de Execução (`InvokeScriptTransaction` - ID 16)
Utilizada por usuários ou outros contratos para chamar funções `@Callable` públicas de um dApp.
*   **Taxa Base**: `0.005 AMZX` (se houver pagamentos ou verificação extra de script, soma-se `0.004 AMZX` de taxa extra de script).
*   **Payload JSON**:
    ```json
    {
      "type": 16,
      "sender": "3MvYourAddressHere...",
      "dApp": "3MvTargetDAppAddressHere...",
      "call": {
        "function": "depositarMoeda",
        "args": [
          { "type": "string", "value": "AMZX" }
        ]
      },
      "payment": [
        { "amount": 100000000, "assetId": null }
      ],
      "fee": 500000,
      "timestamp": 1780578324100,
      "proofs": ["signatureString..."]
    }
    ```

### D. Métodos de Entrada `@Verifier` e `@Callable`
*   **`@Verifier(tx)`**: Intercepta transações criadas pela própria conta. Retorna um valor booleano (`true` ou `false`). Substitui o validador de chave privada tradicional, permitindo programar travas temporais (timelocks) ou regras multi-assinatura complexas para aprovar pagamentos.
*   **`@Callable(invocante)`**: Funções acessíveis por agentes externos. Elas lêem dados da blockchain, computam regras de negócios e retornam uma coleção de mutações de estado (`DataEntry` para escrita no RocksDB) ou transferências físicas de moedas (`ScriptTransfer`).

---

## 🔨 6. Comandos de Empacotamento SBT

O **SBT (Simple Build Tool)** é a ferramenta oficial de compilação, teste e empacotamento do ecossistema. 

```bash
# Compilar todo o projeto do nó (sem rodar)
cd amzx/
sbt compile

# Compilar todo o projeto do Matcher (sem rodar)
cd matcher/
sbt compile

# Executar testes unitários do livro de ofertas
sbt "dex/testOnly com.wavesplatform.dex.market.OrderBookSpec"

# Gerar pacote de instalação Debian (.deb) pronto para produção
sbt node/debian:packageBin
sbt dex/debian:packageBin
```

---

## 📈 7. Otimizações de JVM (Java Virtual Machine) em Produção

Para suportar grandes picos de concorrência e processar milhares de requisições sem lentidão:

*   **ZGC (Z Garbage Collector)**: Configuração obrigatória no Matcher DEX para atingir latências de coleta de lixo abaixo de 1 milissegundo:
    ```bash
    -XX:+UseZGC -XX:ZAllocationSpikeTolerance=5
    ```
*   **G1GC**: Configuração recomendada para o Nó validador de blocos em servidores normais:
    ```bash
    -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+ParallelRefProcEnabled
    ```
*   **Alocação de Heap Estática**: Impede pausas de thread causadas pelo redimensionamento dinâmico de memória heap durante a validação de blocos concorrentes:
    ```bash
    -Xms4g -Xmx4g -XX:+AlwaysPreTouch
    ```
*   **Proteção Contra Travamentos por OOM**: Garante que o processo da JVM seja derrubado imediatamente caso ocorra um estouro inesperado de memória RAM, permitindo que orquestradores de infraestrutura (como Kubernetes ou `systemd`) efetuem a reinicialização e restauração segura da instância RocksDB sem corrupção de arquivos em disco:
    ```bash
    -XX:+ExitOnOutOfMemoryError
    ```
