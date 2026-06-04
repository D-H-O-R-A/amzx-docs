# 🧾 Catálogo de Transações, Layouts Binários e Pipeline de Validação

Este documento provê a especificação exaustiva de todos os tipos de transações que circulam na blockchain **AMZX** (ID 1 ao 18), detalhando o objetivo técnico de cada uma, sua estrutura binária, taxas de execução padrão e as classes do código-fonte encarregadas de realizar a validação estática e dinâmica antes da gravação on-chain.

---

## 🏗️ 1. O Pipeline de Validação de Transações

O processo de aceitação de uma transação pelo nó segue um fluxo rígido em duas fases fundamentais para garantir a segurança contra dados corrompidos, gasto duplo ou assinaturas forjadas.

```
       [Transação Recebida via API REST / Rede P2P]
                            |
                            v
             ===============================
             FASE 1: VALIDAÇÃO ESTÁTICA (Stateless)
             ===============================
             [TxValidator.scala & impl/*Validator.scala]
             - Checa tamanhos máximos de campos
             - Checa se valores de transferência são positivos (não-negativos)
             - Checa assinaturas criptográficas (Signature/Proofs)
                            |
         (Se Válida) ------+------ (Se Inválida: Rejeita e descarta)
                            |
                            v
             ===============================
             FASE 2: VALIDAÇÃO DINÂMICA (Stateful)
             ===============================
             [UTX Pool Actor & StateValidator / Diff]
             - Valida assinaturas contra estado de scripts de contas (RIDE)
             - Checa saldo operacional real do remetente
             - Checa conflito de double-spending concorrente
                            |
         (Se Válida) ------+------ (Se Inválida: Descarta da mempool)
                            |
                            v
             [Armazenamento na Mempool (UTX Pool)]
                            |
                            v
              [Inclusão em Bloco (MinerActor)]
                            |
                            v
                [Gravação Física no RocksDB]
```

---

## 📂 2. Catálogo Técnico Exaustivo de Transações (ID 1 ao 18)

Abaixo estão especificadas todas as transações nativas mapeadas no ecossistema AMZX, com suas respectivas constantes e classes de validação:

### ID 1: Genesis Transaction (`GenesisTransaction.scala`)
*   **Objetivo**: Criar moedas nativas AMZX e distribuí-las para contas fundadoras no bloco zero da blockchain. Só pode ser incluída no bloco gênesis.
*   **Layout Binário**: `Type Byte (1) | Timestamp (8) | Recipient (26 bytes) | Amount (8)`
*   **Taxa / Gas**: Isenta ($0$ taxas).
*   **Validator**: `impl/GenesisTxValidator.scala` (Valida se a transação está apenas na altura zero).

### ID 2: Payment Transaction (`PaymentTransaction.scala`)
*   **Objetivo**: Transferência simples de moedas nativas entre contas.
*   **Layout Binário**: `Type (2) | Sender PubKey (32) | Recipient (26) | Amount (8) | Fee (8) | Timestamp (8) | Signature (64)`
*   **Taxa / Gas**: $0.001$ AMZX.
*   **Validator**: `impl/PaymentTxValidator.scala`.
*   > [!NOTE]
    > Esta transação foi descontinuada na versão 1.x para dar lugar à transação de transferência flexível (ID 4), mas permanece no core para validação de dados históricos.

### ID 3: Issue Transaction (`IssueTransaction.scala`)
*   **Objetivo**: Criar e emitir um novo token customizado (Asset) na blockchain. O token pode ser reemitível ou não-reemitível e pode conter um script RIDE de validação embutido (Smart Asset).
*   **Layout Binário**: `Type (3) | Version | Chain ID | Sender PubKey | Name (Length + Bytes) | Description (Length + Bytes) | Total Quantity (8) | Decimals (1) | Reissuable (1) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $1.00$ AMZX (para tokens normais) ou $10.00$ AMZX (para Smart Assets).
*   **Validator**: `impl/IssueTxValidator.scala`.

### ID 4: Transfer Transaction (`TransferTransaction.scala`)
*   **Objetivo**: Transferir moedas AMZX ou qualquer outro token customizado de uma conta para outra.
*   **Layout Binário**: `Type (4) | Version | Sender PubKey | Asset ID (Optional) | Fee Asset ID (Optional) | Timestamp (8) | Amount (8) | Fee (8) | Recipient (26 ou Alias) | Attachment (Length + Bytes) | Proofs`
*   **Taxa / Gas**: $0.001$ AMZX ($0.005$ AMZX caso o token de origem possua script ativo).
*   **Validator**: `impl/TransferTxValidator.scala` (Garante que o anexo não estoure 140 bytes e montante $> 0$).

### ID 5: Reissue Transaction (`ReissueTransaction.scala`)
*   **Objetivo**: Emitir mais unidades de um token que foi configurado originalmente como reemitível.
*   **Layout Binário**: `Type (5) | Version | Chain ID | Sender PubKey | Asset ID (32) | Quantity (8) | Reissuable (1) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $1.00$ AMZX (ou $10.00$ se for Smart Asset).
*   **Validator**: `impl/ReissueTxValidator.scala`.

### ID 6: Burn Transaction (`BurnTransaction.scala`)
*   **Objetivo**: Destruir de forma permanente e irreversível uma quantidade específica de tokens, reduzindo o fornecimento circulante (*total supply*) do ativo.
*   **Layout Binário**: `Type (6) | Version | Chain ID | Sender PubKey | Asset ID (32) | Quantity (8) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.001$ AMZX.
*   **Validator**: `impl/BurnTxValidator.scala`.

### ID 7: Exchange Transaction (`ExchangeTransaction.scala`)
*   **Objetivo**: Registrar uma execução de cruzamento de ordem de compra e venda liquidada pelo Matcher DEX. Contém as ordens assinadas originais de compra e venda como payloads embutidos.
*   **Layout Binário**: `Type (7) | Version | Buy Order (RLP-like) | Sell Order (RLP-like) | Price (8) | Amount (8) | Buy Matcher Fee (8) | Sell Matcher Fee (8) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.003$ AMZX.
*   **Validator**: `impl/ExchangeTxValidator.scala`.

### ID 8: Lease Transaction (`LeaseTransaction.scala`)
*   **Objetivo**: Arrendar saldo de moedas AMZX para um nó validador para aumentar seu poder de mineração PoS sem transferir a custódia.
*   **Layout Binário**: `Type (8) | Version | Sender PubKey | Recipient (26) | Amount (8) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.001$ AMZX.
*   **Validator**: `impl/LeaseTxValidator.scala`.

### ID 9: Lease Cancel Transaction (`LeaseCancelTransaction.scala`)
*   **Objetivo**: Cancelar um arrendamento ativo, retornando o saldo efetivo de votação/mineração para a carteira de origem imediatamente.
*   **Layout Binário**: `Type (9) | Version | Chain ID | Sender PubKey | Lease ID (32) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.001$ AMZX.
*   **Validator**: `impl/LeaseCancelTxValidator.scala`.

### ID 10: Create Alias Transaction (`CreateAliasTransaction.scala`)
*   **Objetivo**: Registrar e associar uma string legível curta (Alias) ao endereço de 26 bytes do usuário (ex: `alias:R:diego`).
*   **Layout Binário**: `Type (10) | Version | Sender PubKey | Alias (Length + Bytes) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.001$ AMZX.
*   **Validator**: `impl/CreateAliasTxValidator.scala` (Valida comprimento do alias entre 4 e 30 caracteres válidos).

### ID 11: Mass Transfer Transaction (`MassTransferTransaction.scala`)
*   **Objetivo**: Realizar transferências em lote (enviar moedas ou tokens para até 100 endereços de destino diferentes em uma única transação, economizando taxas de rede e espaço de blocos).
*   **Layout Binário**: `Type (11) | Version | Sender PubKey | Asset ID (Optional) | Recipients Number (2) | [Recipient (26) + Amount (8)] * N | Fee (8) | Timestamp (8) | Attachment (Length + Bytes) | Proofs`
*   **Taxa / Gas**: Base de $0.001$ AMZX + $0.0005$ AMZX por cada endereço de destino incluído.
*   **Validator**: `impl/MassTransferTxValidator.scala`.

### ID 12: Data Transaction (`DataTransaction.scala`)
*   **Objetivo**: Gravar pares de chave-valor (metadados estruturados) diretamente no estado de armazenamento da conta do RocksDB. Essencial para dApps, perfis, oráculos e registros de dados immutable.
*   **Layout Binário**: `Type (12) | Version | Sender PubKey | Entries Number (2) | [Key (Length + Bytes) + Type (1) + Value (Variable)] * N | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: Base de $0.001$ AMZX + $0.001$ AMZX por cada megabyte de payload gravado.
*   **Validator**: `impl/DataTxValidator.scala` (Valida limite de 100 chaves por transação e tamanho máximo de 150KB de payload total).

### ID 13: Set Script Transaction (`SetScriptTransaction.scala`)
*   **Objetivo**: Compilar e atuar um script RIDE sobre a conta remetente, transformando-a em uma conta de contrato inteligente multifuncional (dApp) ou conta multifirmada (Multisig).
*   **Layout Binário**: `Type (13) | Version | Chain ID | Sender PubKey | Script (Optional, Length + Bytes) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.01$ AMZX.
*   **Validator**: `impl/SetScriptTxValidator.scala`.

### ID 14: Sponsor Fee Transaction (`SponsorFeeTransaction.scala`)
*   **Objetivo**: Habilitar ou desabilitar o patrocínio de taxas para um ativo próprio. Permite que usuários paguem as taxas das transações usando o seu token ao invés de AMZX (o patrocinador cobre a taxa AMZX correspondente em segundo plano).
*   **Layout Binário**: `Type (14) | Version | Chain ID | Sender PubKey | Asset ID (32) | Min Sponsored Fee (8) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $1.00$ AMZX (ou $10.00$ se for Smart Asset).
*   **Validator**: `impl/SponsorFeeTxValidator.scala`.

### ID 15: Set Asset Script Transaction (`SetAssetScriptTransaction.scala`)
*   **Objetivo**: Atualizar o script de validação de um Smart Asset. Só pode ser executada se o script original do token permitir modificações subsequentes.
*   **Layout Binário**: `Type (15) | Version | Chain ID | Sender PubKey | Asset ID (32) | Script (Length + Bytes) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $1.00$ AMZX (ou $10.00$ se for Smart Asset).
*   **Validator**: `impl/SetAssetScriptTxValidator.scala`.

### ID 16: Invoke Script Transaction (`InvokeScriptTransaction.scala`)
*   **Objetivo**: Chamar e executar métodos públicos `@Callable` de dApps implantados na blockchain, podendo anexar pagamentos múltiplos de moedas na chamada.
*   **Layout Binário**: `Type (16) | Version | Chain ID | Sender PubKey | dApp Address (26) | Function Call (Name + Arguments RLP-like) | Payments [Asset ID + Amount] * N | Fee (8) | Fee Asset ID (Optional) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.005$ AMZX.
*   **Validator**: `impl/InvokeScriptTxValidator.scala`.

### ID 17: Update Asset Info Transaction (`UpdateAssetInfoTransaction.scala`)
*   **Objetivo**: Atualizar metadados não estruturados de um token próprio (Nome e Descrição). Possui restrição de tempo mínimo entre alterações para evitar golpes de alteração rápida.
*   **Layout Binário**: `Type (17) | Version | Chain ID | Sender PubKey | Asset ID (32) | New Name (Length + Bytes) | New Description (Length + Bytes) | Fee (8) | Timestamp (8) | Proofs`
*   **Taxa / Gas**: $0.001$ AMZX (ou $0.005$ se for Smart Asset).
*   **Validator**: `impl/UpdateAssetInfoTxValidator.scala` (Verifica a janela de blocos mínima para atualização definida nas configurações).

### ID 18: Ethereum Transaction (`EthereumTransaction.scala`)
*   **Objetivo**: Transmitir chamadas, transferências de valor e invocações originadas de carteiras EVM compatíveis (como MetaMask) assinadas com chaves secp256k1 nativas.
*   **Layout Binário**: `Type (18) | Ethereum RLP Raw Encoded Bytes`
*   **Taxa / Gas**: Equivalente dinâmico ao gas limite convertido para a escala AMZX (mínimo de $0.005$ AMZX para chamadas de dApps).
*   **Validator**: `impl/CommitToGenerationTxValidator.scala` ou herança de validação Ethereum específica.
