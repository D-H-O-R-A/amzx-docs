# 🪪 Manual de Geração de Endereços e Customização da Camada de Contas

Este documento apresenta a especificação binária, criptográfica e algorítmica de geração de contas e endereços clássicos no ecossistema **AMZX** (conforme codificado no arquivo core `Recipient.scala`), além de fornecer o mapa completo de constantes, classes e caminhos de código necessários para desenvolvedores que queiram customizar o tamanho, prefixo ou regras de validação de endereços da blockchain.

---

## 📐 1. Estrutura Binária de um Endereço AMZX

O endereço clássico na blockchain AMZX é um vetor ordenado de **26 bytes** (representado de forma legível por humanos como uma string em formato **Base58** de 35 caracteres). Ele não é uma chave pública pura, mas sim um envelopamento criptográfico de um hash derivado.

Abaixo está o layout binário exato byte-a-byte de um endereço válido na rede:

```
+=============================================================================+
|                      AMZX ADDRESS BINARY LAYOUT (26 BYTES)                  |
+=============================================================================+
| Offset (Bytes) | Tamanho | Tipo de Dado | Função Criptográfica / Descrição  |
+----------------+---------+--------------+-----------------------------------+
| Byte 0         | 1 byte  | Byte         | Versão do Endereço (Sempre 0x01)  |
+----------------+---------+--------------+-----------------------------------+
| Byte 1         | 1 byte  | Char / Byte  | Chain ID / Magic Byte (Ex: 'R')   |
+----------------+---------+--------------+-----------------------------------+
| Bytes 2 a 21   | 20 bytes| Array[Byte]  | Hash de Chave Pública (PubKeyHash)|
+----------------+---------+--------------+-----------------------------------+
| Bytes 22 a 25  | 4 bytes | Array[Byte]  | Checksum de integridade do payload|
+=============================================================================+
```

---

## 🔒 2. O Algoritmo Criptográfico de Geração de Contas

A derivação de um endereço a partir de uma semente de texto (Seed Phrase) de carteira ocorre de acordo com as seguintes fases estritas:

### Fase A: Derivação de Chaves (Curve25519 / Ed25519)
1.  **Geração do Hash da Semente**: O UTF-8 da string da semente é convertido para bytes e hashizado usando Blake2b256 para gerar a chave de entropia de 32 bytes:
    $$\text{SeedHash} = \text{Blake2b256}(\text{SeedText})$$
2.  **Chave Privada**: O `SeedHash` é processado sequencialmente através de um gerador estático indexado por um nonce incremental (usualmente `0` para a primeira conta) para derivar a chave privada Curve25519 de 32 bytes.
3.  **Chave Pública**: A chave pública Curve25519 de 32 bytes é derivada multiplicando o ponto base da curva pela chave privada:
    $$\text{PubKey} = \text{Curve25519.calculatePublicKey}(\text{PrivKey})$$

### Fase B: Construção do Payload e Hash de Conta
1.  **Cálculo do Hash de Chave Pública (PubKeyHash)**: A chave pública gerada de 32 bytes é processada sequencialmente pelo algoritmo duplo de hash seguro do AMZX (Blake2b-256 seguido de Keccak-256) e apenas os primeiros **20 bytes** do digest resultante são selecionados para compor o corpo do endereço:
    $$\text{FullHash} = \text{Keccak256}(\text{Blake2b256}(\text{PubKey}))$$
    $$\text{PubKeyHash} = \text{Primeiros20Bytes}(\text{FullHash})$$

### Fase C: Selagem com Checksum
1.  **Montagem do Bloco Sem Checksum (withoutChecksum)**: Cria-se um vetor contendo a versão, o Chain ID atual da rede e o PubKeyHash:
    $$\text{payload} = 0x01 \mathbin{\Vert} \text{ChainID} \mathbin{\Vert} \text{PubKeyHash}$$
2.  **Cálculo do Checksum**: O vetor de 22 bytes (`payload`) é processado novamente pelos algoritmos de hash criptográfico duplo. Seleciona-se os primeiros **4 bytes** do resultado:
    $$\text{FullChecksum} = \text{Keccak256}(\text{Blake2b256}(\text{payload}))$$
    $$\text{Checksum} = \text{Primeiros4Bytes}(\text{FullChecksum})$$
3.  **Vetor Final**: Junta-se o payload ao checksum para compor os 26 bytes estáveis e exporta-se em formato Base58.

---

## 🛠️ 3. Guia de Customização: Como Alterar o Endereço (Tamanho, Regras e Prefixos)

Se a sua equipe precisar customizar a camada de contas (por exemplo, aumentar o tamanho do hash de conta de 20 para 32 bytes por motivos de segurança, alterar a versão do endereço ou redefinir o checksum), você precisará atuar nos locais descritos abaixo.

### A. Constantes Fundamentais e Arquivos a Modificar:

As regras que ditam como a rede gera e valida endereços estão centralizadas no arquivo:
👉 **[Recipient.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/account/Recipient.scala)**

Dentro do objeto companheiro `object Address` (linhas 61 a 171), altere as seguintes constantes de acordo com o seu requisito de design de rede:

*   **`AddressVersion`** (Linha 63): `val AddressVersion: Byte = 1`. Se você criar uma nova versão de endereçamento incompatível com redes antigas, mude para `2` ou superior.
*   **`ChecksumLength`** (Linha 64): `val ChecksumLength: Int = 4`. Determina o tamanho do vetor de checksum. Aumentar para 8 bytes reduz a possibilidade matemática de erro acidental, mas aumenta o tamanho do endereço.
*   **`HashLength`** (Linha 65): `val HashLength: Int = 20`. É a constante chave que dita o tamanho do PubKeyHash. Caso queira alterá-la para 32 bytes (para conter hashes puros Keccak256 sem truncamento), mude esta constante para `32`.
*   **`AddressLength`** (Linha 66): `val AddressLength: Int = 1 + 1 + HashLength + ChecksumLength`. Esta variável calcula automaticamente o tamanho físico total em bytes do endereço com base nas anteriores. No modelo padrão, resulta em $1 + 1 + 20 + 4 = 26$ bytes. Se você mudar o `HashLength` para 32, ela mudará automaticamente para 38 bytes.

```scala
// Recipient.scala (Linhas 61-67):
object Address {
  val Prefix: String           = "address:"
  val AddressVersion: Byte     = 1   // <-- Alterar se criar nova especificação
  val ChecksumLength: Int      = 4   // <-- Alterar para mudar tamanho do Checksum
  val HashLength: Int          = 20  // <-- Alterar para mudar tamanho do Hash da PubKey
  val AddressLength: Int       = 1 + 1 + HashLength + ChecksumLength
  val AddressStringLength: Int = base58Length(AddressLength)
```

---

### B. Passo a Passo para Alteração de Regras de Endereço:

```
            [Mudar Constantes de Tamanho em Recipient.scala]
              - HashLength (ex: 32)
              - AddressVersion
                               |
                               v
             [Ajustar Métodos de Alocação de Bytes]
              - apply
              - fromPublicKey
              - createUnsafe
                               |
                               v
            [Atualizar Schemas Protobuf e Serialização]
              - Protobuf Schemas (.proto)
              - Serializers de Transações Scala
                               |
                               v
             [Compilar e Validar com Suíte de Testes]
```

1.  **Atualização de Constantes**: Abra o arquivo `Recipient.scala` e altere o `HashLength` para o tamanho desejado (ex: `32` bytes).
2.  **Adaptação dos Buffers de Alocação**:
    *   No método `fromPublicKey` (linha 91), o nó utiliza a classe `ByteBuffer.allocate` para montar a estrutura em bytes. Você deve garantir que a alocação de memória reflita exatamente os novos tamanhos parametrizados:
        ```scala
        // Certifique-se de que os buffers acompanham as novas constantes:
        val withoutChecksum = ByteBuffer
          .allocate(1 + 1 + HashLength)
          .put(AddressVersion)
          .put(chainId)
          .put(crypto.secureHash(publicKey.arr), 0, HashLength)
          .array()
        ```
3.  **Atualização de Mapeadores de Leitura Rápida**:
    *   No método privado `createUnsafe` (linha 169), mude a lógica de extração de fatias (*slicing*) de arrays de bytes caso mude as margens do cabeçalho de bytes do endereço:
        ```scala
        private def createUnsafe(addressBytes: Array[Byte]): Address =
          new Address(addressBytes(1), addressBytes.drop(2).dropRight(ChecksumLength), addressBytes.takeRight(ChecksumLength))
        ```

---

### ⚠️ Implicações Críticas ao Alterar o Address Length (Alerta de Quebra)

Aumentar ou diminuir o `AddressLength` na camada core da blockchain trará impactos severos e ramificações que exigirão alterações adicionais nos seguintes subsistemas:

1.  **Armazenamento em Banco de Dados (RocksDB)**:
    *   O nó grava dados de saldos de contas usando o endereço em bytes como parte do prefixo da chave binária no RocksDB. Alterar o tamanho do endereço alterará o alinhamento das chaves de indexação de banco, exigindo a reconstrução completa do estado da blockchain a partir do bloco zero (nova Gênesis).
2.  **Camada de Serialização de Transações (gRPC e Protobuf)**:
    *   As definições de interfaces do gRPC descritas em arquivos `.proto` definem os endereços como arrays de bytes ou strings. Se houver validações estáticas de tamanho nas bibliotecas cliente (como no repositório `amzx-pb`), elas rejeitarão os novos tamanhos até que os arquivos Protobuf sejam recompilados.
3.  **Assinatura de Transações no Cliente (Wallets & Libraries)**:
    *   Todas as chaves, SDKs JavaScript, Python, bibliotecas de hardware (Ledger) e a própria MetaMask validam o tamanho fixo de 26 bytes do endereço clássico para assinar dados. Se o tamanho for alterado no nó, todas as extensões e bibliotecas cliente precisarão ser atualizadas de forma síncrona, sob pena de tornarem-se permanentemente incompatíveis.
