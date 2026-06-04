# ⛓️ Manual de Consenso LPoS/FairPoS e Guia de Alteração do Protocolo

Este documento detalha o funcionamento matemático e arquitetural dos mecanismos de consenso **Leased Proof of Stake (LPoS)**, **FairPoS** e **Waves-NG** no ecossistema **AMZX**, fornecendo um guia passo a passo direcionado a engenheiros de core blockchain que queiram modificar ou substituir o algoritmo de consenso do nó.

---

## 📐 1. Matemática do Consenso FairPoS e LPoS

O FairPoS evita a centralização de mineração ("rich-get-richer") comumente vista em algoritmos Nxt tradicionais, introduzindo uma relação não-linear ajustada por tempo para o cálculo do direito de minerar novos blocos.

### A. Cálculo do Hit Gerador (Generator Hit)
Para cada altura de bloco $h$, um nó validador calcula um número pseudo-aleatório chamado **Hit**. Se o hit calculado for menor que o limite matemático de mineração estabelecido pelo estado da rede, o nó ganha o direito de minerar e emitir o Key Block.

Com o recurso de **VRF (Verifiable Random Function)** ativado, o Hit é derivado da prova criptográfica gerada com a chave privada do nó minerador sobre o histórico de entropia anterior (`hitSource`):
$$\text{vrfProof} = \text{signVRF}(\text{privateKey}, \text{hitSource}_{h-100})$$
$$\text{gs} = \text{verifyVRF}(\text{vrfProof}, \text{hitSource}_{h-100}, \text{publicKey})$$
$$\text{Hit} = \text{hit}(\text{gs})$$

Onde o `Hit` resultante é um número inteiro de grande precisão (256 bits) convertido para a escala decimal.

### B. Fórmula de Atraso de Bloco (Block Delay)
O atraso de bloco (tempo mínimo em milissegundos que o minerador deve esperar antes de publicar o próximo bloco) é inversamente proporcional ao seu saldo gerador efetivo (*Generating Balance*, que inclui as moedas clássicas e os arrendamentos de LPoS acumulados):

No **FairPoS**, a fórmula estrita para o cálculo do delay em segundos é definida por:
$$T = C_1 \cdot \ln\left(1 - C_2 \cdot \ln\left(1 - \frac{\text{Hit}}{\text{MaxHit}}\right) \cdot \frac{1}{\text{baseTarget} \cdot \text{Balance}}\right)$$

Onde as constantes padrão de escala do sistema no código Scala são:
*   $C_1 = 70.000$ (Fator de ajuste de tempo)
*   $C_2 = 5 \cdot 10^{17}$ (Constante de suavização exponencial)
*   $\text{MaxHit} = 2^{64} - 1$ (Limite superior de assinatura)
*   $\text{baseTarget}$: O indicador de dificuldade ajustável da rede.
*   $\text{Balance}$: O saldo de mineração efetivo em satoshis (mínimo de $1.000.000.000$ satoshis ou $10$ AMZX para minerar).

Se o tempo atual do relógio do nó ($currentTime$) for superior ao timestamp de referência do bloco anterior mais o atraso de bloco calculado ($T$), o nó minerador gera o bloco chave imediatamente e o transmite para a rede.

---

## ⚡ 2. O Protocolo Waves-NG (Key Blocks e Micro Blocks)

Para atingir vazões de milhares de transações por segundo com tempos de confirmação baixíssimos, a blockchain adota o protocolo de micro-blocos Waves-NG (baseado na proposta Bitcoin-NG).

```
Round de Mineração Liderado pelo Minerador A
=============================================================================
[Key Block (A)] --------> [Micro-Bloco 1] --------> [Micro-Bloco 2] --------> [Micro-Bloco N]
 (Apenas Cabeçalho)       (Contém Txs)             (Contém Txs)             (Contém Txs)
=============================================================================
```

### Características de Emissão:
*   **Key Block**: Não contém transações. É gerado pelo minerador que atinge o hit matemático do PoS. Ele serve apenas para comprovar a liderança do round, conter as assinaturas criptográficas de consenso e selar a dificuldade (`baseTarget`).
*   **Micro-Blocos**: O líder do round passa a emitir micro-blocos contendo transações reais do UTX Pool de forma contínua em intervalos dinâmicos ajustados (ex: 1.5 a 2 segundos). Esses micro-blocos são assinados pelo líder e verificados de forma assíncrona pelos outros validadores.
*   **Divisão de Taxas (40/60 Split)**:
    *   **40%** de todas as taxas das transações incluídas nos micro-blocos atuais são pagas ao minerador líder que as incluiu.
    *   **60%** das taxas são destinadas de forma obrigatória ao minerador líder do **próximo round** (que validará e fechará o bloco contendo os micro-blocos passados), incentivando a cooperação mútua e prevenindo egoísmo de mineração.

---

## 🛠️ 3. Guia do Desenvolvedor: Como Alterar ou Substituir o Mecanismo de Consenso

Se você deseja alterar a lógica de dificuldade, o tempo de bloco ou substituir o algoritmo de consenso do ecossistema AMZX (por exemplo, migrar de FairPoS para Proof of Authority - PoA ou Proof of Stake clássico), siga o mapa de arquivos de engenharia abaixo.

### A. Mapeamento de Classes e Arquivos Críticos:

1.  **[PoSSelector.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/consensus/PoSSelector.scala)**:
    *   **Função**: É a classe de orquestração central de consenso do nó. Ela valida a assinatura de geração dos blocos que entram na rede e dita a ordem de prioridade.
    *   **Ponto de Partida**: Modifique o método `validateBlockDelay` para alterar as regras de checagem de timestamp de blocos concorrentes e `validateBaseTarget` para redefinir as margens de erro de dificuldade.
2.  **[PoSCalculator.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/consensus/PoSCalculator.scala)** (localizado na pasta `com.wavesplatform.consensus`):
    *   **Função**: Contém os algoritmos e equações de cálculo de delay e ajuste de dificuldade. Ele bifurca o comportamento do nó de acordo com a ativação de features (Nxt PoS vs FairPoS V1 vs FairPoS VRF).
    *   **O que Alterar**: Se quiser redefinir a fórmula matemática do delay, modifique o método `calculateDelay` no trait `PoSCalculator` e suas implementações `FairPoSCalculator` e `NxtPoSCalculator`.
3.  **[GeneratingBalanceProvider.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/consensus/GeneratingBalanceProvider.scala)**:
    *   **Função**: Calcula o saldo de mineração efetivo de um endereço, lendo o saldo total e subtraindo/somando os arrendamentos ativos (*leased balances*).
    *   **O que Alterar**: Se quiser desativar o arrendamento ou mudar a porcentagem de influência do saldo arrendado no poder de voto/mineração, modifique a função `generatingBalance`.
4.  **[Miner.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/mining/Miner.scala)** (localizado em `com.wavesplatform.mining`):
    *   **Função**: É o ator do Akka/Pekko encarregado de rodar o loop periódico de mineração. Ele monitora o tempo de sistema local, interage com o `PoSSelector` para verificar se atingiu o delay de bloco e monta o Key Block físico.
    *   **O que Alterar**: Altere as regras de agendamento de tentativas de mineração no método `scheduleNextBlockGeneration`.

---

### B. Passo a Passo para Alteração de Consenso:

```
           [Definir Nova Fórmula de Consenso em Scala]
                               |
                               v
          [Modificar Métodos em PoSCalculator.scala]
           - calculateDelay
           - calculateBaseTarget
                               |
                               v
        [Ajustar Validação de Rede em PoSSelector.scala]
           - validateBlockDelay
           - validateBaseTarget
                               |
                               v
       [Atualizar Actor de Agendamento em Miner.scala]
                               |
                               v
        [Compilar Projeto e Validar com node-it Docker]
```

1.  **Defina a nova lógica de cálculo de tempo**: Abra o arquivo `PoSCalculator.scala` e escreva o novo algoritmo. Para um consenso de tempo fixo (como PoA de 15 segundos fixos), force a função `calculateDelay` a retornar sempre `15000` (milissegundos) independente do hit gerador ou saldo do nó.
2.  **Adapte a dificuldade adaptativa**: Se o tempo fixo for adotado, a dificuldade (`baseTarget`) deixa de fazer sentido. Ajuste `calculateBaseTarget` para manter o target fixo ou estático.
3.  **Ajuste os Validadores de Blocos**: Abra o `PoSSelector.scala` e certifique-se de que os métodos de validação (`validateBlockDelay` e `validateBaseTarget`) estejam conformes com o novo modelo estático/dinâmico de tempo. Caso contrário, os blocos minerados por você serão classificados como inválidos pelos outros nós validadores da rede.
4.  **Ajuste o Quórum e Agendador**: Se for migrar para uma rede baseada em autorização estática onde apenas IPs/Chaves específicas de autoridade mineram, adapte a semente de mineração e o quórum de conexões em `Miner.scala` no trait `Miner`.
5.  **Validação de Compilação**: Rode `sbt compile` para garantir que as assinaturas e heranças das classes de consenso não foram quebradas.

### ⚠️ O que NÃO Alterar de Forma Alguma:
*   **Serialization / Deserialization de Cabeçalhos de Bloco**: Os atributos de consenso (`baseTarget` e `generationSignature`) são serializados diretamente nos bytes do cabeçalho de bloco. Nunca remova ou altere os tipos de dados dessas duas variáveis nos models `BlockHeader` ou `NxtLikeConsensusBlockData`, pois isso quebrará a compatibilidade de rede binária P2P e a gravação de blocos no RocksDB.
*   **Controle de Ativação de Features**: Não tente ignorar o mecanismo de ativação progressiva de features (`BlockchainFeatures`) do código. Se quiser forçar o novo consenso a rodar desde a altura zero, configure-o como ativado por default no arquivo `genesis-generator.conf` nas diretivas de pre-ativação, ao invés de remover as checagens estáticas `isFeatureActivated` no código Scala.
