# 🦊 Compatibilidade Ethereum e MetaMask a Fundo (Ethereum RPC & Bridge)

Este documento destrincha a engenharia de compatibilidade que permite ao **AMZX Node** emular uma rede EVM-like (Ethereum Virtual Machine) compatível com a **MetaMask**, permitindo que chamadas JSON-RPC (`/eth`), chaves baseadas em **secp256k1** e chamadas de dApps funcionem de forma direta sobre o ledger AMZX.

---

## 🏗️ 1. Como Foi Possível? A Fusão de Curvas

Originalmente, a Waves operava exclusivamente com a curva elíptica **Curve25519** (algoritmo Ed25519 de assinaturas EdDSA), enquanto a rede Ethereum e a maioria das redes EVM utilizam a curva **secp256k1** (algoritmo ECDSA).

A compatibilidade total com a MetaMask foi alcançada na versão **1.4.0** do nó através de três mudanças arquiteturais fundamentais:
1.  **Suporte de Baixo Nível para Dupla Criptografia**: Introdução de assinaturas secp256k1 nativas na camada de validação criptográfica do nó (`crypto`).
2.  **Transação do Tipo 18 (`EthereumTransaction`)**: Criação de um tipo de transação on-chain dedicado que envelopa os payloads brutos de transações padronizadas do Ethereum (EIP-1559, EIP-2930, e Legacy).
3.  **Camada de Emulação RPC (`/eth`)**: Um servidor web que intercepta as chamadas HTTP `POST` JSON-RPC emitidas por carteiras web3, traduz os métodos EVM para comandos internos e retorna respostas formatadas com precisão de bytes EVM.

---

## 🔄 2. O Algoritmo Isomórfico de Conversão de Endereços

Uma das maiores dificuldades de engenharia foi mapear um endereço Ethereum de **20 bytes** para o endereço clássico AMZX de **26 bytes** de forma determinística e bidirecional (sem perda de dados).

### A Sacada Genial: O Hash de Chave Pública
No AMZX, o endereço possui o seguinte formato binário de 26 bytes:
$$\text{Bytes do Endereço} = \text{Versão (1 byte)} \mathbin{\Vert} \text{Chain ID (1 byte)} \mathbin{\Vert} \text{Public Key Hash (20 bytes)} \mathbin{\Vert} \text{Checksum (4 bytes)}$$

Como o tamanho do hash de chave pública clássico no AMZX é exatamente **20 bytes** (gerado por `Keccak256(Blake2b256(PubKey))`), os engenheiros mapearam o endereço Ethereum de 20 bytes **diretamente como o Public Key Hash** do endereço AMZX!

### Fluxo de Conversão: Ethereum para AMZX (20 bytes para 26 bytes)
Dado um endereço Ethereum: `0x90F8bf325439F454140c14c520D8B7cddB220D15` (20 bytes brutos):

1.  **Montagem do Prefix**: O prefixo é composto pelo byte de Versão do endereço (`0x01`) e pelo byte do Chain ID da rede (ex: `'R'` = `0x52` ou `82` em decimal).
2.  **Injeção do Hash**: O endereço Ethereum bruto é inserido como o corpo do hash (20 bytes).
3.  **Cálculo do Checksum**: O checksum de 4 bytes é calculado aplicando hashes criptográficos duplicados (`Keccak256` após `Blake2b256`) sobre o payload contendo o prefixo e o hash:
    $$\text{Checksum} = \text{Primeiros4Bytes}(\text{Keccak256}(\text{Blake2b256}(\text{Prefixo} \mathbin{\Vert} \text{EthereumAddress})))$$
4.  **Codificação Base58**: Junta-se o Prefixo, o EthereumAddress e o Checksum, gerando um vetor de 26 bytes que é codificado em Base58 (ex: `3M4qwDomRabJKLZxuXhwfqLApQ...`).

```
+------------------+------------------+-----------------------------+--------------------+
| Versão (1 byte)  | Chain ID (1 byte)| Ethereum Address (20 bytes) | Checksum (4 bytes) |
|      0x01        |  e.g., 0x52 ('R')|  0x90F8bf325439F454140...   |     0x83A4B92C     |
+------------------+------------------+-----------------------------+--------------------+
\___________________________________________ 26 Bytes __________________________________/
                                                    |
                                                    v Base58 Encode
                                          3M4qwDomRabJKLZxuXhwf...
```

Este mapeamento garante que qualquer endereço Ethereum possua um endereço AMZX equivalente único na mesma rede, sem colisões.

---

## 🧮 3. Conversão de Decimais e Escala de Saldos (Satoshis para Wei)

As transações de tokens e moedas nativas na blockchain AMZX operam com **8 casas decimais** (Satoshis):
$$1 \text{ AMZX} = 10^8 \text{ Satoshis}$$

Por outro lado, a rede Ethereum e a MetaMask exigem **18 casas decimais** para a moeda de gas nativa (Wei):
$$1 \text{ ETH} = 10^{18} \text{ Wei}$$

### O Algoritmo de Escala Multiplicadora ($10^{10}$)
Para evitar que a MetaMask mostre saldos errados ou ordens fracionadas grotescas, a API RPC do nó implementa uma escala estrita em tempo de execução:
1.  **Leitura do Banco (RocksDB)**: O nó lê o saldo real em satoshis (ex: $5.000.000.000$ satoshis, equivalentes a $50$ AMZX).
2.  **Conversão de Escala**: Multiplica-se o saldo lido por uma constante de escala de $10^{10}$:
    $$\text{Saldo em Wei} = \text{Saldo em Satoshis} \times 10^{10}$$
    $$5.000.000.000 \times 10^{10} = 50.000.000.000.000.000.000 \text{ Wei} \ (50 \text{ ETH na MetaMask})$$
3.  **Envio via JSON-RPC**: O valor em Wei é formatado como uma string hexadecimal e retornado para a MetaMask.
4.  **Escala Reversa em Transações**: Quando o usuário envia uma transação via MetaMask (especificando, por exemplo, $10$ ETH), a camada RPC intercepta o valor em Wei e faz a divisão exata por $10^{10}$ para gravar os satoshis corretos on-chain ($1.000.000.000$ satoshis).

---

## 🔑 4. Recuperação de Chaves Criptográficas (secp256k1 e ecrecover)

Sempre que uma transação Ethereum é enviada para o nó através do método `eth_sendRawTransaction`, ela vem assinada com o algoritmo ECDSA (`secp256k1`). O nó utiliza a classe `EthereumTransaction.scala` para validar criptograficamente os bytes.

```
       [Raw Hex da Transação do MetaMask]
                       |
                       v
     [Separar RLP: Payload + Assinatura v, r, s]
                       |
                       v
         [Keccak256 Hash do Payload]
                       |
                       +=======> [Algoritmo de Recuperação ECDSA (ecrecover)]
                       |                               |
       [Cálculo de Curva Elíptica]                      v
                       |                   [Chave Pública secp256k1]
                       v                               |
            [Assinatura é Válida?]                     v
              (Sim = Processa Tx)             [Endereço AMZX de 26 Bytes]
```

### Algoritmo de Verificação e Validação On-chain:
1.  **Desempacotamento RLP (Recursive Length Prefix)**: O nó decodifica o payload bruto em hex enviada pela MetaMask para separar a transação em si das variáveis de assinatura: `v` (recovery ID), `r` e `s`.
2.  **Cálculo do Hash de Mensagem**: Calcula-se o hash `Keccak256` sobre o payload da transação (incluindo nonce, gasPrice, gasLimit, to, value e input).
3.  **Recuperação de Chave Pública (ecrecover)**: Utilizando a biblioteca nativa ou bindings Java de criptografia, a JVM executa a multiplicação de ponto na curva secp256k1 para derivar a chave pública de **64 bytes** (excluindo o byte de sinal de compressão) a partir do hash e das variáveis `v, r, s`.
4.  **Derivação e Comparação de Endereço**: A chave pública recuperada é convertida para o endereço correspondente. Se o endereço derivado bater com o remetente da transação, a assinatura é classificada como válida.
5.  **Ajuste do Chain ID**: O valor de `v` codifica o Chain ID para evitar ataques de repetição em múltiplas redes (conforme especificado na EIP-155). O nó valida se o Chain ID embutido em `v` bate exatamente com o caractere ASCII correspondente à rede (ex: Testnet `'T'` = `84`, Stagenet `'S'` = `83`).

---

## 🤖 5. Chamada de dApps e Geração Dinâmica de ABI

Uma transação enviada da MetaMask para interagir com smart contracts RIDE passa por uma tradução de dados complexa na camada de compatibilidade.

### O Fluxo de uma Chamada dApp:
Quando a MetaMask envia um método de contrato (ex: `transferirTokens(address, uint256)`):

1.  **Envio do Payload Data (`input`)**: A MetaMask codifica a chamada usando o padrão de ABI Solidity do Ethereum:
    *   **Método Selector (4 bytes)**: Os primeiros 4 bytes representam o hash Keccak256 da assinatura da string do método (ex: `0xa9059cbb`).
    *   **Parâmetros (N blocos de 32 bytes)**: Os parâmetros seguintes são alinhados em blocos binários de 32 bytes.
2.  **Tradução de ABI no AMZX Node**:
    *   O nó intercepta a transação e lê o script do contrato RIDE implantado no endereço de destino.
    *   Ele analisa as funções decoradas com `@Callable(i)` do contrato RIDE.
    *   Ele mapeia o Método Selector (4 bytes) para descobrir qual função RIDE corresponde àquela assinatura do Ethereum.
3.  **Conversão de Tipos de Parâmetros**:
    *   Os argumentos em formato binário de 32 bytes do Ethereum são traduzidos para os tipos nativos do RIDE:
        *   `address` (Solidity) $\implies$ `Address` (RIDE / 26 bytes com preenchimento).
        *   `uint256` (Solidity) $\implies$ `Int` (RIDE / inteiro assinado de 64 bits, validando se o valor cabe em 8 bytes e dividindo por $10^{10}$ para ajustar decimais).
        *   `string` (Solidity) $\implies$ `String` (RIDE).
4.  **Execução e Gravação de Estado**: O compilador e interpretador RIDE rodam a função e geram uma lista de mutações (write-sets de dados ou transferências) que são gravadas no RocksDB sob a transação ID 18.

### O Gerador de ABI Dinâmico (`GET /eth/abi/{address}`)
Para que o desenvolvedor web3 não precise criar uma ABI manualmente para integrar o frontend de dApps (usando bibliotecas como `ethers.js` ou `web3.js`), o nó expõe uma rota REST especial:

`GET http://localhost:6869/eth/abi/3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF`

#### Como funciona internamente?
1.  O nó acessa a conta no endereço informado e extrai o script RIDE ativo no RocksDB.
2.  Ele processa a Árvore de Sintaxe Abstrata (AST) do script compilado e filtra todas as declarações de funções callable (`@Callable`).
3.  Para cada callable, ele traduz a assinatura para o esquema JSON ABI do Ethereum:
    *   Mapeia o nome da função RIDE para o atributo `name` do JSON.
    *   Mapeia os parâmetros aceitos pela função para a lista de `inputs` contendo os tipos EVM correspondentes.
4.  Retorna o JSON completo. O frontend do dApp web3 pode ler essa rota REST, alimentar a MetaMask e chamar as funções do contrato RIDE como se fossem funções Solidity convencionais!
