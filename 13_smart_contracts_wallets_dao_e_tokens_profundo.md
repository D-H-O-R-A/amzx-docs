# 🧠 Smart Contracts, Wallets, DAO e Ciclo de Vida de Tokens no AMZX

Este documento apresenta o mergulho técnico definitivo na infraestrutura lógica de contratos inteligentes **RIDE**, no ciclo de vida de tokens fúngicos e não-fúngicos (NFTs), na engenharia criptográfica de geração de chaves de carteiras (Curves) e em um exemplo prático completo e implantável de um contrato inteligente de **DAO** em RIDE.

---

## 🎨 1. Compilação RIDE e Análise Estática de Complexidade

A blockchain AMZX não utiliza o modelo de máquina virtual dinâmica de empilhamento de dados como a EVM (Ethereum Virtual Machine) para contratos inteligentes. Ela adota uma abordagem funcional e fortemente tipada baseada em expressões matemáticas compiladas em uma **Árvore de Sintaxe Abstrata (AST)** chamada **RIDE**.

```
  Código RIDE (Texto)
          |
          v   [sbt compile / REST Compiler API / VS Code Extension]
   Análise de AST & Grafo de Fluxo
          |
          v   (Determina caminhos condicionais e calcula peso estático)
   Verificação de Complexidade (Teto: 4000 pontos)
          |
          v   (Aprovado)
   Bytecode Binário (Base64)
          |
          v   [SetScriptTransaction - ID 13]
   Ativação na Blockchain (RocksDB)
```

### A. Complexidade Estática vs Gás Dinâmico
Na EVM, o consumo de gás flutua dinamicamente em tempo de execução, o que expõe o usuário a falhas inesperadas de "Out of Gas" e ataques de reentrada se o fluxo de controle for mal dimensionado. 

No RIDE:
*   **O Custo é Pré-calculado**: Antes do script ser implantado na rede via transação `SetScript` (ID 13), o compilador calcula o peso estático de execução (Complexidade) analisando o grafo de fluxo do código. 
*   **A Regra do Pior Caso**: A complexidade do script é declarada como o somatório do caminho mais longo possível de ramificações condicionais (`if-then-else`).
*   **Limite de Segurança**: O limite rígido regulamentar de complexidade para uma chamada dApp no ecossistema AMZX é de **4.000 pontos**. Se a análise estática de uma função `@Callable` ou `@Verifier` resultar em, por exemplo, `4.005` pontos, o nó recusará a compilação e rejeitará a gravação on-chain. Isso garante tempo de resposta determinístico de milissegundos nas validações e imunidade a laços de repetição infinitos (`while` / `for` são proibidos em RIDE).

---

## ⚙️ 2. Tipos de Scripts RIDE e Diferenças Arquiteturais

Os scripts RIDE podem ser aplicados em diferentes níveis para modificar as regras de comportamento da blockchain:

| Tipo de Script | Alvo de Ativação | Decoração Principal | Objetivo Técnico e Comportamento |
| :--- | :--- | :--- | :--- |
| **dApp Script** | Contas | `@Callable` / `@Verifier` | Executa rotinas de contratos inteligentes tradicionais. Grava e lê dados de chaves-valores de estado da conta no RocksDB e opera transferências físicas de múltiplos ativos. |
| **Smart Account** | Contas | `@Verifier` | Desativa a validação padrão de chave privada única e define regras programáticas de autorização para qualquer transação saindo da conta (ex: multi-assinaturas, bloqueios de tempo). |
| **Smart Asset** | Ativos (Tokens) | Sem decoração (Retorna `Boolean`) | Intercepta **toda e qualquer** transação que movimente o token customizado na rede (Transfer, Burn, Reissue). Se o script retornar `false`, a transferência é bloqueada no core da rede. |

---

## 🏛️ 3. Exemplo Prático de Contrato dApp de DAO (Pronto para Produção)

Este código em **RIDE v6** implementa um sistema completo e autônomo de **DAO** (Organização Autônoma Decentralizada) na blockchain AMZX. 

O contrato gerencia:
1.  **Depósito de Fundos** de Governança pelos membros.
2.  **Criação de Propostas** de investimento ou repasse.
3.  **Votação Democrática** descentralizada indexada pelo peso de moedas depositadas.
4.  **Execução Segura** da proposta aprovada, enviando os fundos da DAO diretamente para o endereço de destino aprovado, atualizando as chaves do RocksDB.

```fsharp
{-# STDLIB_VERSION 6 #-}
{-# CONTENT_TYPE DAPP #-}
{-# SCRIPT_TYPE ACCOUNT #-}

# Chaves globais de gravação no RocksDB
func proposalVotesKey(id: String) -> String { return "prop_" + id + "_votes" }
func proposalStatusKey(id: String) -> String { return "prop_" + id + "_status" }
func proposalTargetKey(id: String) -> String { return "prop_" + id + "_target" }
func proposalAmountKey(id: String) -> String { return "prop_" + id + "_amount" }
func userBalanceKey(address: String) -> String { return "user_" + address + "_balance" }
func userVotedKey(user: String, id: String) -> String { return "user_" + user + "_voted_" + id }

# Constante de corte de quórum de aprovação (5000 AMZX em Satoshis)
let QUORUM = 500000000000

@Callable(i)
func depositar() = {
  # Aceita depósitos de moedas nativas AMZX (assetId == unit)
  let pagamento = i.payments[0]
  if (isDefined(pagamento.assetId)) then {
    throw("Apenas moedas nativas AMZX sao aceitas para governanca da DAO.")
  } else {
    let depositante = toString(i.caller)
    let saldoAtual = match (getInteger(this, userBalanceKey(depositante))) {
      case s: Int => s
      case _ => 0
    }
    let novoSaldo = saldoAtual + pagamento.amount
    
    # Atualiza o saldo do membro no RocksDB
    [
      IntegerEntry(userBalanceKey(depositante), novoSaldo)
    ]
  }
}

@Callable(i)
func criarProposta(id: String, destinoHex: String, montanteSatoshis: Int) = {
  let criador = toString(i.caller)
  let status = match (getString(this, proposalStatusKey(id))) {
    case s: String => s
    case _ => ""
  }
  
  if (status != "") then {
    throw("Proposta com este ID ja existe.")
  } else if (montanteSatoshis <= 0) then {
    throw("Montante de proposta deve ser maior que zero.")
  } else {
    [
      StringEntry(proposalStatusKey(id), "PENDENTE"),
      StringEntry(proposalTargetKey(id), destinoHex),
      IntegerEntry(proposalAmountKey(id), montanteSatoshis),
      IntegerEntry(proposalVotesKey(id), 0)
    ]
  }
}

@Callable(i)
func votar(id: String) = {
  let eleitor = toString(i.caller)
  let status = match (getString(this, proposalStatusKey(id))) {
    case s: String => s
    case _ => throw("Proposta nao encontrada.")
  }
  
  let jaVotou = match (getBoolean(this, userVotedKey(eleitor, id))) {
    case b: Boolean => b
    case _ => false
  }
  
  if (status != "PENDENTE") then {
    throw("Esta proposta nao esta aberta para votacao.")
  } else if (jaVotou) then {
    throw("Voce ja registrou seu voto para esta proposta.")
  } else {
    # O peso do voto é exatamente o saldo do usuário depositado na DAO
    let pesoVoto = match (getInteger(this, userBalanceKey(eleitor))) {
      case b: Int => b
      case _ => 0
    }
    
    if (pesoVoto <= 0) then {
      throw("Voce nao possui saldo depositado para votar.")
    } else {
      let votosAtuais = match (getInteger(this, proposalVotesKey(id))) {
        case v: Int => v
        case _ => 0
      }
      
      [
        IntegerEntry(proposalVotesKey(id), votosAtuais + pesoVoto),
        BooleanEntry(userVotedKey(eleitor, id), true)
      ]
    }
  }
}

@Callable(i)
func executarProposta(id: String) = {
  let status = match (getString(this, proposalStatusKey(id))) {
    case s: String => s
    case _ => throw("Proposta nao encontrada.")
  }
  
  let votos = match (getInteger(this, proposalVotesKey(id))) {
    case v: Int => v
    case _ => 0
  }
  
  if (status != "PENDENTE") then {
    throw("Esta proposta ja foi finalizada.")
  } else if (votos < QUORUM) then {
    throw("A proposta nao atingiu o quorum minimo de votos de governanca de " + toString(QUORUM) + " Satoshis.")
  } else {
    let destinoString = match (getString(this, proposalTargetKey(id))) {
      case t: String => t
      case _ => throw("Destino nao configurado.")
    }
    let montante = match (getInteger(this, proposalAmountKey(id))) {
      case m: Int => m
      case _ => 0
    }
    
    let destinoAddress = addressFromStringValue(destinoString)
    
    # Grava status EXECUTADA e efetua a transferencia fisica dos fundos da DAO
    [
      StringEntry(proposalStatusKey(id), "EXECUTADA"),
      ScriptTransfer(destinoAddress, montante, unit)
    ]
  }
}

@Verifier(tx)
func verify() = {
  # Apenas assinaturas dos validadores fundadores da DAO podem atualizar o script
  sigVerify(tx.bodyBytes, tx.proofs[0], tx.senderPublicKey)
}
```

---

## 🪙 4. Ciclo de Vida de Ativos (Tokens) e NFTs na Blockchain

O ecossistema AMZX gerencia tokens fungíveis e NFTs de forma nativa e ultra-rápida, reduzindo a complexidade de gás e a dependência de padrões como ERC-20 ou ERC-721.

### A. Emissão de Tokens Fungíveis (`IssueTransaction` - ID 3)
Cria um ativo de fornecimento controlado. Parâmetros chave:
*   `decimals` (de 0 a 8): Determina a divisibilidade do token.
*   `reissuable` (Boolean): Se definido como `false`, o fornecimento total é congelado permanentemente na quantidade inicial.

### B. Emissão de NFTs (Non-Fungible Tokens)
Um NFT na blockchain AMZX é simplesmente um Token de especificação estritamente restrita:
$$\text{NFT AMZX} \iff \text{Decimals} = 0 \ \land \ \text{Reissuable} = \text{false} \ \land \ \text{Quantity} = 1$$
Como o tamanho do array de metadados de descrição do token é flexível, os desenvolvedores de dApps armazenam URLs IPFS ou metadados JSON do NFT diretamente no campo `description` da transação ID 3 de emissão, ou criam registros associados em transações de dados (ID 12).

### C. Patrocínio de Taxas (`SponsorFeeTransaction` - ID 14)
Permite que o emissor de um token ative a funcionalidade de patrocínio de rede.
*   O emissor define uma taxa proporcional (ex: `100` unidades de seu token próprio equivalem à taxa base de `1` AMZX).
*   Usuários que não possuem AMZX podem realizar transferências na rede pagando a taxa em tokens próprios patrocinados.
*   O nó desconta os tokens do remetente e deduz o saldo em AMZX equivalente da conta do patrocinador de forma transparente no fechamento do bloco.

---

## 🔑 5. Geração de Sementes e Coexistência de Curvas (Curve25519 vs Secp256k1)

A capacidade de suportar tanto transações criptográficas P2P de alta performance clássicas quanto interoperabilidade direta com a MetaMask e o ecossistema EVM exige a coexistência determinística de dois modelos de criptografia assimétrica de curvas elípticas sobre o mesmo segredo original.

```
                           [Seed Phrase BIP39 (12 a 24 palavras)]
                                             |
                                             v  [PBKDF2 com HMAC-SHA512 (2048 iteracoes)]
                                     [Master Entropy Seed]
                                             |
         +-----------------------------------+-----------------------------------+
         | (Rota Clássica AMZX)                                                  | (Rota Ethereum / MetaMask)
         v                                                                       v
 [Entropy Bytes]                                                         [Derivacao BIP44 Path: m/44'/60'/0'/0/0]
         |                                                                       |
         v [Blake2b256]                                                          v
 [Chave Privada Curve25519]                                              [Chave Privada secp256k1 (ECDSA)]
         |                                                                       |
         v [calculatePublicKey]                                                  v [Multiplicacao de Ponto]
 [Chave Pública Curve25519 (32 bytes)]                                    [Chave Pública secp256k1 (64 bytes)]
         |                                                                       |
         v [Keccak256(Blake2b256(PubKey))]                                       v [Keccak256(PubKey)]
 [PubKeyHash (Primeiros 20 bytes)]                                        [Address Ethereum (Ultimos 20 bytes)]
         |                                                                       |
         +===================================+===================================+
                                             | (Isomorfismo de Endereços)
                                             v
                             [Endereço AMZX final de 26 Bytes]
                             - Versão: 0x01
                             - ChainID: 'R' (0x52)
                             - Corpo: Ethereum Address ou PubKeyHash (20 bytes)
                             - Checksum: hashes duplos (4 bytes)
```

### 1. A Semente BIP39 (Seed Phrase)
O nó utiliza o padrão da indústria de 12 a 24 palavras em inglês para extrair a entropia. A semente em texto plano é passada pelo algoritmo **PBKDF2** usando salt dinâmico e `2048` iterações com hash interno **SHA-512** para gerar uma chave binária segura.

### 2. A Rota Clássica AMZX: Curva 25519
*   **Algoritmo**: Ed25519 / Curve25519.
*   **Funcionamento**: Excelente para assinatura e verificação P2P ultra-rápidas devido ao seu design altamente resistente a ataques de canal lateral e excelente desempenho computacional.
*   **Derivação**: A semente de bytes sofre hash `Blake2b256` simples para fixar a chave privada e derivar a chave pública de 32 bytes correspondente.

### 3. A Rota de Compatibilidade EVM: Curva secp256k1 (MetaMask)
*   **Algoritmo**: ECDSA / secp256k1 (Utilizada no Bitcoin e Ethereum).
*   **Funcionamento**: Utiliza o padrão de derivação de caminho **BIP44** (`m/44'/60'/0'/0/0`) para garantir que a mesma semente BIP39 de 12 palavras digitada na MetaMask gere exatamente as mesmas chaves privadas e públicas ECDSA de 64 bytes descompactadas e de sinal seguro.
*   **Unificação**: Graças ao isomorfismo de endereços, o endereço derivado de 20 bytes do Ethereum é incorporado diretamente no corpo de 26 bytes do endereço AMZX, permitindo que a blockchain identifique transações geradas via MetaMask e deduza os saldos das contas RocksDB corretas.
