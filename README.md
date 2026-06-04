# 📚 Portal de Documentação Técnica - Ecossistema AMZX (Amazonic One)

Bem-vindo à central de documentação e engenharia definitiva do ecossistema **AMZX** (Node Blockchain Scala & Matcher DEX). 

Este repositório de documentos foi projetado com nível de rigor de produção, detalhando de forma exaustiva o funcionamento interno do sistema — desde os algoritmos criptográficos moleculares até as rotas de APIs de gerenciamento de carteiras, pontes de comunicação gRPC de alta velocidade, integrações com a MetaMask e o motor de correspondência em memória do Matcher DEX.

---

## 🗺️ Mapa da Documentação (Estrutura de Arquivos)

Abaixo está o índice estruturado dos documentos disponíveis, cobrindo cada camada do sistema:

### 1. [📖 Visão Geral do Ecossistema AMZX e Rebranding (Scala-First)](file:///home/diegooris/Documentos/amzblockchain/docs/0_visao_geral_e_rebranding.md)
*   **Conteúdo**: Arquitetura geral do ecossistema, hierarquia e sistemas de atores **Scala (Akka / Pekko)** do Nó e do Matcher DEX, funcionamento acoplado da extensão de baixo nível **`DEXExtension`**, e a solução de engenharia para o rebranding de `"Waves"` para `"AMZX"`, contornando limitações de compatibilidade binária com o JAR compilado.

### 2. [⚙️ Configuração e Inicialização do Ledger AMZX](file:///home/diegooris/Documentos/amzblockchain/docs/1_configuracao_blockchain.md)
*   **Conteúdo**: Estruturas de arquivos de configuração baseados no padrão **HOCON**, parametrização de Magic Bytes (Chain ID), ajuste fino de tempos e limites de transação, e o manual completo de criação de blocos gênesis customizados através do `GenesisBlockGenerator` ou computação de assinaturas assistidas em console.
*   **Dicionário de Protocolos**: Tabela detalhada catalogando cada uma das **25 Consensus Protocol Features** (recursos ativados na cadeia de blocos).

### 3. [⛓️ Redes, Nós Validadores e Consenso LPoS/FairPoS](file:///home/diegooris/Documentos/amzblockchain/docs/2_nos_e_validadores.md)
*   **Conteúdo**: Criptografia de curvas elípticas **Curve25519** versus **secp256k1**, cálculo binário byte-a-byte de geração de endereços da rede, protocolo de handshake TCP usando Netty com offset de pacotes binários, e o motor de consenso **Waves-NG** de blocos chaves e micro-blocos (taxas de divisão 40/60).
*   **Fórmulas Matemáticas**: Equações matemáticas estritas do algoritmo **FairPoS/LPoS** para cálculo de block delay, VRF (Verifiable Random Functions), hits geradores e ajustes dinâmicos de baseTarget (+1% / -1%).
*   **Leasing Guide**: Manual passo a passo para arrendamento (*leasing*) não-custodial de poder de stake, com esquemas JSON de transações de leasing (ID 8) e cancelamento (ID 9), além do fluxo de pagamento de pools usando `MassTransfer` (ID 11).

### 4. [📈 Configuração e Integração do AMZX Matcher DEX](file:///home/diegooris/Documentos/amzblockchain/docs/3_matcher_dex.md)
*   **Conteúdo**: O motor de correspondência financeiro (Matching Engine), ordenamento TreeMap em memória RAM das bids e asks, algoritmo síncrono tail-recursive `doMatch` e geração de transações de troca on-chain.
*   **APIs WebSocket**: Especificação exaustiva do canal WebSocket `/ws/v0`, schemas JSON de mensagens enviadas por clientes (como inscrições `obs` e `aus` com tokens JWT criptografados) e dados retornados pelo servidor (deltas de livros de ofertas e atualizações de saldos privados).

### 5. [🚀 Guia de Inicialização, Monitoramento e Diagnóstico (Operações)](file:///home/diegooris/Documentos/amzblockchain/docs/4_inicializacao_e_operacao.md)
*   **Conteúdo**: Manual de comandos para inicialização rápida usando o script consolidado `start_amzx.sh`, comandos de monitoramento de logs em tempo real, depuração de erros comuns de inicialização (como assinaturas inválidas de gênesis e refusão de canais gRPC) e procedimentos de encerramento seguro de serviços.
*   **Wallet REST API**: Guia de controle programático de carteiras contendo chamadas **Curl**, cabeçalhos de autenticação e payloads reais em formato JSON para criação, listagem, exportação de sementes secretas e exclusão de endereços de posse do nó.

### 6. [🛠️ Referência Técnica e Desenvolvimento (Guia do Desenvolvedor)](file:///home/diegooris/Documentos/amzblockchain/docs/5_referencia_tecnica_e_desenvolvimento.md)
*   **Conteúdo**: Tabela mestre de mapeamento de portas de rede de todos os serviços, assinaturas gRPC baseadas em Protocol Buffers, e otimizações de JVM (ZGC de baixa latência e tratamento de estouros de heap).
*   **MetaMask Integration**: Detalhamento técnico da camada de compatibilidade Ethereum RPC (`POST /eth`), cálculo dinâmico de Chain ID, fator de multiplicação de decimal ($10^{10}$) de Satoshis para Wei, decodificação secp256k1 e o gerador de ABI para chamadas Web3 (`GET /eth/abi/{address}`).
*   **Contratos RIDE**: Análise estática de complexidades estritas em código funcional RIDE, transações de implantação (ID 13) e invocação (ID 16), e lógica de implementação de métodos `@Verifier` e `@Callable`.

### 7. [🔌 Integração de Baixa Latência: AMZX Node & Matcher DEX](file:///home/diegooris/Documentos/amzblockchain/docs/6_integracao_node_matcher.md)
*   **Conteúdo**: Arquitetura interna de fusão de streams gRPC em tempo real (Blockchain updates, UTX mempool e DataReceived), máquina de estados estrita do Matcher DEX (`Normal`, `TransientRollback`, `TransientResolving`), resolução matemática rigorosa de forks baseada em pesos acumulados `baseTarget` e rollback de 100 blocos.
*   **Mecânica de Cancelamento**: Dicionário conceitual de saldos operacionais e tabela de corrida crítica com todos os **45 casos de transações** conflitantes mapeados e traduzidos para AMZX.

### 8. [🧪 Ambientes de Teste, Simulação e Desenvolvimento](file:///home/diegooris/Documentos/amzblockchain/docs/7_ambientes_de_teste_e_desenvolvimento.md)
*   **Conteúdo**: Guia para criação de redes de teste locais (Private Testnets) com Docker, credenciais da Rich Account gênesis, aceleração de sincronização via RocksDB Tarball, comandos SBT (`sbt checkPR`, `sbt packageAll`), testes multi-nó com `node-it` e o Ride Runner offline.

### 9. [🦊 Compatibilidade Ethereum e MetaMask a Fundo (EVM-like Layer)](file:///home/diegooris/Documentos/amzblockchain/docs/8_compatibilidade_ethereum_e_meta_mask_profundo.md)
*   **Conteúdo**: Funcionamento interno da emulação EVM-RPC (`/eth`), curvas secp256k1 vs Curve25519, isomorfismo de endereços, fator multiplicador decimal de Satoshis para Wei ($10^{10}$) e o gerador de ABI REST dinâmico (`/eth/abi/{address}`).

### 10. [⚙️ Manual de Consenso LPoS/FairPoS e Modificação do Protocolo](file:///home/diegooris/Documentos/amzblockchain/docs/9_manual_de_consenso_e_alteracao_protocolo.md)
*   **Conteúdo**: Equações matemáticas estritas do FairPoS/LPoS, algoritmo de Hits geradores, divisão de taxas Waves-NG (40/60) e guia completo para alterar ou substituir o mecanismo de consenso no código Scala (`PoSCalculator.scala`, `PoSSelector.scala`, etc.).

### 11. [🪪 Geração de Endereços e Customização da Camada de Contas](file:///home/diegooris/Documentos/amzblockchain/docs/10_geracao_de_enderecos_e_customizacao_address.md)
*   **Conteúdo**: Layout exaustivo byte-a-byte do endereço de 26 bytes, dupla cifragem de hash, e guia passo a passo com variáveis exatas para customizar o comprimento do endereço em `Recipient.scala` e RocksDB.

### 12. [🧾 Catálogo de Transações, Layouts Binários e Pipeline de Validação](file:///home/diegooris/Documentos/amzblockchain/docs/11_tipos_de_transacoes_e_validacao.md)
*   **Conteúdo**: Catálogo completo das transações ID 1 a 18, layouts binários físicos de transmissão, taxas regulamentares, e a máquina de pipeline stateless/stateful de validação.

### 13. [🌐 Guia de Referência Completa da API REST Full do Ecossistema](file:///home/diegooris/Documentos/amzblockchain/docs/12_api_rest_full_completa.md)
*   **Conteúdo**: Tabela exaustiva de endpoints públicos e administrativos do Nó (porta 6869) e Matcher DEX (porta 6886). Inclui comandos Curl completos, cabeçalhos de API Key hash, payloads JSON de produção e códigos HTTP de erros.

### 14. [🧠 Smart Contracts, Wallets, DAO e Ciclo de Vida de Tokens](file:///home/diegooris/Documentos/amzblockchain/docs/13_smart_contracts_wallets_dao_e_tokens_profundo.md)
*   **Conteúdo**: Análise de complexidade estática RIDE (limite de 4000), diferenças entre dApps, Smart Accounts e Smart Assets, criação de NFTs, geração de carteiras BIP39, coexistência de curvas elípticas e um **código de DAO completo em RIDE v6** pronto para produção.

### 15. [🛡️ Segurança, Vetores de Ataque, Notas do Desenvolvedor e Melhorias](file:///home/diegooris/Documentos/amzblockchain/docs/14_seguranca_brechas_notas_desenvolvedor_e_melhorias.md)
*   **Conteúdo**: Hardening de servidores validadores (NTP Chrony, firewalls, proxies reverse), análise de vetores de ataque (Frontrunning, DDoS, Sybil), notas para alterações seguras do desenvolvedor e o roteiro de melhorias sugeridas (HSM, SBT Zinc, telemetria Prometheus).

### 16. [📡 Arquitetura de Rede, Protobuf, Benchmarks e Dicionário de Classes](file:///home/diegooris/Documentos/amzblockchain/docs/15_arquitetura_de_rede_protobuf_benchmarks_e_classes_mestres.md)
*   **Conteúdo**: Detalhamento de esquemas Protobuf (`block.proto`, `transaction.proto`), matemática atômica de Consensus FairPoS/LPoS, canais Netty TCP, testes automatizados (`node-it` Fork simulations) e o dicionário mestre de classes Scala (Nó & Matcher).

### 17. [📡 Diagnóstico Técnico de Integração: DEXExtension, Versionamento e Incompatibilidades do Matcher DEX](file:///home/diegooris/Documentos/amzblockchain/docs/16_diagnostico_extensao_e_incompatibilidade_matcher.md)
*   **Conteúdo**: Análise exaustiva do acoplamento do nó da blockchain AMZX e o Matcher DEX por meio da `DEXExtension`, detalhamento técnico das incompatibilidades de versão na JVM (Scala 3 vs 2.13, Akka vs Pekko, Protobuf runtime v4 vs v3, dependência cega da versão 1.4.13) e roteiro de migração passo a passo completo.

### 18. [🛡️ Criptografia e Segurança da Blockchain AMZX (Rigor Técnico)](file:///home/diegooris/Documentos/amzblockchain/docs/17_criptografia_e_seguranca_blockchain.md)
*   **Conteúdo**: Arquitetura criptográfica detalhada do nó AMZX, catálogo de curvas elípticas (Curve25519, secp256k1, BLS12-381), duplo hashing Keccak-Blake2b, provedores de JVM dinâmicos de alta performance (ACCP/Conscrypt) e mitigação ativa de chaves públicas elípticas fracas de baixa ordem.

### 19. [📉 Criptografia e Segurança do Matcher DEX (Auditoria e Brechas)](file:///home/diegooris/Documentos/amzblockchain/docs/18_criptografia_e_seguranca_matcher_dex.md)
*   **Conteúdo**: Auditoria de segurança criptográfica do Matcher DEX, assinaturas de ordens MetaMask EIP-712 (secp256k1), autenticação JWT e API Key hash, e detalhamento técnico de brecha operacional crítica (vulnerabilidade de DoS e Spoofing) por falta de blacklist de chaves Curve25519 fracas.

### 20. [📖 Manual Operacional e de Desenvolvimento AMZX (Guia Definitivo)](file:///home/diegooris/Documentos/amzblockchain/docs/19_manual_operacional_do_usuario_e_desenvolvedor.md)
*   **Conteúdo**: Manual de referência consolidado passo a passo para inicialização de nós de rede, mineração de validadores, geração de redes customizadas, configuração do Matcher DEX, derivação algorítmica de endereços determinísticos via Nonce, fluxos de arrendamento de staking (Leasing e cancelamento), e regras de recompensas de rede.

---

## ☕ 12. Ambiente de Runtime e Compilação Homologado (Requisito Crítico)

Conforme documentado oficialmente no arquivo [README.md da Blockchain](file:///home/diegooris/Documentos/amzblockchain/waves/README.md), o ecossistema AMZX possui requisitos rígidos de infraestrutura para garantir estabilidade, imunidade a vazamento de memória e compatibilidade binária:

### ⚠️ Requisito de JVM: OpenJDK 17 (LTS)
*   **Versão Recomendada**: **OpenJDK 17** (`openjdk-17-jre` / `adoptopenjdk17`).
*   **Incompatibilidade do Java 21**: O uso de Java 21 ou versões mais recentes **não é homologado** para a blockchain ou para o Matcher DEX. A execução em Java 21 gera avisos severos de reflexão ilegal profunda, quebras de linkage em tempo de execução ao instanciar extensões gRPC dinâmicas (`DEXExtension`) e riscos de vazamento de memória ou comportamento indefinido nas bibliotecas internas de criptografia e concorrência baseadas em atores.
*   **Caminho do Java**: `/usr/lib/jvm/java-1.17.0-openjdk-amd64` (ou o caminho específico de instalação do OpenJDK 17 no sistema Linux).

### 🛠️ Matriz de Versões de Compilação
- **AMZX Node (Blockchain Core & waves-ext)**: Compilado nativamente com **Scala 3.8.3** utilizando SBT.
- **AMZX Matcher DEX**: Compilado nativamente com **Scala 2.13.14** utilizando SBT.
- **Ferramenta de Build**: SBT 1.9.x+ rodando sob a JVM do **OpenJDK 17** (`JAVA_HOME` explicitamente apontando para o JDK 17).

---

## 💎 Filosofia de Design e Práticas Recomendadas

1.  **Segurança Criptográfica Primeiro**:
    -   Nunca armazene sementes ou chaves privadas em texto plano em ambientes de produção.
    -   Utilize hashes duplicados SHA-256 Base58 para credenciais de API REST (`api-key-hash`).
    -   Adote a estratégia de arrendamento (LPoS) para minerar: mantenha os fundos ricos em carteiras frias offline (Cold Wallets) e arrende o poder de stake para nós quentes online com saldo real nulo.
2.  **Sincronização de Relógio Estrita (NTP)**:
    -   O algoritmo FairPoS e o empacotamento de micro-blocos Waves-NG exigem precisão absoluta de tempo de rede. Certifique-se de que deamons locais como **Chrony** ou **NTPd** estejam ativos e sincronizados com servidores confiáveis (ex: `pool.ntp.org`).
3.  **Monitoramento e Resiliência**:
    -   Configure proxies reversos com limitação de requisições (*rate-limiting*) e criptografia SSL/TLS (NGINX/HAProxy) na frente das portas de APIs públicas e canais WebSocket para proteção contra ataques DDoS.
    -   Configure as diretivas de alocação de Heap estática na JVM do nó e do Matcher (`-Xms` idêntico ao `-Xmx`) para mitigar travamentos de redimensionamento e pausas indesejadas de Garbage Collection.

