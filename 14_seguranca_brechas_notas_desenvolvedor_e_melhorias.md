# 🛡️ Segurança, Vetores de Ataque, Notas do Desenvolvedor e Melhorias AMZX

Este documento aborda a segurança física e lógica de nós e validadores, mapeia vulnerabilidades e vetores de ataque potenciais, fornece notas técnicas detalhadas para desenvolvedores que desejam alterar o código-fonte da blockchain (consenso, endereçamento, transações) e lista um plano de melhorias acionáveis para a equipe de desenvolvimento atual.

---

## 🔒 1. Segurança e Hardening para Nós e Validadores

A operação de nós validadores em ambiente de rede de produção exige uma postura rígida de segurança da infraestrutura de servidores (Hardening) para evitar sequestro de saldos, ataques de negação de serviço e corrupção de estado.

```
                    IP Público (Internet)
                             |
                             v  [Filtro de Firewall UFW]
               +-----------------------------+
               | Porta P2P 6868 (Liberada)   |
               | Porta SSH 22 (Restrita)     |
               +-----------------------------+
                             |
                             +-------------------+
                             |                   |
                             v                   v
                     [Proxy Reverso NGINX]   [Local Only gRPC]
                     Porta HTTP API 6869     Porta gRPC 6887
                     (SSL/TLS + Rate-Limit)      (Matcher Only)
                             |
                             v
                     [AMZX Scala Node JVM]
                     - RocksDB Encrypted (Optional)
                     - API Key Double Hash
```

### A. Proteção de Portas de Rede (Firewall UFW)
Nós validadores **nunca** devem expor as portas REST API (`6869` do Nó e `6886` do Matcher) diretamente para a internet. 
*   **Porta P2P (`6868`)**: Deve ser aberta publicamente para permitir que outros nós validadores sincronizem blocos de forma descentralizada.
*   **Portas de API (`6869` / `6886`)**: Devem ser restritas ao localhost (`127.0.0.1`) ou protegidas por um proxy reverso (ex: NGINX ou HAProxy) que implemente criptografia SSL/TLS, limitação de taxa de chamadas (*rate-limiting*) e filtragem de pacotes suspeitos.
*   **Portas gRPC (`6887` / `6870`)**: Devem aceitar tráfego exclusivamente originado do IP local do Matcher DEX.

### B. Sincronização de Relógio Perfeita (NTP / Chrony)
O motor de consenso FairPoS e a emissão acelerada de micro-blocos Waves-NG apoiam-se estritamente na precisão temporal dos timestamps de blocos concorrentes. Se o relógio do seu servidor validador estiver dessincronizado por mais de **10 segundos** em relação à rede:
1.  Os blocos gerados por você serão classificados como inválidos e rejeitados pelos outros nós (Timestamp is in the future / too far past).
2.  Você perderá recompensas de mineração (*forged blocks*).
3.  **Ação Recomendada**: Instale e configure o daemon **Chrony** no Linux para sincronizar continuamente o tempo via NTP contra pools de relógios atômicos confiáveis:
    ```bash
    sudo apt install chrony
    sudo systemctl enable --now chrony
    chronyc sources
    ```

### C. Proteção da Chave Privada da Carteira (Wallet dat)
*   **Chave Quente em Servidor**: O nó validador precisa da chave privada de mineração ativa em memória JVM quente para poder assinar os blocos que minerar. Se o servidor for invadido, a chave privada em texto plano ou o arquivo `wallet.dat` pode ser roubado.
*   **Estratégia Recomendada**: Adote o **LPoS (Leased Proof of Stake)**. Mantenha os seus fundos AMZX ricos e importantes guardados em uma **Cold Wallet** offline (assinada via hardware) e arrende (*lease*) todo o seu saldo para um endereço de nó quente gerador online que possui saldo real zero. Desta forma, se o seu servidor minerador ativo for invadido, o invasor obterá apenas uma chave quente sem moedas para roubar, e os fundos da rede permanecerão perfeitamente seguros na Cold Wallet original.

---

## ⚠️ 2. Vetores de Ataque Potenciais e Brechas de Segurança

### A. Frontrunning e Sandwich Attacks no Matcher DEX
*   **O Cenário**: Diferente das DEX baseadas em AMM (Automated Market Makers) na EVM, o Matcher AMZX opera com um livro de ofertas centralizado em memória RAM ordenado por um algoritmo TreeMap de correspondência assíncrona.
*   **A Vulnerabilidade**: Como as ordens de compra e venda são transmitidas via REST/WebSocket para o Matcher centralizado e o Matcher decide deterministicamente a ordem de cruzamento e geração da transação `ExchangeTransaction` (ID 7) on-chain, um operador malicioso de Matcher (ou um invasor que tome controle do serviço) pode inspecionar a fila de ordens pendentes em memória RAM e injetar ordens próprias à frente (Frontrunning) ou cercar ordens grandes de usuários (Sandwich) para obter lucros sem risco de preço.
*   **Mitigação**: Implementação estrita de auditoria forense criptográfica de carimbos de data/hora (timestamps) em todas as ordens cruzadas, garantindo que o Matcher obedeça de forma comprovável à fila síncrona FIFO (First In, First Out) de recepção de chaves.

### B. Ataques DDoS na Rede P2P e UTX Pool
*   **O Cenário**: Usuários maliciosos podem inundar as portas de API pública e P2P com milhões de transações de dados de tamanho enorme (150KB cada) ou invocações de scripts inválidas com o objetivo de exaurir a memória heap da JVM.
*   **A Vulnerabilidade**: Se a JVM estourar a memória RAM disponível, ela gerará um travamento de `OutOfMemoryError` e derrubará o nó.
*   **Mitigação**: Ativar o parâmetro `-XX:+ExitOnOutOfMemoryError` para derrubar e reiniciar o nó rapidamente por systemd, acoplado com uma taxa de taxas dinâmica (dynamic gas multipliers) onde o tamanho em bytes de payloads de dados e transações de transferência aumente o custo em AMZX exponencialmente se a UTX Pool estiver próxima de sua capacidade máxima de limite de tamanho configurado (`amzx.utx.max-size`).

### C. Ataques de Divisão de Consenso e Reversão (Rollbacks)
*   **O Cenário**: Se um minerador ou um pool minerador atingir mais de **51%** do poder total de stakes arrendados (Leased Balance) na rede, ele pode reescrever transações passadas gerando e propagando uma cadeia de blocos paralela secreta mais longa.
*   **A Vulnerabilidade**: Ao ser propagada, os outros nós serão forçados a realizar um Rollback estrito de até **100 blocos** para se ajustarem à cadeia mais longa, cancelando as transações que ocorreram na ramificação descartada e gerando dupla despesa (double spending).
*   **Mitigação**: O parâmetro de configuração `amzx.blockchain.rollback-limit = 100` protege a blockchain contra rollbacks arbitrários profundos. Qualquer tentativa de alteração de cadeia superior a 100 blocos é sumariamente banida pela camada P2P do nó.

---

## 🛠️ 3. Notas Detalhadas para o Desenvolvedor (Guia de Alterações)

Esta seção atua como o manual mestre orientando o desenvolvedor sobre como navegar, alterar ou preservar subsistemas chaves do ecossistema Waves/AMZX no código-fonte em Scala.

### Mapeamento de Onde Alterar e Suas Implicações:

```
+=============================================================================+
|                      AMZX SOURCE CODE ARCHITECTURE MAP                      |
+=============================================================================+
| Área de Alteração    | Arquivos Scala Críticos     | O Que e Para Que Alterar?      |
+----------------------+-----------------------------+--------------------------------+
| Algoritmos de        | PoSCalculator.scala         | Alterar equações matemáticas   |
| Consenso e           | PoSSelector.scala           | de block delay, FairPoS, LPoS, |
| Dificuldade          | Miner.scala                 | baseTarget ou migração de PoS  |
|                      |                             | para PoA de tempo fixo.        |
+----------------------+-----------------------------+--------------------------------+
| Estrutura de         | Recipient.scala             | Alterar o layout de bytes de   |
| Endereçamento e      | Address.scala               | contas (comprimento, versão,   |
| Customização         |                             | constantes de hash e checksum) |
+----------------------+-----------------------------+--------------------------------+
| Regras e             | TxValidator.scala           | Adicionar novos tipos de       |
| Validação de         | impl/*Validator.scala       | transações ou redefinir taxas  |
| Transações           |                             | e tamanhos de bytes máximos.   |
+=============================================================================+
```

### A. Área de Consenso (PoS Selector, Calculator & Miner)
*   **Onde**:
    *   👉 **[PoSCalculator.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/consensus/PoSCalculator.scala)**
    *   👉 **[PoSSelector.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/consensus/PoSSelector.scala)**
    *   👉 **[Miner.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/mining/Miner.scala)**
*   **O que ALTERAR**: Se você precisar mudar o tempo entre os blocos (ex: fixar em 10 segundos) ou alterar o peso dos arrendamentos. Mude o método `calculateDelay` na classe `PoSCalculator` para ignorar o Hit de curva elíptica e retornar um valor estático (ex: `10000` milissegundos).
*   **O que NÃO ALTERAR**: As assinaturas de geração dos blocos que são serializadas e gravadas nos bancos RocksDB (`generationSignature` de 32 bytes). Remover esses campos quebrará a serialização de persistência física em banco de dados e gerará crashes generalizados de inicialização.

### B. Área de Endereçamento e Contas (Recipient & Address)
*   **Onde**:
    *   👉 **[Recipient.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/account/Recipient.scala)**
*   **O que ALTERAR**: Se você precisar aumentar a segurança criptográfica mudando o tamanho do hash de conta (`HashLength`) de 20 para 32 bytes (suportando hashes puros Keccak256 sem truncamento) ou redefinindo o tamanho de checksum (`ChecksumLength`) de 4 para 8 bytes. Altere as variáveis constantes estáticas dentro de `object Address`:
    ```scala
    val HashLength: Int = 32
    val ChecksumLength: Int = 8
    ```
*   **O que NÃO ALTERAR**: O byte de Versão do endereço (`AddressVersion = 1`). Alterá-lo impedirá de forma irreversível que chaves antigas ou assinaturas em bibliotecas e hardware wallets de clientes (Ledger) decodifiquem os novos endereços se não houver um fork coordenado em toda a base instalada de softwares clientes de forma síncrona.

### C. Área de Pipelines de Transações e Validadores
*   **Onde**:
    *   👉 **[TxValidator.scala](file:///home/diegooris/Documentos/amzblockchain/waves/node/src/main/scala/com/wavesplatform/transaction/TxValidator.scala)**
    *   Módulo correspondente na subpasta `/waves/node/src/main/scala/com/wavesplatform/transaction/impl/` para cada classe (ex: `TransferTxValidator.scala`, `IssueTxValidator.scala`).
*   **O que ALTERAR**: Se a sua equipe reguladora precisar impor novas políticas econômicas na rede, como taxas base de transações flutuantes, ou aumentar o limite máximo de tamanho de campos de anexo (Attachment de 140 bytes para 256 bytes) ou restringir a criação de ativos customizados por endereços que não tenham sido validados. Altere as rotinas dentro de `impl/*Validator.scala`.
*   **O que NÃO ALTERAR**: A estrutura estrita de bytes de payloads e o alinhamento das chaves de indexação de banco de dados do RocksDB. Qualquer mudança no leiaute físico do payload que é serializado on-chain tornará os blocos minerados anteriormente incompatíveis de serem lidos, exigindo a declaração de uma nova Gênesis.

---

## 📈 4. Plano de Melhorias Acionáveis para a Equipe de Engenharia Atual

Abaixo está estruturado o plano de refinamento arquitetural que a equipe atual de engenheiros pode aplicar para aumentar a performance de desenvolvimento e expandir as capacidades do ecossistema AMZX:

### Melhoria A: Suporte a Módulo de Segurança em Hardware (HSM / PKCS#11)
*   **Objetivo**: Retirar a chave privada quente de mineração da memória RAM exposta do nó validador.
*   **Como Fazer**: Implementar conexões criptográficas via padrão **PKCS#11** na classe `Miner.scala` e biblioteca `crypto` do nó, delegando as operações de assinatura digital de blocos para um hardware de HSM externo dedicado (ou barramento AWS CloudHSM). A chave privada nunca sairá do hardware seguro, blindando o validador contra ataques de intrusão lógica de sistema operacional.

### Melhoria B: Otimização de Performance de Compilação SBT (Thin Client & Parallel JVM)
*   **Objetivo**: Reduzir os tempos exaustivos de compilação fria Scala do nó e Matcher, acelerando o ciclo de CI/CD.
*   **Como Fazer**: 
    1. Ative a execução em paralelo do SBT inserindo no arquivo `.sbtopts`:
       `-D-Dsbt.compiler.close-on-idleness=true -D-Dsbt.supershell=false`
    2. Configure o SBT para usar compiladores incrementais robustos através do sistema **Zinc** e ative o uso do SBT Thin Client:
       `sbt --client compile`

### Melhoria C: Monitoramento de Telemetria com Prometheus e Grafana
*   **Objetivo**: Obter visibilidade em tempo real de gargalos de rede gRPC, latência de RocksDB e velocidade de correspondência de ordens.
*   **Como Fazer**: Configurar e expor as métricas do sistema de atores Akka/Pekko através do módulo **Kamon** ou **Prometheus Akka HTTP exporter**, coletando dados operacionais críticos como tempos de processamento da função síncrona `doMatch` e tamanho da fila de transações pendentes na UTX mempool.
