# ⚙️ Configuração de uma Blockchain Customizada (AMZX Private Network)

O **AMZX Scala Node** permite a criação de redes blockchain totalmente independentes, privadas ou governamentais, através de parametrizações personalizadas. Este documento atua como o manual cirúrgico de engenharia para configurar os arquivos, estruturar o Bloco Gênesis, definir a distribuição de moedas iniciais, ativar recursos de rede e assinar o bloco gênesis de forma correta.

---

## 🔑 1. O Conceito de Magic Byte (Chain ID)

O **Chain ID** (ou caractere de esquema de endereço) é um único caractere ASCII de 1 byte que diferencia fisicamente a sua blockchain de qualquer outra. Ele é fundamental por dois motivos críticos:

1.  **Geração e Prefixos de Endereço**: A criptografia de chaves públicas gera endereços únicos que incorporam o byte da rede. Alterar o caractere muda completamente a representação textual de todos os endereços derivados da mesma chave pública.
2.  **Proteção contra Ataques de Replay (Replay Attack Prevention)**: Garante que transações assinadas para a sua blockchain customizada não possam ser interceptadas e retransmitidas em outras redes públicas ou de testes do ecossistema, pois o nó recusará a assinatura se o byte de rede embutido na transação divergir do seu próprio.

### Mapeamento ASCII Sugerido para AMZX:
*   **AMZX Mainnet Privada**: `'A'` (Código ASCII `65`) — Endereços tipicamente iniciam com `3M` ou `3N` dependendo da derivação.
*   **AMZX Testnet Privada**: `'a'` (Código ASCII `97`)
*   **AMZX Devnet Local**: `'D'` (Código ASCII `68`)

---

## 🧬 2. Dicionário Completo de Funcionalidades do Protocolo (Blockchain Features)

A rede possui uma arquitetura de ativação de recursos por votação ou pré-ativação manual chamada **Blockchain Features**. Em uma rede customizada (`CUSTOM`), podemos definir a ativação imediata de todos os recursos do protocolo na altura zero (`height = 0`) para obter uma blockchain moderna e completa desde o primeiro bloco.

Abaixo está o dicionário completo de todas as 25 features implementadas no núcleo do nó:

| ID | Nome Técnico da Feature | Descrição do Protocolo | Impacto de Consenso / Lógica de Programação |
| :---: | :--- | :--- | :--- |
| **1** | `SmallerMinimalGeneratingBalance` | Minimum Generating Balance of 1000 WAVES | Reduz o saldo gerador necessário para mineração de 10.000 para 1.000 moedas, facilitando a descentralização de validadores. |
| **2** | `NG` | NG Protocol | Ativa o protocolo de consenso acelerado Waves-NG (Key Blocks e Micro Blocks rápidos de 2s), acabando com os tempos longos de blocos estáticos. |
| **3** | `MassTransfer` | Mass Transfer Transaction | Habilita a transação Tipo 11 (`MassTransfer`), permitindo pagamentos em lote de até 100 destinatários de uma única vez com taxas super otimizadas. |
| **4** | `SmartAccounts` | Smart Accounts | Permite anexar scripts de validação RIDE a contas normais, tornando-as contas inteligentes controladas por códigos, oráculos e multi-signatures. |
| **5** | `DataTransaction` | Data Transaction | Habilita a transação Tipo 12 (`DataTransaction`), permitindo que contas salvem metadados e pares chave-valor em seu banco de dados on-chain. |
| **6** | `BurnAnyTokens` | Burn Any Tokens | Permite que qualquer usuário queime voluntariamente tokens customizados de sua propriedade utilizando a transação Tipo 6 (`BurnTransaction`). |
| **7** | `FeeSponsorship` | Fee Sponsorship | Habilita o patrocínio de taxas via transação Tipo 14. Permite que criadores de tokens paguem as taxas da blockchain de seus usuários usando seus próprios tokens. |
| **8** | `FairPoS` | Fair PoS | Substitui o algoritmo antigo Nxt PoS pelo moderno FairPoS, introduzindo uma barreira contra o acúmulo desproporcional de delay de blocos. |
| **9** | `SmartAssets` | Smart Assets | Permite anexar scripts RIDE a tokens criados (Tipo 15). Um token inteligente pode impor regras de conformidade e restrições de transferência. |
| **10** | `SmartAccountTrading` | Smart Account Trading | Permite que contas inteligentes (Smart Accounts) e dApps assinem e submetam ordens de compra e venda diretamente na corretora descentralizada (DEX). |
| **11** | `Ride4DApps` | RIDE 4 DAPPS | Ativa a especificação RIDE v3, introduzindo o conceito de dApps com funções públicas invocáveis (`@Callable`) e funções de verificação (`@Verifier`). |
| **12** | `OrderV3` | Order Version 3 | Habilita o formato de ordens Versão 3 na DEX Matcher, suportando regras modernas de taxas em múltiplos ativos ou ativos nativos. |
| **13** | `ReduceNFTFee` | Reduce NFT fee | Reduz de forma drástica a taxa de emissão para tokens não-fungíveis (NFTs) definidos com quantidade = 1, casas decimais = 0 e não-reemissíveis. |
| **14** | `BlockReward` | Block Reward and Community Driven Policy | Ativa a emissão inflacionária dinâmica e recompensas de bloco por voto dos mineradores. Permite que a rede decida se aumenta ou diminui o suprimento. |
| **15** | `BlockV5` | Ride V4, VRF, Protobuf, Failed transactions | O divisor de águas: introduz RIDE v4, geração de hits via VRF (imunidade total a Grinding Attacks), suporte a Protobuf em blocos e salvamento de transações falhas. |
| **16** | `SynchronousCalls` | Ride V5, dApp-to-dApp invocations | Habilita RIDE v5. Permite que dApps chamem funções públicas de outros dApps em tempo de execução de forma síncrona (chamada reentrante segura). |
| **17** | `RideV6` | Ride V6, MetaMask support | Ativa a prorrogação de complexidade do RIDE v6 e introduz o ecossistema de compatibilidade para carteiras Web3 líderes de mercado (**MetaMask**). |
| **18** | `ConsensusImprovements` | Consensus and MetaMask updates | Aplica melhorias no tempo de propagação de consensos e corrige refinamentos de limites na compatibilidade das assinaturas ECDSA da MetaMask. |
| **19** | `BlockRewardDistribution` | Block Reward Distribution | Permite a divisão automática das recompensas dos mineradores para múltiplos endereços secundários predefinidos nas configurações do nó. |
| **20** | `CappedReward` | Capped XTN buy-back & DAO amounts | Define limites estritos sobre taxas de recompras e alocações de subsídios para carteiras de fomento da DAO de governança da rede. |
| **21** | `CeaseXtnBuyback` | Cease XTN buy-back | Encerra definitivamente as rotinas automáticas on-chain de reajustes e recompras da antiga stablecoin de referência. |
| **22** | `LightNode` | Light Node | Ativa o modo de nó leve, onde o RocksDB otimiza as snapshots de transações e melhora a eficiência de cache para nós que rodam em servidores compactos. |
| **23** | `BoostBlockReward` | Boost Block Reward | Habilita incrementos temporários para incentivar os nós validadores em fases específicas de fomento da rede. |
| **24** | `EcrecoverFix` | ecrecover fix | Corrige bugs na verificação nativa RIDE da função de criptografia `ecrecover`, blindando a rede contra ataques de assinaturas forjadas. |
| **25** | `DeterministicFinality`| Deterministic Finality & RIDE V9 | Ativa a finalização determinística de blocos e libera a especificação ultra-moderna da linguagem RIDE V9. |

---

## 📝 3. Estrutura do Arquivo de Configuração (`application.conf`)

Toda a definição técnica da sua rede reside nas configurações estruturadas em formato **HOCON** (Human-Optimized Config Object Notation). O arquivo de configuração mestre padrão do nó validador localiza-se em `waves/node/src/main/resources/application.conf` (podendo ser estendido por arquivos como `devnet.conf` ou passados via argumento de execução).

Para ativar uma rede customizada independente, ajuste a seção `waves.blockchain` conforme o modelo completo abaixo:

```hocon
waves {
  # Diretório físico no disco para salvar os dados
  directory = "/var/lib/waves"

  blockchain {
    # Modo de Rede: MAINNET, TESTNET, STAGENET ou CUSTOM
    type = CUSTOM
    
    custom {
      # O Magic Byte (Chain ID) que identificará sua blockchain customizada
      address-scheme-character = "A"
      
      # Parâmetros de Funcionalidades (Pre-activated features)
      functionality {
        feature-check-blocks-period = 500
        blocks-for-feature-activation = 400
        
        # Ativação imediata das features na altura 0
        pre-activated-features {
          1 = 0   # Menor saldo gerador
          2 = 0   # Waves-NG protocol
          3 = 0   # Mass Transfer
          4 = 0   # Smart Accounts
          5 = 0   # Data Transaction
          6 = 0   # Burn Tokens
          7 = 0   # Fee Sponsorship
          8 = 0   # FairPoS
          9 = 0   # Smart Assets
          10 = 0  # Smart Trading
          11 = 0  # Ride v3 (dApps)
          12 = 0  # Order v3
          13 = 0  # Reduzir Taxa de NFT
          14 = 0  # Block Reward
          15 = 0  # Block v5 (VRF, Protobuf, Failed txs)
          16 = 0  # Ride v5 (dApp síncrono)
          17 = 0  # Ride v6 (MetaMask / secp256k1)
          18 = 0  # Consensus Improvements
          19 = 0  # Block Reward Distribution
          20 = 0  # Capped Reward
          21 = 0  # Cease XTN Buyback
          22 = 0  # Light Node
          23 = 0  # Boost Reward
          24 = 0  # Ecrecover fix
          25 = 0  # Deterministic Finality
        }
        
        min-block-time = 5s
        max-transaction-time-back-offset = 120m
        max-transaction-time-forward-offset = 90m
      }
      
      # Definição do Bloco Gênesis (O Bloco Zero da Rede)
      genesis {
        # Tempo ideal médio entre blocos gerados por LPoS
        average-block-delay = 10s
        
        # Parâmetro de dificuldade inicial
        initial-base-target = 15000
        
        # Suprimento Total da Moeda Base em Satoshis.
        # O token nativo tem 8 casas decimais. Portanto, 1 AMZX = 10^8 satoshis.
        # Para emitir 100.000.000 (100 Milhões) de moedas iniciais:
        # 100.000.000 * 10^8 = 10.000.000.000.000.000 Satoshis
        initial-balance = 10000000000000000
        
        # Endereço do gerador do bloco gênesis (Endereço Administrador gerado sob a rede 'A')
        generator = "3MyA947A11y7xskpP8..."
        
        # Data de criação da rede em milissegundos unix timestamp
        timestamp = 1780000000000
        block-timestamp = 1780000000000
        
        # Assinatura criptográfica do bloco gênesis (Veja a seção 4 para saber como calcular de forma automática!)
        signature = "5FYourGenesisSignatureHere..."
        
        # Distribuição de moedas inicial pelas transações de Gênesis
        # A soma dos amounts de todas as transações DEVE ser exatamente igual ao 'initial-balance' configurado acima.
        transactions = [
          { recipient = "3MyA947A11y7xskpP8...", amount = 5000000000000000 }, # 50 Milhões de moedas
          { recipient = "3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF", amount = 5000000000000000 }  # 50 Milhões de moedas
        ]
      }
      
      # Recompensas por Bloco Gerado
      rewards {
        term = 100000
        initial = 600000000  # Recompensa inicial de 6 moedas por bloco minerado
        min-increment = 50000000
        voting-interval = 10000
      }
    }
  }
}
```

---

## 🛠️ 4. Guia Definitivo para Geração de Gênesis Customizado

O bloco gênesis é assinado digitalmente para garantir que o conteúdo das transações de distribuição de moedas iniciais, o timestamp e o endereço gerador não sejam alterados. Qualquer modificação nestes parâmetros invalidará a assinatura, e o nó se recusará a iniciar.

Existem duas formas técnicas homologadas de calcular e aplicar essa assinatura para criar sua rede privada do zero.

### Método A: Utilizando a ferramenta CLI integrada `GenesisBlockGenerator`

A forma mais elegante e profissional é utilizar a ferramenta geradora nativa empacotada no executável da blockchain.

1.  **Crie o arquivo de entrada do gerador**:
    Crie um arquivo chamado `genesis-generator.conf` na raiz com o seguinte layout:
    ```hocon
    genesis-generator {
      networkType = "Custom"
      averageBlockDelay = 10s
      minBlockTime = 5s
      preActivatedFeatures = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 22, 24, 25]
      
      distributions = [
        {
          seedText = "amzblockchain seed de mineracao e distribuicao inicial node 1"
          amount = 5000000000000000  # 50.000.000 moedas em Satoshis
          miner = true
        },
        {
          seedText = "amzblockchain seed de fundacao e carteira corporativa node 2"
          amount = 5000000000000000  # 50.000.000 moedas em Satoshis
          miner = true
        }
      ]
    }
    ```

2.  **Execute o Gerador via SBT ou Fat JAR**:
    Se você estiver em ambiente de desenvolvimento, execute via SBT:
    ```bash
    JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 sbt "generateGenesis genesis-generator.conf"
    ```
    Ou chame-o diretamente de forma rápida utilizando o Fat JAR compilado:
    ```bash
    java -cp waves/node/target/waves-all-1.6.3.jar com.wavesplatform.GenesisBlockGenerator genesis-generator.conf
    ```

3.  **Colete as informações do console**:
    O gerador exibirá no terminal a semente secreta do seu novo administrador, chaves privadas, chaves públicas, endereços de rede customizados calculados sob a rede `'A'` e, mais importante, o bloco HOCON formatado contendo a assinatura correta calculada:
    ```text
    Settings:
    genesis {
      average-block-delay = 10000ms
      initial-base-target = 15000
      timestamp = 1780000000000
      block-timestamp = 1780000000000
      signature = "4WNhR15xizRv3uYwaXgtMzuvQ8ukKKimoDSp19ms3W3UU9XHgFo8q5jgcnWYKkJ7qtam9g5i5kLr6g38iJe8Fdgh"
      initial-balance = 10000000000000000
      transactions = [
        {recipient = "3MyA947A11y7xskpP8...", amount = 5000000000000000},
        {recipient = "3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF", amount = 5000000000000000}
      ]
    }
    ```

4.  **Atualize seu `application.conf`**:
    Copie o bloco gerado acima e substitua-o literal e integralmente na seção `custom.genesis` do seu arquivo de configurações principal.

---

### Método B: O Truque Assistido do Nó (Assistência por Falha Criptográfica)

Se você não quer estruturar um arquivo gerador separado, o próprio mecanismo de validação do nó validador pode calcular a assinatura de forma automática!

1.  **Edite o `application.conf`**: Defina os destinatários, amounts, timestamp e o caractere de rede que você deseja para a sua blockchain customizada.
2.  **Forneça uma assinatura inválida**: No campo `signature` do bloco `genesis`, insira qualquer string arbitrária curta (ex: `"test"` ou `"invalid"`).
3.  **Execute a inicialização do nó**:
    ```bash
    java -jar waves/node/target/waves-all-1.6.3.jar application.conf
    ```
4.  **Capture a assinatura esperada nos logs**:
    O nó falhará na inicialização ao detectar que a assinatura declarada `"invalid"` não corresponde à matemática dos blocos. Ele imprimirá no log a assinatura correta esperada para os seus dados:
    ```text
    2026-06-04 13:00:12,453 [main] ERROR c.w.s.WavesSettings$ - Genesis block signature is invalid!
    2026-06-04 13:00:12,454 [main] ERROR c.w.s.WavesSettings$ - Expected signature: 4WNhR15xizRv3uYwaXgtMzuvQ8ukKKimoDSp19ms3W3UU9XHgFo8q5jgcnWYKkJ7qtam9g5i5kLr6g38iJe8Fdgh
    2026-06-04 13:00:12,455 [main] ERROR c.w.s.WavesSettings$ - Please update 'genesis.signature' in your config file with the expected signature.
    ```
5.  **Aplique a assinatura correta**: Copie a assinatura informada em `Expected signature` e cole-a no campo `signature`.
6.  **Limpe o banco de dados local antigo** (deletando a pasta de dados) para evitar conflitos de cache e reinicie o nó. Ele inicializará o gênesis com sucesso absoluto!
