# 📡 Arquitetura de Rede, Protobuf, Benchmarks e Dicionário de Classes AMZX

Este documento apresenta o mapeamento de baixo nível definitivo da camada de infraestrutura de rede, especificações binárias **Protobuf**, mecânica atômica de consenso, testes automatizados e o dicionário mestre contendo as principais classes de engenharia do **AMZX Node** e do **AMZX Matcher DEX**.

---

## 🧬 1. Especificação Detalhada de Protocol Buffers (Protobuf)

A blockchain AMZX adota Protocol Buffers v3 (**Protobuf**) como a linguagem de definição de interface (IDL) para serialização de transações, blocos, estados de saldo e chamadas gRPC de alta performance, reduzindo o tamanho de pacotes binários e otimizando o consumo de banda de rede P2P.

Os arquivos estão compilados de forma estática sob o namespace de serialização `com.wavesplatform.protobuf`. Abaixo estão os esquemas exaustivos de campo:

### A. Especificação do Bloco (`block.proto`)
Contém o leiaute binário de blocos e micro-blocos na rede:

```protobuf
syntax = "proto3";
package waves;
option java_package = "com.wavesplatform.protobuf.block";

import "waves/transaction.proto";

message Block {
    message Header {
        message ChallengedHeader {
            int64 base_target = 1;                   // Dificuldade da rede
            bytes generation_signature = 2;          // Assinatura geradora de consenso
            repeated uint32 feature_votes = 3;       // Votos de recursos do protocolo
            int64 timestamp = 4;                     // Timestamp UTC de forjamento
            bytes generator = 5;                     // Chave pública do nó minerador
            int64 reward_vote = 6;                   // Voto de emissão de recompensa
            bytes state_hash = 7;                    // Hash do estado RocksDB resultante
            bytes header_signature = 8;              // Assinatura digital do cabeçalho
            FinalizationVoting finalization_voting = 9; // Dados de votação finalizada
        }

        int32 chain_id = 1;                          // Byte de rede identificador ASCII
        bytes reference = 2;                         // Assinatura do bloco anterior (pai)
        int64 base_target = 3;                       // baseTarget de dificuldade atual
        bytes generation_signature = 4;              // Assinatura de consenso da curva
        repeated uint32 feature_votes = 5;           // Votação das Consensus Features
        int64 timestamp = 6;                         // Timestamp do bloco
        int32 version = 7;                           // Versão física (Sempre 5)
        bytes generator = 8;                         // Chave pública do minerador (32 bytes)
        int64 reward_vote = 9;                       // Desejo de recompensa do minerador
        bytes transactions_root = 10;                // Raiz da árvore Merkle de transações
        bytes state_hash = 11;                       // Hash SHA-256 do RocksDB após bloco
        ChallengedHeader challenged_header = 12;     // Cabeçalho de desafio para forks
        FinalizationVoting finalization_voting = 13; // Estrutura de finalidade de blocos
    }

    Header header = 1;                               // Estrutura de metadados
    bytes signature = 2;                             // Assinatura final do bloco inteiro
    repeated SignedTransaction transactions = 3;     // Vetor de transações embutidas
}

message MicroBlock {
    int32 version = 1;                               // Versão física do micro-bloco
    bytes reference = 2;                             // Assinatura do micro-bloco ou key block anterior
    bytes updated_block_signature = 3;               // Nova assinatura do bloco atualizado consolidado
    bytes sender_public_key = 4;                     // Chave pública do líder emissor
    repeated SignedTransaction transactions = 5;     // Transações adicionadas na UTX Pool
    bytes state_hash = 6;                            // Hash do estado resultante
    FinalizationVoting finalization_voting = 7;      // Votos de finalização
}
```

### B. Especificação de Transações (`transaction.proto`)
Define o invólucro unificado das transações (ID 1 ao 18) e suas provas:

```protobuf
syntax = "proto3";
package waves;
option java_package = "com.wavesplatform.protobuf.transaction";

message SignedTransaction {
    Transaction transaction = 1;                     // Detalhes lógicos da transação
    repeated bytes proofs = 2;                       // Vetor de assinaturas (Max 8 provas de 64 bytes)
}

message Transaction {
    int32 chain_id = 1;                              // Byte de rede do AMZX
    bytes sender_public_key = 2;                     // Chave pública do emissor (32 bytes)
    Amount fee = 3;                                  // Taxa de gas (montante e ativo)
    int64 timestamp = 4;                             // Timestamp UTC da criação
    int32 version = 5;                               // Versão lógica da transação
    
    # Campo dinâmico identificador do tipo lógico da transação
    oneof data {
        GenesisTransactionData genesis = 101;
        PaymentTransactionData payment = 102;
        IssueTransactionData issue = 103;
        TransferTransactionData transfer = 104;
        ReissueTransactionData reissue = 105;
        BurnTransactionData burn = 106;
        ExchangeTransactionData exchange = 107;
        LeaseTransactionData lease = 108;
        LeaseCancelTransactionData lease_cancel = 109;
        CreateAliasTransactionData create_alias = 110;
        MassTransferTransactionData mass_transfer = 111;
        DataTransactionData data_entries = 112;
        SetScriptTransactionData set_script = 113;
        SponsorFeeTransactionData sponsor_fee = 114;
        SetAssetScriptTransactionData set_asset_script = 115;
        InvokeScriptTransactionData invoke_script = 116;
        UpdateAssetInfoTransactionData update_asset_info = 117;
        bytes ethereum_transaction = 118;            // Bytes puros RLP da MetaMask
    }
}
```

---

## ⚡ 2. Mecânica Atômica de Consenso LPoS / FairPoS

O mecanismo de consenso **FairPoS** (Fair Leased Proof of Stake) dita a ordem de liderança e a geração de blocos de forma matemática não-linear, reduzindo privilégios desproporcionais de grandes pools e mantendo o tempo médio de emissão estável.

```
       [Ciclo de Vida do Consenso FairPoS / LPoS por Altura h]
=============================================================================
  1. Computação de Entropia:
     hitSource = vrfSign(minerPrivateKey, state_entropy_h-100)
     
  2. Geração de Hit Matemático:
     Hit = Hash(hitSource) transformado em inteiro de 64-bits sem sinal
     
  3. Cálculo de Block Delay (Atraso):
     T = C1 * ln(1 - C2 * ln(1 - Hit/MaxHit) / (baseTarget * GeneratingBalance))
     
  4. Agendamento do Forjamento:
     O MinerActor agenda o disparo de 'forge' para: Timestamp anterior + T
     
  5. Se relógio local atingir a janela e quórum P2P estiver ativo:
     - Monta Key Block (cabeçalho)
     - Propaga via P2P
     - Lidera rodada de emissão de MicroBlocks (Waves-NG)
=============================================================================
```

### A. O Algoritmo de Geração de Hit VRF (Verifiable Random Function)
O validador não pode prever ou fraudar a ordem de geração de blocos futuros. Isso é impedido por meio da prova criptográfica de aleatoriedade verificável:
1.  **Entrada de Entropia**: O nó lê os dados de consenso de 100 blocos atrás para obter a semente estável de entropia (`hitSource`).
2.  **Assinatura de Curva**: O minerador assina criptograficamente a semente utilizando sua chave privada e anexa o resultado no cabeçalho do bloco como `generationSignature`.
3.  **Computação do Hit**: O digest binário da assinatura passa por uma codificação e normalização para retornar um inteiro sem sinal de 64 bits distribuído uniformemente na faixa de $[0, 2^{64}-1]$.

### B. Fórmula Estrita de Block Delay (Atraso em Milissegundos)
O tempo exato de espera que o nó precisa cumprir antes de propagar um bloco novo é regido pela equação FairPoS:

$$T = C_1 \cdot \ln\left(1 - C_2 \cdot \ln\left(1 - \frac{\text{Hit}}{\text{MaxHit}}\right) \cdot \frac{1}{\text{baseTarget} \cdot \text{Balance}}\right)$$

Onde:
*   $T$: Tempo de espera obrigatório em milissegundos.
*   $C_1$: Constante de calibração temporal (padrão: `70.000`).
*   $C_2$: Parâmetro de decaimento não-linear (padrão: `5.000.000.000.000.000` / $5 \cdot 10^{15}$).
*   $\text{Hit}$: O valor pseudo-aleatório criptográfico computado.
*   $\text{MaxHit}$: Limite superior escalar ($2^{64}-1$).
*   $\text{baseTarget}$: O fator de ajuste de dificuldade dinâmica da rede (calculado a cada bloco para manter os blocos médios em 60 segundos).
*   $\text{Balance}$: Saldo gerador efetivo do nó (em Satoshis) derivado de moedas próprias mais arrendamentos (`LPoS`) recebidos ativos no RocksDB.

### C. Ajuste Dinâmico de Dificuldade (`baseTarget`)
A rede ajusta automaticamente a dificuldade analisando a janela de tempo real de criação dos últimos 3 blocos em relação ao tempo alvo ideal de 60 segundos:
*   Se o tempo real de geração foi menor que 60 segundos $\implies$ a dificuldade aumenta, reduzindo o valor de `baseTarget` em até $1\%$.
*   Se o tempo real de geração foi maior que 60 segundos $\implies$ a dificuldade diminui, aumentando o valor de `baseTarget` em até $1\%$.

---

## 🧱 3. Pipeline de Validação e Geração de Blocos

Para que um bloco seja adicionado ao ledger de forma definitiva, ele passa por uma validação estrita em três camadas pelas classes core do nó:

```
[Bloco Recebido da Rede P2P]
           |
           v
+=============================================================================+
| 1. VALIDAÇÃO STATELESS (Sem estado)                                         |
| - Classe: BlockValidator                                                   |
| - Checa integridade de assinaturas criptográficas do cabeçalho.             |
| - Valida tamanhos máximos físicos permitidos do bloco.                     |
| - Executa MerkleRoot hash de transações e confronta com o cabeçalho.        |
+=============================================================================+
           | (Aprovado)
           v
+=============================================================================+
| 2. VALIDAÇÃO DE CONSENSO                                                    |
| - Classe: PoSSelector                                                      |
| - Valida se o timestamp de forjamento respeita o atraso mínimo de delay (T) |
| - Valida se a assinatura de geração VRF do minerador é criptograficamente   |
|   coerente com a semente histórica.                                         |
| - Checa se o baseTarget de dificuldade obedece à variação de até 1%.        |
+=============================================================================+
           | (Aprovado)
           v
+=============================================================================+
| 3. VALIDAÇÃO STATEFUL (Com estado)                                          |
| - Classe: BlockchainUpdaterImpl / StateValidator                           |
| - Executa sequencialmente o pipeline de cada transação do bloco.           |
| - Roda o interpretador RIDE para Smart Accounts e Smart Assets.            |
| - Valida se o saldo e os nonces das contas estão corretos no RocksDB.       |
+=============================================================================+
           | (Aprovado)
           v
[Gravação Física no RocksDB (StateSnapshot) & Incremento de Altura h]
```

---

## 📡 4. Protocolo de Comunicação P2P (Camada Netty TCP)

A sincronização de rede entre os nós do ecossistema AMZX apoia-se em conexões persistentes TCP bidirecionais assíncronas de baixa latência controladas pelo framework de rede de alta performance **Netty**.

### A. Fluxo de Handshake de Conexão
Sempre que dois nós iniciam contato, eles transmitem um cabeçalho binário de identificação estruturado:
```
+------------------+-------------------+-----------------+--------------------+
| Magic (4 bytes)  | Versão (12 bytes) | Nome (30 bytes) | Nonce (8 bytes IP) |
| Ex: 0x12345678   |  e.g., 1.4.0      |  "amzx-node-01" |   0x90F8bf3254...  |
+------------------+-------------------+-----------------+--------------------+
```
*   **Magic Bytes**: Quatro bytes estáticos que determinam se a rede é de homologação ou produção, evitando que conexões da Testnet atinjam a Mainnet.
*   **Nonce**: Número aleatório gerado para evitar que o nó conecte-se de forma redundante a si mesmo através de loopbacks de IP.

### B. Catálogo de Mensagens Binárias P2P
As mensagens trafegam de forma assíncrona encapsuladas em pacotes de cabeçalho curto de tamanho de bytes seguido de payload:

*   **`GetPeers` / `Peers`**: Solicita e retorna a lista de IPs de conexões conhecidas na rede para autodescoberta dinâmica (*peer discovery*).
*   **`GetSignatures` / `Signatures`**: Disparada por nós que estão atrasados. Envia as últimas assinaturas de blocos conhecidas e retorna a lista sequencial de assinaturas de blocos subsequentes para iniciar a sincronização.
*   **`GetBlock` / `Block`**: Solicita o corpo completo de transações e cabeçalhos de um bloco de altura $h$ identificado por sua assinatura criptográfica exclusiva.
*   **`MicroBlockInv` / `MicroBlockRequest` / `MicroBlock`**: Mecânica do Waves-NG. O líder do round anuncia a ID de um novo micro-bloco de transações. Os nós validadores requisitam o payload físico e o processam imediatamente em memória cache de estado.

---

## 📊 5. Suíte de Micro-benchmarks JMH (Java Microbenchmark Harness)

O repositório em `/benchmark` utiliza a biblioteca oficial de auditoria industrial de performance **JMH** para realizar perfis micro-temporais com precisão científica de nanossegundos. 

Isso impede que otimizações de código gerem vazamentos silenciosos ou atrasos de processamento na blockchain:

*   **`CalculateDelayBenchmark.scala`**: Mede a velocidade de cálculo da fórmula exponencial do FairPoS na JVM. Garante que o cálculo matemático de atraso gaste menos de **100 microssegundos** por tentativa de bloco.
*   **`RocksDBWriterBenchmark.scala`**: Avalia o tempo de escrita em batch no banco de dados físico RocksDB sob picos intensos de transações simultâneas.
*   **`SigVerifyBenchmark.scala`**: Compara o tempo de validação de assinaturas de curvas elípticas. O benchmark certifica que a verificação de assinaturas Ed25519 (Curve25519) execute na margem de **80 a 120 microssegundos** na JVM de produção, enquanto a verificação secp256k1 via ECDSA roda em torno de **150 a 180 microssegundos**.
*   **`ScriptEvaluatorBenchmark.scala`**: Perfis de interpretação e execução de smart contracts RIDE, comparando tempos de avaliação das ASTs em diferentes níveis de complexidade funcionais.

---

## 🧪 6. Estrutura de Testes Automatizados (CI & Quality Assurance)

O ecossistema é blindado contra regressões de código ou furos lógicos por meio de três suítes de testes robustas executáveis via SBT:

### A. Testes Unitários (`node-tests`)
*   **Escopo**: Verificações locais sem estado de classes, funções utilitárias e regras criptográficas.
*   **Como Executar**:
    ```bash
    sbt node/test
    ```

### B. Testes de Integração de Rede Multi-nó (`node-it`)
*   **Escopo**: Executa simulações reais de redes ativas utilizando contêineres temporários do **Docker**. 
*   **Funcionamento (`MultiNodeSpec.scala`)**:
    *   O SBT inicializa automaticamente uma topologia de rede local com 4 a 10 contêineres Docker rodando instâncias de nós AMZX.
    *   O framework injeta transações falsas e simula cenários complexos como **partições de rede (netsplit)**, latência artificial de latência e forks paralelos coordenados para certificar-se de que a lógica de seleção de consenso de cadeia mais longa e o limite de 100 blocos de rollback comportam-se de forma resiliente.
*   **Como Executar**:
    ```bash
    sbt "node-it/testOnly com.wavesplatform.it.sync.ForksIntegrationSpec"
    ```

---

## 📖 7. Dicionário das Principais Classes do Sistema (Nó & Matcher)

Para facilitar a navegação do desenvolvedor que queira dar manutenção ou customizar o ecossistema, mapeamos os arquivos de engenharia fundamentais e suas responsabilidades arquiteturais:

### A. Core Scala Node (Blockchain)

#### 1. `BlockchainUpdaterImpl.scala`
*   **Package**: `com.wavesplatform.state`
*   **Função**: É o coração do ledger. Gerencia a escrita e o avanço físico do RocksDB, orquestra rollbacks seguros, faz o merge sequencial de novos blocos no estado consolidado e atualiza a altura $h$ da cadeia.

#### 2. `UtxPoolImpl.scala`
*   **Package**: `com.wavesplatform.utx`
*   **Função**: Gerencia a UTX Pool (Mempool de transações pendentes em memória). Valida transações de forma dinâmica em relação ao estado atual do banco, remove transações obsoletas e alimenta o MinerActor com as transações prontas para forjamento.

#### 3. `Miner.scala`
*   **Package**: `com.wavesplatform.mining`
*   **Função**: Ator Akka encarregado do agendamento de mineração baseada nas equações de tempo extraídas do `PoSCalculator`, monta os blocos gênesis ou blocos de rodadas comuns e transmite para a rede.

#### 4. `PoSSelector.scala`
*   **Package**: `com.wavesplatform.consensus`
*   **Função**: Validador de consenso estrito. Toda vez que um bloco novo chega via rede P2P, esta classe checa a assinatura do gerador VRF, confere o target de dificuldade dinâmica e assegura que o minerador cumpriu o atraso mínimo exigido de tempo.

#### 5. `PoSCalculator.scala`
*   **Package**: `com.wavesplatform.consensus`
*   **Função**: Fornece os algoritmos e equações de cálculo de delay FairPoS, LPoS e baseTarget.

#### 6. `Recipient.scala`
*   **Package**: `com.wavesplatform.account`
*   **Função**: Define a lógica e as constantes binárias de alocação de endereços de contas clássicas e apelidos (`Address` de 26 bytes e `Alias`).

#### 7. `ScriptEvaluator.scala`
*   **Package**: `com.wavesplatform.lang.v1.evaluator`
*   **Função**: O interpretador e executor de expressões RIDE. Avalia os métodos callable `@Callable` e verificadores `@Verifier` de forma isolada, gerando as mutações e regras de escrita que serão repassadas ao banco RocksDB.

#### 8. `EthRpcRoute.scala`
*   **Package**: `com.wavesplatform.api.http`
*   **Função**: Rota HTTP Akka-HTTP encarregada de interceptar métodos do ecossistema Ethereum (`POST /eth`), convertendo montantes em satoshis para Wei ($10^{10}$) e decodificando assinaturas de chaves públicas `secp256k1`.

---

### B. Core Scala Matcher (Matcher DEX)

#### 1. `OrderBook.scala`
*   **Package**: `com.wavesplatform.dex.market`
*   **Função**: Gerencia a memória de ofertas pendentes de um par específico (`AmountAsset`/`PriceAsset`). Utiliza coleções imutáveis **TreeMap** altamente eficientes ordenadas por preço para cruzar ofertas de compra e venda de forma ultra-rápida.

#### 2. `Matcher.scala`
*   **Package**: `com.wavesplatform.dex`
*   **Função**: Ator de inicialização principal da corretora Matcher DEX. Sobe as tabelas de estado, inicia canais gRPC de sincronia, gerencia as credenciais criptográficas e manipula o ciclo de vida do RocksDB de ordens locais.

#### 3. `OrderValidator.scala`
*   **Package**: `com.wavesplatform.dex.queue`
*   **Função**: Valida o estado de ordens pendentes criadas pelos clientes (checa assinaturas, tempos de expiração, taxas de mercado e fundos disponíveis) antes de inseri-las nas filas do order book.

#### 4. `DEXExtension.scala`
*   **Package**: `com.wavesplatform.dex.grpc.integration`
*   **Função**: A extensão física de gRPC injetada no Nó validador. Canaliza os eventos de alteração da blockchain em tempo real, saldos de contas modificados e transações da UTX Pool e transmite de forma direta para o Matcher DEX.
