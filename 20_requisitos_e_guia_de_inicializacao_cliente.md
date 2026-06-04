# 🔷 Guia de Requisitos, Inicialização e Preenchimento de Dados: AMZX Blockchain

Este documento serve como um manual operacional completo de ponta a ponta para orientar clientes, parceiros e operadores sobre como preparar o ambiente, preencher os parâmetros de inicialização e colocar em execução o nó da blockchain **AMZX** e o motor de liquidez **Matcher DEX**.

---

## 📋 1. Requisitos de Software (Pré-requisitos)

Para que todo o ecossistema compile e execute com segurança e sem conflitos, o servidor ou máquina local (preferencialmente **Linux Ubuntu 20.04 LTS ou 22.04 LTS**) precisa dos seguintes componentes instalados:

1.  **Java 17 (OpenJDK 17)**: O ecossistema foi totalmente homologado e sintonizado sob a JVM do Java 17. Versões anteriores ou mais recentes podem apresentar incompatibilidades de bytecode.
2.  **SBT (Scala Build Tool)**: Gerenciador de compilação oficial do Scala, necessário para executar e gerenciar os projetos do Nó e do Matcher DEX.
3.  **Utilitários de Sistema**: `git` (para controle de versão) e `bc` (calculadora de terminal, usada para converter valores monetários em Satoshis).

### 🛠️ Comando Único para Instalação de Dependências (Ubuntu/Debian):
Copie, cole e execute o bloco abaixo no terminal do servidor para instalar tudo de uma vez:
```bash
# Atualizar repositórios do sistema e instalar dependências essenciais
sudo apt-get update
sudo apt-get install openjdk-17-jdk git bc curl gnupg -y

# Adicionar repositório e credenciais oficiais do SBT
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/sbt-signatures.gpg
sudo apt-get update
sudo apt-get install sbt -y
```

---

## 🔌 2. Portas de Rede Necessárias (Devem estar liberadas)

Os serviços utilizam as seguintes portas para comunicação de APIs, troca de blocos e liquidez de ordens. Certifique-se de liberar estas portas em seu firewall (ou grupos de segurança da AWS, Azure, Google Cloud, etc.):

*   **`6869`** (HTTP REST): Porta da API pública do Nó AMZX (consultas de saldos, envio de transações de ativos, criação de carteiras).
*   **`6868`** (TCP P2P): Porta de comunicação ponto a ponto entre nós da rede blockchain (comunicação de sincronização de rede).
*   **`6887`** (gRPC DEX Extension): Canal gRPC integrado no Nó para consulta rápida de saldos pela DEX.
*   **`6881`** (gRPC Blockchain Updates): Canal gRPC para streaming de blocos novos em tempo real do Nó para a DEX.
*   **`6886`** (HTTP REST): Porta da API pública do Matcher DEX (livro de ofertas, ordens de compra/venda, histórico de negociações).

---

## 🧙‍♂️ 3. Guia Detalhado de Preenchimento de Dados (Wizard)

Ao executar o script assistente interativo (`amz-network-wizard/init-network.sh`), o operador será questionado sobre vários parâmetros fundamentais para a criação da rede privada. 

Abaixo está a explicação detalhada de cada campo de preenchimento, o comportamento esperado e as regras recomendadas:

### 1️⃣ Network Character (Chain ID)
*   **O que é:** O caractere identificador da sua rede blockchain privada. Ele é embutido criptograficamente em cada endereço de carteira gerado na rede para evitar que transações de uma rede (ex: Mainnet ou Testnet) sejam executadas acidentalmente em outra.
*   **Regra de preenchimento:** Deve ser **uma única letra** (alfabética). O assistente automaticamente converterá letras minúsculas para maiúsculas.
*   **Sugestão Padrão:** `D` (pressione Enter para aceitar).

### 2️⃣ Native Coin Name
*   **O que é:** O nome visível do ativo de saldo nativo (moeda padrão) do sistema, utilizado para pagar taxas de transação e transacionar valores.
*   **Regra de preenchimento:** Qualquer string alfanumérica de identificação (geralmente em maiúsculo).
*   **Sugestão Padrão:** `AMZX` (pressione Enter para aceitar).

### 3️⃣ Total Genesis Supply (Moedas Inteiras)
*   **O que é:** A quantidade total de moedas que existirão no momento da criação da blockchain (no Bloco Gênese). Essa quantidade inicial é cunhada e distribuída na carteira do Administrador Principal.
*   **Regra de preenchimento:** Insira um valor numérico inteiro (ex: `100000000` para cem milhões de moedas). 
*   **Nota Técnica:** A blockchain processa valores internamente com 8 casas decimais de precisão (Satoshis). O script assistente já faz a conversão do valor inserido multiplicando-o automaticamente por $10^8$ para poupar o operador de digitar zeros adicionais.
*   **Sugestão Padrão:** `100000000` (pressione Enter para aceitar).

### 4️⃣ Admin Genesis Seed Phrase
*   **O que é:** A frase mnemônica (semente de recuperação) que serve como raiz de segurança da sua blockchain privada. A partir desta frase, o sistema deriva criptograficamente a Chave Privada, Chave Pública e o Endereço de Administrador que receberá todas as moedas criadas no Genesis Supply.
*   **Regra de preenchimento:** Uma frase longa de palavras em inglês separadas por espaço (ex: um conjunto de 12 a 24 palavras seguras ou uma frase longa descritiva).
*   **Importante:** Guarde esta frase com segurança máxima. Quem tiver acesso a ela terá o controle total dos fundos iniciais e do poder de mineração inicial da rede.
*   **Sugestão Padrão:** `amzblockchain test private local network genesis seed wordlist` (pressione Enter para aceitar para testes locais).

### 5️⃣ Portas de Serviços (Node API, P2P, gRPC, Matcher API)
*   **O que é:** O assistente solicitará a confirmação de cada porta de comunicação listada na **Seção 2** deste guia.
*   **Regra de preenchimento:** Valores numéricos inteiros válidos de portas TCP (entre `1024` e `65535`).
*   **Sugestão Padrão:** Pressione **Enter** em todas elas para aceitar o padrão recomendado (`6869`, `6868`, `6887`, `6881`, `6886`).

### 6️⃣ Run Directory (Nome da Pasta de Execução)
*   **O que é:** O nome da pasta física onde o assistente irá gravar os arquivos de configuração gerados (`blockchain.conf` e `matcher.conf`), as chaves criptográficas da rede e os scripts executáveis rápidos.
*   **Regra de preenchimento:** Nome de pasta válido (sem espaços ou caracteres especiais).
*   **Sugestão Padrão:** `run-amzx-D` (onde `D` é o Chain ID inserido). Pressione Enter para aceitar.

---

## 🚀 4. Executando o Sistema

Com os dados preenchidos e a rede provisionada, a inicialização é feita de forma simples e coordenada em terminais separados.

### 🏁 Passo 1: Provisionar a Rede Criptográfica (Primeira vez)
Execute o assistente e siga as instruções na tela preenchendo ou aceitando as sugestões de dados padrão:
```bash
cd amz-network-wizard
./init-network.sh
```

### 🔷 Passo 2: Iniciar o Nó Blockchain
Abra um terminal dedicado (Terminal 1) e execute o script gerado para colocar a rede blockchain para rodar e minerar:
```bash
cd amz-network-wizard/run-amzx-D
./start-node.sh
```
*O terminal começará a exibir a geração de blocos em tempo real.*

### 📈 Passo 3: Iniciar o Matcher DEX
Abra um segundo terminal dedicado (Terminal 2) e execute o script de inicialização do motor de ofertas:
```bash
cd amz-network-wizard/run-amzx-D
./start-matcher.sh
```
*A DEX conectará ao seu nó via gRPC na porta de controle rápido e estabelecerá o estado `"Working"`.*

---

## 🔍 5. Validação das APIs e Swagger Interativo

Uma vez inicializados os serviços, você pode testar, validar e interagir diretamente por meio das especificações OpenAPI Swagger integradas:

1.  **API do Nó Blockchain:** Acesse [http://localhost:6869/api-docs/index.html](http://localhost:6869/api-docs/index.html) para realizar consultas de blocos, saldos de carteiras e gerenciar chaves.
2.  **API do Matcher DEX:** Acesse [http://localhost:6886/api-docs/index.html](http://localhost:6886/api-docs/index.html) para interagir com o Livro de Ofertas e executar negociações de ativos digitais.
3.  **Verificação de Saúde da DEX:** Acesse [http://localhost:6886/matcher](http://localhost:6886/matcher) no seu navegador para obter o status em JSON do motor. O retorno esperado deve ser:
    ```json
    {
      "status": "Working"
    }
    ```
