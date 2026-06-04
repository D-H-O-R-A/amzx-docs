# 🧪 Ambientes de Teste, Simulação e Desenvolvimento AMZX

Este documento fornece as instruções de engenharia para criar, configurar e operar ambientes de desenvolvimento rápidos, redes privadas locais (Private Testnets), execuções de testes automatizados e simulação offline de smart contracts RIDE no ecossistema **AMZX** (better2better.com.br).

---

## 📦 1. Operação com Docker e Docker Compose

O uso de containers Docker permite instanciar nós validadores da rede AMZX ou redes privadas isoladas de forma limpa, persistente e parametrizada.

### A. Rede Privada de Alta Velocidade (Private Node)
Útil para o desenvolvimento rápido de dApps e contratos inteligentes localmente. É configurada para gerar blocos a cada **10 segundos**, possui todas as 25 Consensus Protocol Features pré-ativadas e utiliza o ID de rede `'R'`.

#### Comando de Inicialização Rápida:
```bash
docker run -d \
  --name amzx-private-node \
  -p 6869:6869 \
  -p 6862:6862 \
  wavesplatform/waves-private-node
```

#### Conta Mineradora Rica de Desenvolvimento (Genesis Rich Account):
A rede privada é criada com uma conta mineradora padrão contendo todos os tokens AMZX gerados no bloco gênesis. Você pode usar a semente secreta abaixo para assinar transações e distribuir fundos:

*   **Texto de Semente (Seed Phrase)**: `waves private node seed with waves tokens`
*   **Semente Codificada (Base58)**: `TBXHUUcVx2n3Rgszpu5MCybRaR86JGmqCWp7XKh7czU57ox5dgjdX4K4`
*   **Chave Privada da Conta**: `83M4HnCQxrDMzUQqwmxfTVJPTE9WdE7zjAooZZm2jCyV`
*   **Chave Pública da Conta**: `AXbaBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k1m2bNCzXV`
*   **Endereço da Conta**: `3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF`
*   **API Key Padrão do Nó**: `waves-private-node`

---

### B. Persistência de Dados e Customização com Docker Volume
Para garantir que o estado da blockchain e as carteiras geradas persistam entre reinicializações de containers, mapeie diretórios locais da máquina hospedeira para dentro do container:

```bash
# 1. Criar os diretórios locais no host
mkdir -p /docker/amzx/data
mkdir -p /docker/amzx/config

# 2. Executar o nó acoplando os volumes e passando parâmetros de JVM
docker run -d \
  --name amzx-node-docker \
  -v /docker/amzx/data:/var/lib/waves \
  -v /docker/amzx/config:/etc/waves \
  -p 6869:6869 \
  -p 6862:6862 \
  -e WAVES_NETWORK=stagenet \
  -e WAVES_WALLET_PASSWORD="MinhaSenhaSuperSeguraDaDEX123" \
  -e WAVES_HEAP_SIZE="4g" \
  wavesplatform/wavesnode
```

#### Variáveis de Ambiente Suportadas:

| Variável | Valor Padrão | Descrição |
| :--- | :---: | :--- |
| `WAVES_NETWORK` | `mainnet` | Rede de conexão ativa. Opções: `mainnet`, `testnet`, `stagenet`. |
| `WAVES_WALLET_SEED` | *(Nulo)* | Semente geradora da carteira padrão codificada em Base58. |
| `WAVES_WALLET_PASSWORD` | *(Nulo)*| Senha de criptografia do arquivo de carteiras (`wallet.dat`). |
| `WAVES_LOG_LEVEL` | `INFO` | Nível de verbosidade de log no console: `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`. |
| `WAVES_HEAP_SIZE` | `2g` | Limite de memória Heap máxima atribuída à JVM (formato `-Xmx`). |
| `JAVA_OPTS` | *(Nulo)* | Argumentos adicionais de inicialização da JVM Java (ex: `-Dwaves.rest-api.enable=yes`). |

---

## ⚡ 2. Inicialização Rápida com Sincronização Acelerada (Bootstrap)

Se você estiver iniciando um nó completo para se conectar a uma rede pública existente (`mainnet` ou `testnet`), o download bloco-a-bloco a partir da rede peer-to-peer pode levar horas ou dias, pois cada bloco é validado individualmente.

Para acelerar esse processo, você pode baixar o instantâneo compactado do estado do ledger (Bootstrap Tarball) diretamente dos servidores oficiais, extraí-lo no diretório de dados e iniciar o nó. O nó detectará o banco de dados RocksDB populado e iniciará a sincronização apenas a partir da última altura salva, pulando a validação pesada dos blocos passados.

```bash
# 1. Preparar o diretório de dados
mkdir -p /docker/amzx/data

# 2. Baixar e extrair o arquivo compactado de estado em tempo real (exemplo para Testnet)
wget -qO- http://blockchain-testnet.wavesnodes.com/blockchain_last.tar --show-progress | tar -xvf - -C /docker/amzx/data

# 3. Iniciar o nó acoplado ao diretório populado
docker run -d \
  -v /docker/amzx/data:/var/lib/waves \
  -p 6869:6869 \
  -e WAVES_NETWORK=testnet \
  -e WAVES_WALLET_PASSWORD="SuaSenhaDeDesenvolvimento" \
  wavesplatform/wavesnode
```

---

## 🛠️ 3. Ferramentas de Build e Automação SBT (Scala Build Tool)

Tanto o Nó quanto o Matcher DEX utilizam a ferramenta **SBT** para compilação, resolução de dependências Scala/Java e empacotamento.

### Comandos de Compilação e Teste:

```bash
# Compilar todo o projeto localmente
sbt compile

# Executar todas as verificações de formatação estática e integridade do PR
sbt checkPR

# Executar os testes unitários da blockchain
sbt node/test

# Criar a imagem Docker a partir do código-fonte local atualizado
sbt node-it/docker
```

### Geração de Pacotes de Produção:
Os pacotes gerados contêm o binário empacotado da aplicação e arquivos de serviço para Linux (`.deb`) na pasta `target/`:

```bash
# Gerar instaladores para a Mainnet (padrão)
sbt packageAll

# Gerar instaladores adaptados para a Testnet
sbt -Dnetwork=testnet packageAll
```

### Execução de Testes de Integração em Lote (Integration Tests):
Os testes de integração simulam cenários multi-nós reais usando Docker de forma programática.

```bash
# Executar todos os testes de integração sequencialmente
sbt -Dwaves.it.max-parallel-suites=1 node-it/test

# Executar uma classe de teste de integração específica
sbt "node-it/testOnly com.wavesplatform.it.sync.transactions.MassTransferTransactionSuite"
```

---

## 🏃 4. Simulação de Contratos RIDE Offline (Ride Runner)

O componente **AMZX Ride Runner** (`ride-runner`) é um microsserviço de utilidade que permite compilar, executar e testar contratos inteligentes programados na linguagem funcional RIDE de forma completamente **offline**, sem a necessidade de instanciar ou sincronizar um nó de rede completo.

O Ride Runner opera de duas formas:
1.  **Como Serviço REST**: Emula o endpoint `/utils/script/evaluate` fornecido pela API REST do nó.
2.  **Como Aplicação de CLI**: Executa um arquivo RIDE contra um estado de blockchain simulado descrito em um arquivo HOCON.

### Instalação e Execução via Docker Compose:
Você pode adicionar o Ride Runner à sua malha local de microserviços adicionando a seguinte definição ao seu arquivo `docker-compose.yml`:

```yaml
version: '3.8'
services:
  amzx-ride-runner:
    image: wavesplatform/ride-runner:latest
    restart: "unless-stopped"
    ports:
      - "6890:6890"  # Porta do serviço REST de avaliação
      - "9095:9095"  # Porta do exportador de métricas Prometheus
    environment:
      - RIDE_LOG_LEVEL=INFO
      - RIDE_HEAP_SIZE=1g
    volumes:
      - ./ride-runner/data:/var/lib/waves-ride-runner
      - ./ride-runner/config/local.conf:/etc/waves-ride-runner/local.conf:ro
```

### A. Avaliação Programática via API REST (Porta 6890)
Envie uma requisição HTTP POST para simular uma transação invocando o método de um smart contract e veja o resultado de mudança de estado calculado na hora:

```bash
curl -X POST http://localhost:6890/utils/script/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "script": "base64:AAIDAAAAAAAAAAQIABAAAAACAAAAA...",
    "expression": "evalCustomCall()",
    "address": "3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF"
  }'
```

---

### B. Execução de Script RIDE por CLI com Estado Preparado
Você pode definir um cenário completo de testes especificando saldos fictícios, transações passadas, variáveis de dados e oráculos em um arquivo de configuração `.conf` (formato HOCON) e passando como argumento para o JAR do Ride Runner.

```bash
# Executar a simulação offline
java -jar waves-ride-runner-all-1.4.13.jar ./meu_cenario_de_teste.conf
```

#### Exemplo de Arquivo de Estado Simulado (`meu_cenario_de_teste.conf`):
```hocon
# Configuração do estado da blockchain para o teste
blockchain {
  height = 1502300
  chain-id = 82 # 'R' (Decimal)
}

# Definição de contas e seus saldos fictícios
accounts {
  # Endereço do dApp
  "3M4qwDomRabJKLZxuXhwfqLApQkU592nWxF" {
    balance = 100000000000 # 1000 AMZX em satoshis
    data {
      "limite_diario" = 500
      "proprietario" = "3M5aBkJNocyrVpwqTzD4TpUY8fQ6eeRto9k"
    }
  }
}

# Código do script RIDE que será avaliado
script = """
  {-# CONTENT_TYPE DAPP #-}
  {-# SCRIPT_TYPE ACCOUNT #-}
  {-# IMPORT compat3 #-}

  @Callable(i)
  func transferir(valor: Int) = {
    let limite = getStringValue(this, "limite_diario")
    if (valor > limite) then throw("Limite diário excedido!")
    else [ScriptTransfer(i.caller, valor, unit)]
  }
"""

# Chamada a ser avaliada
expression = "transferir(600)"
```

### Limitações Conhecidas do Ride Runner:
Embora o Ride Runner seja extremamente rápido para testes de lógica matemática de contratos, ele possui as seguintes limitações em relação à execução nativa on-chain dentro do Nó:
1.  **Scripts de Ativos (Asset Scripts)**: Scripts associados diretamente a tokens (Smart Assets) para validar transações de envio não são suportados nativamente via chamada REST `/utils/script/evaluate`.
2.  **Funções RIDE Bloqueadas**: Scripts RIDE que tentarem invocar métodos que exijam acesso estrito à memória histórica persistente do RocksDB falharão offline. As funções bloqueadas são:
    *   `isDataStorageUntouched(address/alias)`
    *   `transferTransactionById(transactionId)`
