# ⛏️ Anatomia Molecular de Nós Validadores, Criptografia, Consenso (LPoS, FairPoS & Waves-NG) e Leasing

Este documento apresenta uma dissecação anatômica exaustiva, de nível cirúrgico e de baixíssimo nível, de como funcionam os nós, os validadores, a criptografia e o consenso na blockchain. Analisamos de ponta a ponta cada primitivo criptográfico, os bytes que trafegam na camada de rede P2P durante o Handshake, as equações matemáticas do FairPoS e o manual definitivo de Leasing não-custodial e recompensas.

---

## 🧬 1. Anatomia Criptográfica Molecular (As Primitivas de Segurança)

A segurança, a autoria de transações e a integridade de estados são garantidas por uma arquitetura criptográfica dual-engine que integra criptografia clássica de curvas elípticas de alta performance com curvas de compatibilidade com o ecossistema EVM/MetaMask.

```
                  +-------------------------------------------------------------+
                  |                 ARQUITETURA CRIPTOGRÁFICA                   |
                  +-------------------------------------------------------------+
                     |                         |                        |
                     v                         v                        v
           +-------------------+     +-------------------+    +-------------------+
           |  Assinaturas PoS  |     |  Compatibilidade  |    |  Funções de Hash  |
           |    & Contas       |     |     MetaMask      |    |  & Integridade    |
           | (Curve25519/Ed)   |     | (ECDSA secp256k1) |    | (Blake2b / Keccak)|
           +-------------------+     +-------------------+    +-------------------+
```

### A. Criptografia Assimétrica Padrão (Curve25519 e Ed25519)
*   **Chaves de Contas Clássicas**: A rede adota a curva elíptica **Curve25519** em conjunto com o esquema de assinaturas **Ed25519** para contas nativas e transações normais.
*   **Chave Privada**: Uma semente aleatória de **32 bytes** (256 bits) de altíssima entropia.
*   **Chave Pública**: Possui exatamente **32 bytes** (256 bits), derivada da multiplicação escalar de pontos na Curve25519 a partir da chave privada.
*   **Assinatura de Transação**: Tem comprimento fixo de **64 bytes** (512 bits), contendo os pontos `R` (32 bytes) e `S` (32 bytes). Ela assina o array de bytes serializados da transação (`bodyBytes`). A validação é extremamente rápida, permitindo verificar milhares de assinaturas por segundo por thread de CPU.

### B. Criptografia de Compatibilidade Ethereum (ECDSA secp256k1)
*   Para estender suporte nativo a carteiras Web3 líderes do mercado como a **MetaMask**, a rede integra a curva elíptica **secp256k1**, o mesmo padrão utilizado no Bitcoin e Ethereum.
*   **Chave Privada**: **32 bytes** (256 bits).
*   **Chave Pública**: **64 bytes** (descomprimida, excluindo o byte de prefixo `0x04`) ou **33 bytes** (comprimida).
*   **Assinatura de Transação**: Tem comprimento de **65 bytes** (520 bits), estruturada nos componentes `R` (32 bytes), `S` (32 bytes) e `V` (1 byte). O byte `V` representa o ID de recuperação da assinatura, permitindo que a rede recupere a chave pública e derive o endereço do remetente diretamente a partir da assinatura e do hash da transação.

### C. Funções de Hash e Integridade de Dados
*   **Blake2b-256**: O algoritmo de hash criptográfico padrão da rede para geração de IDs de transações, hashes de blocos e árvore de Merkle. Ele processa dados em blocos de 64 bits de forma extremamente otimizada em CPUs de 64 bits, sendo muito mais rápido que o SHA-256 tradicional. Retorna um hash de exatamente **32 bytes** (256 bits).
*   **Keccak-256**: Utilizado em conjunto com o Blake2b-256 no processo de derivação de endereços de contas e para garantir a compatibilidade estrita com os hashes de transações e contratos do ecossistema Ethereum. Retorna um hash de exatamente **32 bytes** (256 bits).

### D. Processo de Geração do Endereço Nativo (Passo a Passo Matemático)
Para derivar um endereço público nativo legível (Base58) a partir de uma chave pública Curve25519 de 32 bytes, a blockchain executa deterministicamente a seguinte rotina:

```
[Chave Pública (32 bytes)]
          |
          v
    Blake2b-256
          |
          v
     Keccak-256
          |
          v
[Hash de 32 bytes] ---> Extrai os primeiros 20 bytes
                                 |
                                 v
                     [Hash Extraído (20 bytes)]
                                 |
              Concatena:         v
[Versão (1 byte: 0x01)] + [Chain ID (1 byte)] + [Hash Extraído (20 bytes)]
                                 |
                                 v
                     [Payload Base (22 bytes)]
                                 |
         +-----------------------+-----------------------+
         |                                               |
         |  Calcula Checksum:                            |
         v  Blake2b-256 -> Keccak-256                    |
[Checksum (primeiros 4 bytes)]                           |
         |                                               |
         +-----------------------+-----------------------+
                                 |
                                 v
                     [Bytes Totais (26 bytes)]
                                 |
                                 v
                              Base58
                                 |
                                 v
                 [Endereço Final (ex: 3P... / 3M...)]
```

1.  **Primeiro Hash**: Aplica-se o hash **Blake2b-256** aos 32 bytes da chave pública, resultando em um hash de 32 bytes.
2.  **Segundo Hash**: Aplica-se o hash **Keccak-256** ao resultado do primeiro hash, gerando um novo hash de 32 bytes.
3.  **Truncagem**: Extraem-se os **primeiros 20 bytes** do hash Keccak resultante.
4.  **Concatenação de Cabeçalho**: Prependem-se 2 bytes ao hash truncado:
    *   **Byte 0**: O byte de versão do endereço (`0x01` / Versão 1).
    *   **Byte 1**: O Chain ID da rede em formato numérico (por exemplo, `87` para a Mainnet `'W'`, ou `65` para rede Custom `'A'`).
    *   O resultado temporário possui exatamente **22 bytes**.
5.  **Cálculo do Checksum**:
    *   Aplica-se o hash Blake2b-256 aos 22 bytes do passo anterior.
    *   Aplica-se o hash Keccak-256 ao resultado do Blake2b.
    *   Extraem-se os **primeiros 4 bytes** deste hash Keccak final. Estes 4 bytes são o **Checksum**.
6.  **Montagem Final dos Bytes**: Concatena-se o Checksum (4 bytes) ao final do Payload Base (22 bytes), resultando em uma estrutura binária de exatamente **26 bytes**.
7.  **Codificação Base58**: Codificam-se os 26 bytes resultantes utilizando o alfabeto Base58 (evitando caracteres ambíguos como `0`, `O`, `I`, `l`). O resultado é uma string de aproximadamente **35 caracteres**, tipicamente iniciando com `3`.

### E. Mapeamento de Endereços Ethereum (MetaMask) para Endereços Nativos
Quando a MetaMask envia uma transação baseada em chaves secp256k1, o endereço Ethereum padrão é derivado pegando os últimos 20 bytes do hash Keccak-256 de sua chave pública de 64 bytes.
Para mapear esse endereço de 20 bytes da Ethereum de volta ao formato da rede nativa e garantir a unicidade de saldos:
1.  O nó pega os **20 bytes** do endereço Ethereum.
2.  Prepende o byte de versão `0x01` e o byte de Chain ID ativo da rede (ex: `65` / `'A'`).
3.  Calcula o Checksum de 4 bytes através do duplo hash (Blake2b-256 seguido de Keccak-256) dos 22 bytes precedentes.
4.  Concatena os bytes resultando em **26 bytes**.
5.  Codifica em **Base58** gerando um endereço de formato correspondente perfeito. Isso permite que uma conta MetaMask acesse o mesmo saldo e contratos do ecossistema de forma transparente!

---

## 📡 2. Comunicação na Camada de Rede P2P e Handshake de Protocolo

A comunicação entre os nós validadores ocorre de forma peer-to-peer (P2P) sobre conexões TCP persistentes. O fluxo de conexão e sincronização de blocos obedece a regras rígidas de rede.

### A. Layout Binário do Pacote de Handshake (Aperto de Mão Criptográfico)
No momento em que duas instâncias de nós tentam estabelecer conexão na porta pública (ex: `6868`), elas trocam obrigatoriamente um pacote binário chamado **Handshake**. O layout exato dos bytes que trafegam pelo socket TCP, gerado pela biblioteca de rede **Netty** do nó, é estruturado da seguinte forma:

| Campo | Tamanho em Bytes | Tipo de Dado | Descrição e Validação |
| :--- | :---: | :--- | :--- |
| **AppNameLength** | 1 byte | Byte (Signed) | O tamanho do nome da aplicação (máximo de 127 bytes). |
| **ApplicationName** | Variável | String UTF-8 | Nome da aplicação (ex: `wavesW` para Waves Mainnet ou `amzxA` para AMZX Custom). |
| **VersionMajor** | 4 bytes | Int (Big-Endian) | Número principal da versão do nó (ex: `1`). |
| **VersionMinor** | 4 bytes | Int (Big-Endian) | Número secundário da versão do nó (ex: `6`). |
| **VersionPatch** | 4 bytes | Int (Big-Endian) | Número do patch de correção (ex: `3`). |
| **NodeNameLength** | 1 byte | Byte (Signed) | Tamanho do nome descritivo do nó (máximo de 127 bytes). |
| **NodeName** | Variável | String UTF-8 | Nome amigável configurado pelo administrador do nó em `network.node-name`. |
| **NodeNonce** | 8 bytes | Long (Big-Endian) | Número aleatório que evita que o nó estabeleça conexões consigo mesmo. |
| **DeclaredAddressLength** | 4 bytes | Int (Big-Endian) | Tamanho do endereço IP declarado (pode ser `0`, `8` ou `20`). |
| **DeclaredAddress** | Variável | Bytes + Int | Se length = `8`: IPv4 (4 bytes) + Porta (4 bytes Int). Se length = `20`: IPv6 (16 bytes) + Porta (4 bytes Int). Se `0`: Sem endereço declarado. |
| **Timestamp** | 8 bytes | Long (Big-Endian) | Timestamp unix em segundos da geração do pacote (ignorado na decodificação). |

### B. Manutenção da Topologia de Rede e Gerenciamento de Peer-DB
*   **Handshake Timeout Handler**: Se o nó remoto conectar via TCP mas falhar em transmitir o pacote de Handshake completo dentro do tempo limite configurado (`handshake-timeout = 30s`), a conexão é interrompida imediatamente para evitar ataques de esgotamento de conexões (Resource Exhaustion).
*   **Peer Database (`peers.dat`)**: O nó mantém uma base local persistente de nós conhecidos. Novos nós descobertos dinamicamente através do mecanismo de Peer Exchange (`enable-peers-exchange = yes`) são injetados nesta base.
*   **Keep-Alives e Verificação de Atividade**: O nó envia pings periódicos para verificar a integridade da conexão. Se uma conexão TCP ficar ociosa por mais do que `break-idle-connections-timeout = 5m`, a conexão é derrubada para liberar descritores de sockets no sistema operacional.
*   **Mecanismo de Blacklisting (Lista Negra de Nós)**:
    Se um peer enviar uma mensagem corrompida, violar o protocolo, propagar blocos inválidos com assinaturas fraudulentas ou tentar realizar spam de transações na UTX pool:
    *   O IP do nó é banido temporariamente pelo tempo configurado (`black-list-residence-time = 1h`).
    *   Durante a suspensão, qualquer tentativa de conexão de entrada proveniente desse IP é rejeitada no nível de socket.

---

## ⚙️ 3. O Fluxo de Validação de Validadores e o Ciclo Waves-NG

Para escalar o processamento de transações sem comprometer a segurança, a rede adota o protocolo **Waves-NG**, dividindo a criação de blocos em blocos de cabeçalho estáveis e micro-blocos transacionais rápidos.

```
TEMPO DE UM ROUND DE CONSENSO:
|
v [Tempo 0s] - O Minerador A (Líder Eleito) gera o Bloco Chave (Key Block)
+---------------------------------------------------------------------------------+
| Cabeçalho, Assinatura de Consenso (VRF), ID do Bloco Anterior, Provas PoS       | -> Sem transações
+---------------------------------------------------------------------------------+
|
v [Tempo +2s] - Minerador A propaga o primeiro Micro-bloco
  +---------------------------------------------------+
  | transação_1, transação_2, transação_N...          | -> Transações validadas e aplicadas
  +---------------------------------------------------+
|
v [Tempo +4s] - Minerador A propaga o segundo Micro-bloco
  +---------------------------------------------------+
  | transação_N+1, transação_N+2...                   |
  +---------------------------------------------------+
|
v [Tempo X s] - Minerador B (Próximo Líder) assume o Consenso e gera um novo Key Block
+---------------------------------------------------------------------------------+
| Novo Key Block do Minerador B (Fecha o ciclo do round anterior de micro-blocos) |
+---------------------------------------------------------------------------------+
```

### A. O Ciclo Operacional de Blocos e Micro-blocos
1.  **Geração do Key Block (Bloco Chave)**:
    *   O validador eleito pelo algoritmo de Proof of Stake cria um **Key Block**.
    *   Este bloco **não contém transações**. Sua função é conter os metadados do validador, a referência ao bloco anterior e a assinatura criptográfica baseada em VRF que prova que ele ganhou o sorteio matemático do consenso para liderar este round.
    *   O Key Block define o minerador como o **Líder do Round** corrente e é propagado em milissegundos pela rede.
2.  **Streaming de Micro-blocos (Micro Blocks)**:
    *   Como líder autorizado, o nó consome as transações recebidas em sua pool de memória (**UTX Pool**).
    *   A cada intervalo configurado (`miner.micro-block-interval = 1500ms` a `2000ms`), o líder agrupa novas transações válidas em um **Micro-bloco**, assina-o com sua chave privada e propaga-o imediatamente à rede através da mensagem `microblockinv`.
    *   Ao receberem o micro-bloco, os outros nós realizam validações imediatas de assinaturas de transações e scripts RIDE, atualizando temporariamente seu RocksDB de estados. As transações são consideradas confirmadas preliminarmente em menos de 2 segundos!
3.  **Fechamento do Round**:
    *   O ciclo de micro-blocos de um round termina quando o próximo líder ganha o consenso PoS e publica o seu próprio **Key Block**.
    *   Este novo Key Block faz referência ao ID do último micro-bloco de transações gerado pelo líder anterior, selando e consolidando permanentemente todas as transações de forma definitiva e imutável na blockchain.

### B. Distribuição de Taxas Transacionais (Regra de Incentivo Financeiro 40/60)
Para alinhar os incentivos econômicos de segurança e evitar que mineradores subsequentes criem bifurcações de blocos (forks) de forma intencional para capturar taxas do round anterior, a rede divide a receita das taxas de transação coletadas:
*   **40% das Taxas**: São pagas imediatamente ao líder atual do round que colocou a transação dentro de seu Micro-bloco correspondente.
*   **60% das Taxas**: São pagas ao minerador subsequente que fechar o round gerando o próximo Key Block.
*   Isso cria um incentivo para que o próximo minerador colabore ativamente e valide de forma rápida todos os micro-blocos do líder anterior, pois a sua própria recompensa de taxas futuras depende diretamente de referenciar o último micro-bloco produzido.

### C. Limites de Transações por Bloco
*   **Blocos Clássicos (v1/v2)**: Limitados a no máximo **100 transações** por bloco.
*   **Blocos NG / Protobuf (v3/v4/v5)**: Suportam até **6000 transações** de alto desempenho por bloco chave compilado, garantindo vazão para aplicações comerciais reais.

---

## 🧮 4. O Algoritmo de Escolha de Validador (A Matemática do FairPoS e LPoS)

A elegibilidade e a frequência com que um validador assina blocos são ditadas pelo algoritmo **Leased Proof of Stake (LPoS)** e reguladas matematicamente pelo módulo **FairPoS**.

### A. O Conceito de Saldo Gerador (Generating Balance)
Para participar da mineração, uma conta deve possuir poder de voto de stake representado pelo seu Saldo Gerador:
*   **Requisito Mínimo**: A conta deve possuir um saldo gerador mínimo de **1.000 Waves/AMZX** (ou equivalente a $10^{11}$ frações decimais em Satoshis).
*   **Janela de Maturação de 1000 Blocos**: Para evitar manipulações de consenso onde usuários movem stake rapidamente de um nó para outro durante sorteios, o saldo de stake adicionado (por depósitos ou arrendamentos recebidos) leva exatamente **1000 blocos** de estabilidade em disco para amadurecer e passar a ser considerado no cálculo do saldo gerador do validador.

### B. A Equação de Delay do Bloco (Tempo de Espera Obrigatório)
Diferente do Proof of Work, onde os mineradores gastam energia computacional de hardware tentando adivinhar uma assinatura, no FairPoS o nó validador calcula matematicamente um **tempo de espera obrigatório (delay)** em milissegundos para cada bloco. O validador que obtiver o **menor delay** transmite o bloco primeiro e ganha o round.

A equação exata de cálculo do delay implementada pelo algoritmo **FairPoS (V2)** em `PoSCalculator.scala` é definida como:

$$\text{delay (ms)} = T_{\text{min}} + C_1 \cdot \ln \left( 1 - \frac{C_2 \cdot \ln(h)}{\text{baseTarget} \cdot \text{GeneratingBalance}} \right)$$

Onde:
*   $T_{\text{min}}$: O tempo mínimo absoluto permitido entre blocos na rede em milissegundos (ex: `min-block-time = 5000ms`).
*   $C_1$: Uma constante multiplicadora de escala do tempo de rede definida como **$70.000$**.
*   $C_2$: Uma constante de escala de dificuldade definida como **$5 \cdot 10^{17}$** (expressa em formato exponencial `5e17` no código-fonte).
*   $\text{baseTarget}$: O indexador de dificuldade dinâmico ajustado pela rede a cada bloco (veja abaixo).
*   $\text{GeneratingBalance}$: O saldo gerador atual do validador (saldo próprio + leased balance) medido na menor fração da moeda (Satoshis, onde $1\text{ Moeda} = 10^8\text{ Satoshis}$). Como o saldo gerador está no denominador, quanto maior for o stake do nó, menor será a expressão logarítmica, comprimindo o tempo de espera calculável!
*   $h$: O valor normalizado de hit do minerador, calculado como:
    $$h = \frac{\text{Hit}}{\text{MaxHit}}$$
    *   $\text{Hit}$: Um valor pseudo-aleatório de 64 bits derivado criptograficamente (veja abaixo).
    *   $\text{MaxHit}$: O valor máximo possível para o hit de 8 bytes ($2^{64}-1 = 18.446.744.073.709.551.615$).
    *   Assim, $h$ representa uma proporção de probabilidade distribuída uniformemente no intervalo de números reais entre $[0, 1]$.

### C. Como é calculado o valor de `Hit` com VRF (Anti-Grinding)
Para inviabilizar ataques de previsão de blocos futuros (Grinding Attacks), a blockchain gera o valor de `Hit` utilizando a **Função Aleatória Verificável (VRF)**:

```
+------------------------------------+
|   Assinatura do Bloco Anterior     | (32 bytes de HitSource)
+------------------------------------+
                  |
                  v
+------------------------------------+
|  Assinatura VRF (secp256k1/Curve)  | -> Gerada APENAS pelo Minerador usando
|    signVRF(privateKey, hitSource)  |    sua chave privada exclusiva (96 bytes)
+------------------------------------+
                  |
                  v
+------------------------------------+
|   Extrai primeiros 8 bytes (Hit)   | -> Convertido para Inteiro BigInt de 64 bits
+------------------------------------+
                  |
                  v
+------------------------------------+
|   Inserido na Fórmula de Delay     |
+------------------------------------+
```

1.  O nó extrai a assinatura do bloco imediatamente anterior, que serve como uma fonte de hit aleatória e imutável de 32 bytes (`hitSource`).
2.  O nó minerador assina criptograficamente a fonte de hit utilizando a sua chave privada exclusiva através da função VRF:
    $$\text{vrfProof} = \text{signVRF}(\text{privateKey}, \text{hitSource})$$
    A prova VRF resultante possui exatamente **96 bytes** (`GenerationVRFSignatureLength`).
3.  Qualquer nó validador na rede pode auditar e verificar publicamente que a prova gerada é autêntica e válida utilizando a chave pública registrada do minerador:
    $$\text{verifyVRF}(\text{vrfProof}, \text{hitSource}, \text{publicKey})$$
4.  O valor numérico do $\text{Hit}$ é extraído pegando os **primeiros 8 bytes** (`HitSize = 8` bytes) da prova VRF gerada de 96 bytes e convertendo-os em um inteiro de 64 bits (`BigInt`).
5.  Como o minerador não pode controlar a assinatura do bloco anterior e a assinatura VRF exige sua chave privada secreta real, é matematicamente impossível alterar o valor de $\text{Hit}$ para tentar encurtar o tempo de bloco.

### D. Ajuste de Dificuldade Dinâmico (`baseTarget`)
O parâmetro de dificuldade da rede `baseTarget` é reajustado a cada bloco para garantir que o tempo médio de confirmação permaneça estável (ex: `average-block-delay = 60s`).
O algoritmo calcula a média de tempo de geração dos **últimos 3 blocos** da blockchain comparando com o timestamp do bloco de 3 alturas atrás (great-grandparent block):

$$\text{avgDelay} = \frac{\text{Timestamp}_{\text{Current}} - \text{Timestamp}_{\text{GreatGrandParent}}}{3 \cdot 1000}$$

*   **Network is Slow**: Se $\text{avgDelay} > \text{maxDelay}$ (por exemplo, acima de 82 segundos):
    A rede aumenta o `baseTarget` em exatamente 1% para tornar a mineração mais fácil no próximo bloco:
    $$\text{newBaseTarget} = \text{prevBaseTarget} + \max(1, \lfloor\text{prevBaseTarget}/100\rfloor)$$
*   **Network is Fast**: Se $\text{avgDelay} < \text{minDelay}$ (por exemplo, abaixo de 38 segundos):
    A rede diminui o `baseTarget` em exatamente 1% para aumentar a dificuldade de mineração e desacelerar a geração de blocos:
    $$\text{newBaseTarget} = \max(1, \text{prevBaseTarget} - \max(1, \lfloor\text{prevBaseTarget}/100\rfloor))$$
*   **Stable Network**: Se $\text{avgDelay}$ estiver dentro do canal ideal de tempo, a dificuldade é mantida idêntica:
    $$\text{newBaseTarget} = \text{prevBaseTarget}$$

---

## 🤝 5. Manual Completo de Arrendamento LPoS para Usuários (Leasing & Unleasing)

O ecossistema LPoS permite que qualquer detentor de moedas participe da proteção e recompensas da rede sem precisar gerenciar servidores de nós ou abdicar da custódia física de seus fundos.

```
       +-------------------------------------------------------------+
       |                        COLD WALLET                          |
       |                   - Mantém as chaves                        |
       |                   - Custódia total ativa                    |
       +-------------------------------------------------------------+
                               |
                               | Transação de Lease (Type 8)
                               v  (Delega apenas o peso de mineração)
       +-------------------------------------------------------------+
       |                       MINING NODE                           |
       |                   - Saldo real do nó: 0                     |
       |                   - Generating Balance: +1.000.000          |
       +-------------------------------------------------------------+
```

### A. O Mecanismo de Não-Custódia (Segurança Absoluta do Leasing)
*   **Fundos Intocados**: Quando você arrenda suas moedas para um nó validador, os tokens **nunca saem da sua carteira**. Não ocorre nenhuma transferência de fundos.
*   **Delegação Criptográfica**: A transação de `Lease` apenas instrui criptograficamente a blockchain a computar temporariamente o seu saldo como "peso de mineração" no `GeneratingBalance` do nó escolhido.
*   **Imunidade a Golpes**: O validador **não pode** transferir, queimar, gastar ou prender os seus tokens. Suas moedas continuam salvas em seu endereço e protegidas pela sua própria chave privada (e podem estar protegidas em carteiras frias como a Ledger).
*   **Cancelamento Instantâneo**: Se você quiser transferir ou gastar suas moedas, basta enviar uma transação de `LeaseCancel` (Type 9). O arrendamento é quebrado no mesmo instante on-chain e os fundos tornam-se imediatamente líquidos para movimentação.

### B. Estrutura de Transações de Leasing (Payloads JSON e Campos)

#### 1. Transação de Início de Arrendamento: `LeaseTransaction` (Type 8)
Esta transação é disparada pelo usuário arrendador para direcionar o peso de seu saldo para o validador.

##### Estrutura JSON Completa (Campos Decodificados):
```json
{
  "type": 8,
  "version": 3,
  "senderPublicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV",
  "recipient": "3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF",
  "amount": 100000000000, 
  "fee": 100000,
  "timestamp": 1780000000000,
  "proofs": [
    "35ZUnZf16SHe7Kk..."
  ]
}
```
*   `type`: O identificador fixo da transação de arrendamento na blockchain (sempre `8`).
*   `version`: Versão da estrutura de dados transacional (versão `3` para assinaturas criptográficas modernas).
*   `senderPublicKey`: Chave pública do usuário que possui os fundos em Curve25519 (Base58).
*   `recipient`: Endereço de destino do nó validador/minerador.
*   `amount`: A quantidade exata de moedas que se deseja arrendar, especificada na menor unidade indivisível (Satoshis). O exemplo de `100000000000` equivale a **1.000,00000000** moedas.
*   `fee`: Taxa cobrada para registro da transação on-chain (mínimo de `100000` Satoshis ou `0.001`).
*   `proofs`: Array de assinaturas da transação. O primeiro índice `proofs[0]` deve conter a assinatura criptográfica Ed25519 de 64 bytes gerada com a chave privada do usuário.

#### 2. Transação de Cancelamento de Arrendamento: `LeaseCancelTransaction` (Type 9)
Disparada pelo usuário a qualquer momento para desvincular o stake do nó e liberar o saldo.

##### Estrutura JSON Completa:
```json
{
  "type": 9,
  "version": 3,
  "senderPublicKey": "AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV",
  "leaseId": "5ASUNefZs2dLRroid7LPS24PL85K5Y6WZqA1bfQGCHxk",
  "fee": 100000,
  "timestamp": 1780000500000,
  "proofs": [
    "5B74gZMpdzQSP6X..."
  ]
}
```
*   `type`: O identificador fixo de cancelamento de leasing na blockchain (sempre `9`).
*   `leaseId`: O hash identificador exclusivo da transação de leasing original de Tipo 8 (`LeaseId`) que você deseja revogar de forma imediata.

---

### C. Como Funcionam as Recompensas de Arrendamento (Rewards) e Pools

Muitos usuários imaginam que a blockchain realiza o pagamento automático de dividendos e distribuição de moedas diretamente de forma programada on-chain para os arrendadores. **Isso não ocorre no nível de consenso do protocolo de rede.**

#### 1. Recebimento de Recompensas de Blocos e Taxas pelo Validador
De acordo com as regras rígidas de consenso de rede da blockchain, **100% de todas as taxas transacionais recolhidas e 100% das recompensas de blocos novos emitidos entram única e exclusivamente na carteira do validador** (o endereço configurado na chave `wallet.seed` da máquina que assinou o bloco).

#### 2. Distribuição Automatizada por Scripts de Pools (Reward Distributors)
Para remunerar os usuários que arrendaram stake para aumentar o seu poder de mineração, os administradores de nós utilizam **distribuidores de recompensas automatizados baseados em scripts** (conhecidos como *Lease Reward Distributors*):

```
+---------------------------------------------------------------------------------+
|                       BLOCO MINERADO (RECOMPENSAS)                              |
|   100% das moedas geradas entram na Wallet do Nó Minerador                      |
+---------------------------------------------------------------------------------+
                                       |
                                       | Executado periodicamente via Script
                                       v
+---------------------------------------------------------------------------------+
|               SCRIPT OPERADOR DE DISTRIBUIÇÃO DA POOL                           |
|   1. Varre a blockchain e lista todos os blocos minerados pela pool             |
|   2. Consulta a lista de arrendadores ativos naquele bloco                      |
|   3. Desconta a comissão de infraestrutura da pool (ex: 5% a 10%)              |
|   4. Calcula as moedas devidas proporcionalmente ao stake de cada endereço      |
+---------------------------------------------------------------------------------+
                                       |
                                       | Transmite MassTransferTransactions
                                       v
+------------------+          +------------------+          +------------------+
| Carteira User 1  |          | Carteira User 2  |          | Carteira User N  |
|  Rec. Proporcional|         |  Rec. Proporcional|         |  Rec. Proporcional|
+------------------+          +------------------+          +------------------+
```

##### Ciclo de Operação do Distribuidor de Recompensas:
1.  **Auditoria Histórica**: A cada período pré-definido pelo administrador da pool (por exemplo, a cada semana ou a cada 10.000 blocos), roda-se um script distribuidor local que interage com a API REST do nó (`http://localhost:6869`).
2.  **Cálculo da Participação Percentual (Pro-Rata)**:
    *   O script identifica quais blocos foram assinados pelo nó validador naquela janela de tempo.
    *   Para cada bloco minerado, o script consulta quais endereços possuíam arrendamentos ativos e maduros direcionados ao nó.
    *   Calcula a participação pro-rata de cada carteira:
        $$\% \text{ de Participação do Usuário} = \frac{\text{Leased Balance do Usuário}}{\text{Generating Balance Total do Nó}} \times 100$$
3.  **Desconto de Comissão de Infraestrutura (Fee da Pool)**:
    *   Os operadores de nós validadores retêm uma pequena comissão operacional das recompensas geradas, normalmente entre **5% e 10%**.
    *   Esta comissão é usada para cobrir os custos de hospedagem de servidores VPS de alta disponibilidade, largura de banda de rede, firewalls DDoS, monitoramento e manutenção de segurança técnica.
4.  **Distribuição em Lote usando `MassTransferTransaction` (Type 11)**:
    *   Após calcular o saldo líquido de cada arrendador, o script monta de forma automática transações do Tipo 11 (`MassTransferTransaction`).
    *   Uma transação de MassTransfer permite realizar transferências para até **100 endereços diferentes de uma única vez**, compartilhando uma taxa de transação única extremamente reduzida (calculada como `0.001 + 0.0005 * número_de_receptores` moedas).
    *   O nó executa a transação assinada e transfere os Waves/AMZX devidos instantaneamente de forma segura e transparente para a carteira de cada arrendador, sem que estes precisem fazer nada para resgatar.
