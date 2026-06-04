# 🌐 Guia de Referência Completa da API REST Full do Ecossistema AMZX

Este documento fornece a especificação técnica exaustiva e de nível de produção de todos os endpoints REST HTTP expostos pelo **AMZX Node** (porta padrão `6869`) e pelo **AMZX Matcher DEX** (porta padrão `6886`). 

Todos os exemplos de requisição e resposta foram adaptados para refletir a marca **AMZX** (better2better.com.br), com valores em decimais nativos (Satoshis / $10^{-8}$ decimals) e formatos de payload de produção.

---

## 🔒 1. Segurança e Autenticação das APIs

As rotas administrativas e confidenciais do nó e do Matcher exigem autenticação do lado do servidor para evitar acessos não autorizados. Isso é feito por meio de um cabeçalho HTTP personalizado:

```http
X-API-Key: sua_api_key_secreta
```

### O Algoritmo de Validação Interna do Hash da API Key
Para que o nó não armazene a palavra-passe (`api-key`) em texto plano em seu arquivo de configuração `application.conf`, a segurança utiliza um hash duplo SHA-256. 
*   **Texto Plano**: `"sua_api_key_secreta"`
*   **Processamento de Hash**: `SHA256(SHA256("sua_api_key_secreta"))`
*   **Configuração no Nó**:
    ```hocon
    amzx.rest-api.api-key-hash = "4r2aT2x6eP7gW8sM9oD1uC3aB4v..."
    ```

Sempre que uma requisição atinge um endpoint administrativo, a classe `ApiKeyRoute.scala` intercepta a chamada, extrai o valor de `X-API-Key`, executa o duplo hash SHA-256 e compara o hash resultante com o valor configurado. Se houver discrepância, retorna **`HTTP 401 Unauthorized`**.

---

## ⛓️ 2. API REST do AMZX Node (Porta `6869`)

As rotas do Nó Scala gerenciam o ledger, blocos, transações, estado de contas e a camada de compatibilidade com a MetaMask.

### 2.1 Gerenciamento de Contas e Carteiras (`/addresses`)

#### `POST /addresses`
*   **Escopo**: Administrativo (Exige `X-API-Key`).
*   **Função**: Gera uma nova conta (chave privada, chave pública e endereço) sequencial de forma determinística na carteira do nó e a grava em `wallet.dat`.
*   **Requisição**: Sem corpo.
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "address": "3MvYourNewGeneratedAddress43Ksd9Fq73Lw"
    }
    ```

#### `GET /addresses`
*   **Escopo**: Administrativo (Exige `X-API-Key`).
*   **Função**: Retorna a lista completa de todos os endereços guardados no arquivo de carteira criptografado local do nó.
*   **Resposta (HTTP 200 OK)**:
    ```json
    [
      "3MvYourNewGeneratedAddress43Ksd9Fq73Lw",
      "3MxFirstGenesisOperationalAddressHere...",
      "3MyA947A11y7xskpP8..."
    ]
    ```

#### `GET /addresses/balance/details/{address}`
*   **Escopo**: Público (Não exige autenticação).
*   **Função**: Retorna o saldo detalhado da conta informada em Satoshis ($10^{-8}$ decimais), diferenciando saldo regular, gerador (PoS), disponível e efetivo (LPoS).
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "address": "3MvYourNewGeneratedAddress43Ksd9Fq73Lw",
      "regular": 50000000000,
      "generating": 35000000000,
      "available": 50000000000,
      "effective": 35000000000
    }
    ```
    *Nota: O saldo disponível acima é de exactamente 500 AMZX, com 350 AMZX efetivos de poder minerador.*

#### `GET /addresses/scriptInfo/{address}`
*   **Escopo**: Público.
*   **Função**: Retorna metadados se o endereço corresponder a um contrato inteligente dApp RIDE.
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "address": "3MvYourNewGeneratedAddress43Ksd9Fq73Lw",
      "script": "base64CompiledBinaryStringHere...",
      "scriptText": "DApp(None, List(), [Callable(invocante, ...)], None)",
      "version": 6,
      "complexity": 142,
      "verifierComplexity": 142,
      "callableComplexities": {
        "depositarMoeda": 85
      },
      "extraFee": 400000,
      "publicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV"
    }
    ```

---

### 2.2 Controle de Consultas de Dados de Contas (`/addresses/data`)

No AMZX, contas dApps podem armazenar dados estruturados diretamente no RocksDB usando chaves-valores através de transações de dados (ID 12).

#### `GET /addresses/data/{address}`
*   **Escopo**: Público.
*   **Função**: Retorna todas as chaves e valores armazenados no banco de dados para aquela conta dApp.
*   **Parâmetros de Query Opcionais**:
    *   `matches`: Expressão regular para filtrar chaves (ex: `?matches=status_.*`).
*   **Resposta (HTTP 200 OK)**:
    ```json
    [
      {
        "key": "dao_proposal_01_status",
        "type": "string",
        "value": "approved"
      },
      {
        "key": "dao_proposal_01_votes_yes",
        "type": "integer",
        "value": 4500
      },
      {
        "key": "dao_proposal_01_is_executed",
        "type": "boolean",
        "value": true
      }
    ]
    ```

#### `GET /addresses/data/{address}/{key}`
*   **Escopo**: Público.
*   **Função**: Retorna o valor de uma única chave específica armazenada na conta dApp.
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "key": "dao_proposal_01_status",
      "type": "string",
      "value": "approved"
    }
    ```

---

### 2.3 Exploração e Verificação de Blocos (`/blocks`)

#### `GET /blocks/height`
*   **Escopo**: Público.
*   **Função**: Retorna a altura máxima atual da cadeia de blocos sincronizada.
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "height": 148203
    }
    ```

#### `GET /blocks/at/{height}`
*   **Escopo**: Público.
*   **Função**: Retorna as informações físicas exaustivas do bloco minerador em determinada altura, contendo todas as transações embutidas.
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "version": 5,
      "timestamp": 1780578324000,
      "reference": "2WNhR15xizRv3uYwaXgtMzuvQ8ukKKimoDSp19ms3W3UU9XHgFo8q5jgcnWYKkJ7",
      "nxt-consensus": {
        "base-target": 45,
        "generation-signature": "7H16Kz8B6hWp... "
      },
      "transactions": [
        {
          "type": 4,
          "id": "8UfKz6bL9...",
          "sender": "3MxFirstGenesisOperationalAddressHere...",
          "recipient": "3MvYourNewGeneratedAddress43Ksd9Fq73Lw",
          "amount": 10000000000,
          "fee": 100000,
          "timestamp": 1780578324100,
          "proofs": ["2DbaY3zP..."]
        }
      ],
      "generator": "3MxFirstGenesisOperationalAddressHere...",
      "signature": "3M4qwDomRabJKLZxuXhwf...",
      "fee": 100000,
      "transactionCount": 1
    }
    ```

---

### 2.4 Processamento e Propagação de Transações (`/transactions`)

#### `POST /transactions/broadcast`
*   **Escopo**: Público.
*   **Função**: Injeta uma transação serializada e devidamente assinada no nó. A transação passa pelo pipeline de validação (Stateless e Stateful). Se aprovada, ela entra no UTX mempool e é propagada para toda a rede de computação P2P.
*   **Payload de Exemplo (Transferência de Token - ID 4)**:
    ```json
    {
      "type": 4,
      "version": 3,
      "senderPublicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV",
      "assetId": null,
      "recipient": "3MvYourNewGeneratedAddress43Ksd9Fq73Lw",
      "amount": 1000000000,
      "fee": 100000,
      "feeAssetId": null,
      "timestamp": 1780578324100,
      "attachment": "base58EncodedAttachmentString...",
      "proofs": [
        "5DfbAkZ9PwRtmzUq8vYksKpVwBnzWf18gX76sK23zY91u74B3gXhCpwNzo3pX1m2z"
      ]
    }
    ```
*   **Resposta (HTTP 200 OK - Retorna os detalhes com a ID calculada)**:
    ```json
    {
      "type": 4,
      "id": "8UfKz6bL9sKwB3zP...",
      "version": 3,
      "senderPublicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV",
      "assetId": null,
      "recipient": "3MvYourNewGeneratedAddress43Ksd9Fq73Lw",
      "amount": 1000000000,
      "fee": 100000,
      "feeAssetId": null,
      "timestamp": 1780578324100,
      "attachment": "base58EncodedAttachmentString...",
      "proofs": [
        "5DfbAkZ9PwRtmzUq8vYksKpVwBnzWf18gX76sK23zY91u74B3gXhCpwNzo3pX1m2z"
      ]
    }
    ```

#### `GET /transactions/status`
*   **Escopo**: Público.
*   **Função**: Consulta o status atual de uma lista de IDs de transação enviadas como query params.
*   **Parâmetros de Query**: `id=8UfKz6bL9sKwB3zP...`
*   **Resposta (HTTP 200 OK)**:
    ```json
    [
      {
        "id": "8UfKz6bL9sKwB3zP...",
        "status": "confirmed",
        "height": 148201,
        "confirmations": 2
      }
    ]
    ```

---

### 2.5 Camada de Compatibilidade MetaMask (`POST /eth` & `/eth/abi`)

Como detalhado, o nó emula o barramento JSON-RPC do Ethereum na rota pública `/eth`.

#### `POST /eth` (EVM-compatible JSON-RPC)
*   **Escopo**: Público (MetaMask/Ethers.js).
*   **Função**: Intercepta e processa chamadas de especificação Web3 JSON-RPC.
*   **Payload de Requisição Exemplo (`eth_getBalance`)**:
    ```json
    {
      "jsonrpc": "2.0",
      "method": "eth_getBalance",
      "params": ["0x90F8bf325439F454140c14c520D8B7cddB220D15", "latest"],
      "id": 1
    }
    ```
*   **Resposta da Requisição (HTTP 200 OK)**:
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": "0x2b5e3af16b1880000"
    }
    ```
    *Nota: O resultado hexadecimal `0x2b5e3af16b1880000` em Wei traduz-se em exatamente 50.000.000.000 Satoshis no banco RocksDB do nó (50 AMZX), escalonado pelo fator multiplicador $10^{10}$.*

#### `GET /eth/abi/{address}`
*   **Escopo**: Público.
*   **Função**: Inspeciona a Árvore de Sintaxe Abstrata (AST) do script RIDE compilado ativado na conta e gera automaticamente o array ABI JSON equivalente para frontends Solidity.
*   **Resposta (HTTP 200 OK)**:
    ```json
    [
      {
        "inputs": [
          {
            "internalType": "string",
            "name": "nomeToken",
            "type": "string"
          }
        ],
        "name": "depositarMoeda",
        "outputs": [],
        "stateMutability": "payable",
        "type": "function"
      }
    ]
    ```

---

## 📈 3. API REST do AMZX Matcher DEX (Porta `6886`)

O Matcher DEX expõe rotas HTTP e endpoints WebSockets para gerenciar as ordens financeiras de mercado e correspondência rápida.

### 3.1 Chave e Configurações Globais

#### `GET /matcher`
*   **Escopo**: Público.
*   **Função**: Retorna a chave pública da conta de correspondência central do Matcher DEX. Todas as ordens geradas pelos clientes devem usar essa chave pública no campo `matcherPublicKey` para que o cruzamento de ordem seja aceito.
*   **Resposta (HTTP 200 OK)**:
    ```json
    "8UfKz6bL9sKwB3zP2WNhR15xizRv3uYwaXgtMzuvQ8uk"
    ```

---

### 3.2 Gerenciamento do Livro de Ofertas (`/matcher/orderbook`)

Os pares de negociação do Matcher DEX são compostos por dois ativos identificados pelas IDs de tokens: `AmountAsset` (ativo que está sendo negociado) e `PriceAsset` (ativo cotado usado como base de pagamento). Se for usar a moeda nativa AMZX, utiliza-se a palavra-passe `"WAVES"`.

#### `GET /matcher/orderbook/{amountAsset}/{priceAsset}`
*   **Escopo**: Público.
*   **Função**: Retorna a profundidade atual das ofertas de compra (bids) e venda (asks) em memória TreeMap do par de negociação, agrupados e ordenados por preço.
*   **Exemplo de Rota**: `GET /matcher/orderbook/4WNhR15x.../WAVES`
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "timestamp": 1780578324500,
      "pair": {
        "amountAsset": "4WNhR15x...",
        "priceAsset": "WAVES"
      },
      "bids": [
        { "price": 105000000, "amount": 5000000000 },
        { "price": 104500000, "amount": 3000000000 }
      ],
      "asks": [
        { "price": 105500000, "amount": 1200000000 },
        { "price": 106000000, "amount": 4500000000 }
      ]
    }
    ```

#### `POST /matcher/orderbook`
*   **Escopo**: Público (Assinado pelo remetente).
*   **Função**: Envia uma nova ordem de compra ou venda assinada pelo cliente para entrar na fila do TreeMap do Matcher.
*   **Payload de Requisição Exemplo**:
    ```json
    {
      "version": 4,
      "id": "OrdemCalculadaID...",
      "senderPublicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV",
      "matcherPublicKey": "8UfKz6bL9sKwB3zP2WNhR15xizRv3uYwaXgtMzuvQ8uk",
      "assetPair": {
        "amountAsset": "4WNhR15x...",
        "priceAsset": "WAVES"
      },
      "orderType": "buy",
      "price": 105000000,
      "amount": 5000000000,
      "timestamp": 1780578324600,
      "expiration": 1782306324600,
      "matcherFee": 300000,
      "matcherFeeAssetId": null,
      "proofs": [
        "29DbaY3zP789182k..."
      ]
    }
    ```
*   **Resposta (HTTP 201 Created)**:
    ```json
    {
      "status": "OrderAccepted",
      "message": {
        "id": "OrdemCalculadaID...",
        "timestamp": 1780578324602
      }
    }
    ```

#### `POST /matcher/orderbook/{amountAsset}/{priceAsset}/cancel`
*   **Escopo**: Público (Exige assinatura criptográfica de cancelamento no payload).
*   **Função**: Remove uma ordem ativa do TreeMap de ordens em memória imediatamente.
*   **Payload de Exemplo**:
    ```json
    {
      "senderPublicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV",
      "orderId": "OrdemCalculadaID...",
      "signature": "3M4qwDomRabJKLZxuXhwfqLApQ..."
    }
    ```
*   **Resposta (HTTP 200 OK)**:
    ```json
    {
      "status": "OrderCanceled",
      "orderId": "OrdemCalculadaID..."
    }
    ```

---

## 🚫 4. Tabela de Tratamento de Erros e Códigos HTTP

O ecossistema utiliza códigos JSON padronizados com herança de erro unificada para facilitar o diagnóstico de integrações de frontend ou automações.

| Status HTTP | Código de Erro Interno | Descrição Técnica e Causa Raiz |
| :---: | :---: | :--- |
| **`400 Bad Request`** | `10` | **InvalidSignature**: Assinatura criptográfica de provas (`proofs`) inválida ou corrompida. |
| **`400 Bad Request`** | `11` | **TransactionNotAllowedByScript**: Rejeição de transação devido à lógica estrita do `@Verifier` do dApp. |
| **`400 Bad Request`** | `20` | **NegativeAmount**: O montante de transferência ou emissão está menor ou igual a zero. |
| **`400 Bad Request`** | `35` | **ComplexityLimitExceeded**: A complexidade do script RIDE de dApp excede os limites de segurança de rede (4000). |
| **`401 Unauthorized`** | `1` | **ApiKeyMissing**: Cabeçalho de autorização `X-API-Key` ausente em rota protegida. |
| **`401 Unauthorized`** | `2` | **ApiKeyInvalid**: O hash duplo derivado da chave enviada difere do configurado no nó. |
| **`404 Not Found`** | `30` | **AssetDoesNotExist**: Consulta de detalhes ou balanço de uma ID de token que não foi emitida. |
| **`422 Unprocessable`** | `112` | **InsufficientFunds**: O remetente não possui saldo de moeda nativa suficiente para cobrir o gas de rede e o envio. |
| **`503 Service Unavailable`**| `400` | **MatcherIsResolvingRollback**: O Matcher DEX está recalculando o estado do livro de ofertas após um fork de rede e está temporariamente suspenso para escritas. |

### Exemplo de Payload de Resposta de Erro da API:
```json
{
  "error": 112,
  "message": "The sender's address 3MvYourAddress... has insufficient balance (required: 1000100000 Satoshis, available: 500000000 Satoshis) to process transaction type 4.",
  "details": {
    "sender": "3MvYourAddress...",
    "required": 1000100000,
    "available": 500000000
  }
}
```
