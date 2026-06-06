# 🛡️ Manual Operacional: Arquitetura de Subdomínios, Nginx Reverse Proxy e SSL Certbot

Este documento detalha a arquitetura de exposição segura para ambientes de produção da blockchain **AMZX** e do motor de liquidez **Matcher DEX**. 

Em redes de produção, **nunca** expomos as portas locais de microsserviços diretamente para a internet. Em vez disso, utilizamos o **Nginx** como um servidor de proxy reverso e balanceador de carga, encriptando todo o tráfego com certificados digitais SSL/TLS obtidos gratuitamente via **Certbot (Let's Encrypt)**.

---

## 🌐 1. Arquitetura de Exposição de Rede e Subdomínios

Para prover uma infraestrutura limpa, amigável e segura, mapeamos as portas internas do sistema para subdomínios específicos operando sobre as portas padrão da Web: **`80` (HTTP)** e **`443` (HTTPS / GRPCS / WSS)**.

| Subdomínio Recomendado | Porta Original | Protocolo Interno | Protocolo Externo (SSL) | Serviço Exposto |
| :--- | :--- | :--- | :--- | :--- |
| **`nodes.seu-dominio.com`** | `6869` | HTTP | `HTTPS (443)` | Swagger UI e API RESTful do Nó Blockchain |
| **`matcher.seu-dominio.com`** | `6886` | HTTP | `HTTPS (443)` | Swagger UI e API RESTful da DEX Matcher |
| **`rpc.seu-dominio.com`** | `6869/eth` | HTTP | `HTTPS (443)` | RPC compatível com Ethereum / MetaMask |
| **`grpc-dex.seu-dominio.com`**| `6887` | gRPC (HTTP/2) | `GRPCS / WSS (443)`| Integração gRPC de Controle Nó-DEX |
| **`grpc-updates.seu-dominio.com`**| `6881` | gRPC (HTTP/2) | `GRPCS / WSS (443)`| Stream de Blocos em Tempo Real para a DEX |
| **`peer.seu-dominio.com`** | `6868` | TCP Customizado | `TCP Direto (6868)` | Protocolo de Consenso P2P entre Nós |

---

## 📖 2. Descrição Detalhada de Cada Subdomínio

---

### 🔷 A. `nodes.seu-dominio.com`
*   **O que faz:** Expõe de forma segura a API pública do nó blockchain e a sua documentação interativa Swagger.
*   **Por que usar:** Em vez de carteiras de usuários, portais web e ferramentas externas consultarem as APIs do nó na porta desprotegida `6869` (HTTP), elas consultam via protocolo criptografado `HTTPS` (porta `443`). Isso previne ataques de interceptação (Man-in-the-Middle) de senhas, chaves públicas e transações.
*   **Quando usar:** 
    *   Sempre que precisar consultar o saldo de uma carteira de forma segura.
    *   Ao assinar e transmitir uma nova transação criptografada para a rede.
    *   Para desenvolvedores consultarem a documentação interativa Swagger para estudar as APIs.
*   **Como usar:**
    *   Swagger UI (Interface Web): `https://nodes.seu-dominio.com/api-docs/index.html`
    *   Consulta de saldo (Exemplo HTTP GET): 
        ```bash
        curl -X GET "https://nodes.seu-dominio.com/addresses/balance/3FbABsgsehxLq43KvFxeTejdwS4uGnoaFGE"
        ```

---

### 📈 B. `matcher.seu-dominio.com`
*   **O que faz:** Expõe o livro de ofertas (Orderbook) e o painel de liquidez da Matcher DEX.
*   **Por que usar:** O Matcher processa ordens de mercado em milissegundos e lida diretamente com intenções de trading de ativos dos usuários. O uso de SSL garante que o envio de ordens de limite e cancelamento via assinatura de chave Curve25519 trafegue de forma blindada na rede.
*   **Quando usar:**
    *   Pelo painel da sua corretora descentralizada (DEX Front-end) para popular os gráficos de preços e os livros de compra e venda.
    *   Sempre que um usuário submeter uma ordem de compra/venda de ativos na plataforma.
*   **Como usar:**
    *   Estudar rotas de trading (Swagger): `https://matcher.seu-dominio.com/api-docs/index.html`
    *   Verificar a saúde da DEX: `https://matcher.seu-dominio.com/matcher` -> Retorno esperado: `{"status":"Working"}`.

---

### 🦊 C. `rpc.seu-dominio.com`
*   **O que faz:** Provê um endpoint JSON-RPC 100% compatível com a especificação do Ethereum.
*   **Por que usar:** O nó AMZX possui suporte nativo a transações da EVM. O Nginx faz um redirecionamento inteligente e transparente: qualquer requisição enviada para a raiz do subdomínio (`https://rpc.seu-dominio.com/`) é repassada internamente para o caminho especializado `/eth` do nó (ex: `http://127.0.0.1:6869/eth/`). Isso permite que os usuários usem carteiras tradicionais de mercado (como a **MetaMask**) apontando diretamente para o seu subdomínio sem precisar digitar caminhos adicionais.
*   **Quando usar:**
    *   Sempre que integrar dApps desenvolvidos para a Ethereum, BSC, Polygon ou Avalanche na sua blockchain.
    *   Ao configurar a rede privada de blockchain dentro da carteira MetaMask do usuário final.
*   **Como usar:**
    *   Na MetaMask, clique em "Adicionar Rede Customizada" e preencha:
        *   **Nome da Rede:** AMZX Private Network
        *   **Nova URL do RPC:** `https://rpc.seu-dominio.com`
        *   **ID da Cadeia (Chain ID):** O número correspondente ao seu caractere configurado.
        *   **Símbolo de Moeda Nativa:** `AMZX`

---

### 🤝 D. `peer.seu-dominio.com`
*   **O que faz:** Identifica o endereço de comunicação de rede ponto a ponto (P2P) entre os nós mineradores e validadores.
*   **Por que usar:** Blockchain utiliza um protocolo binário otimizado e criptografado nativamente para os nós trocarem blocos e assinaturas de transação na velocidade máxima da rede. Como esse tráfego não é HTTP, ele trafega diretamente sobre a porta `6868`.
*   **Quando usar:**
    *   Ao adicionar novos nós mineradores de parceiros ou servidores secundários na sua rede para descentralizar o consenso.
*   **Como usar:**
    *   No arquivo de configuração de um novo nó adicionado à rede (`blockchain.conf`), define-se o host do nó original em `known-peers`:
        ```hocon
        waves.network.known-peers = [ "peer.seu-dominio.com:6868" ]
        ```
    *   *Nota de Firewall:* O Nginx não encripta o protocolo binário de consenso via SSL convencional. Por isso, a porta TCP `6868` deve ser aberta diretamente no grupo de segurança/firewall do seu servidor de nuvem para tráfego bidirecional.

---

### ⚡ E. `grpc-dex.seu-dominio.com`
*   **O que faz:** Canal de alta performance de integração gRPC que conecta o motor do Matcher DEX ao Nó Blockchain.
*   **Por que usar:** O motor de correspondência de ordens precisa verificar o saldo de carteiras em milissegundos para bloquear fundos de ofertas abertas. gRPC utiliza HTTP/2 de forma bidirecional, provendo velocidades até 10x superiores ao HTTP REST tradicional. Nginx age como um túnel seguro de HTTP/2 encriptado (`GRPCS` na porta `443`), direcionando o tráfego para a porta gRPC interna `6887`.
*   **Quando usar:**
    *   Na comunicação entre servidores dedicados rodando o Nó Blockchain em uma máquina e a Matcher DEX em outra.
*   **Como usar:**
    *   No arquivo de configuração da DEX (`matcher.conf`), preencha a URL de integração apontando para o seu subdomínio gRPC seguro na porta `443`:
        ```hocon
        # Integração gRPC criptografada com o Nó
        matcher.grpc.integration.host = "grpc-dex.seu-dominio.com"
        matcher.grpc.integration.port = 443
        ```

---

### 📡 F. `grpc-updates.seu-dominio.com`
*   **O que faz:** Canal de transmissão contínua (stream) de atualização de novos blocos minerados.
*   **Por que usar:** Permite que o Matcher DEX escute em tempo real cada novo bloco que é validado pela rede, atualizando os saldos bloqueados por ordens liquidadas instantaneamente.
*   **Quando usar:**
    *   Para alimentar robôs de trading de alta frequência com eventos de blockchain em tempo real.
    *   Para indexadores de blocos de alto desempenho e exploradores de blocos baseados em gRPC.
*   **Como usar:**
    *   Conexão via cliente gRPC seguro apontando para `grpc-updates.seu-dominio.com:443`.

---

## 🛠️ 3. Funcionamento Interno da Configuração do Nginx

O arquivo de configuração gerado pelo assistente de rede (`nginx-amzx.conf`) utiliza as diretivas mais modernas do Nginx para segmentar os subdomínios dentro de um único IP público na porta padrão:

1.  **Compatibilidade HTTP/1.1 e Websockets:**
    O Nginx repassa os cabeçalhos de `Upgrade` e `Connection`, garantindo suporte nativo a fluxos de dados em tempo real e conexões persistentes necessárias para as conexões das carteiras dos clientes.
2.  **Roteamento Inteligente do Ethereum RPC:**
    O bloco de servidor `rpc.seu-dominio.com` redireciona o tráfego de `/` diretamente para a porta interna `/eth/`, criando um ponto de entrada limpo e elegante compatível com qualquer ferramenta de ecossistema EVM.
3.  **Habilitação Nativa de gRPC (HTTP/2):**
    Utilizando a diretiva `grpc_pass`, o Nginx traduz e repassa as chamadas de API RPC binárias HTTP/2 eficientemente sem a necessidade de expor portas gRPC desprotegidas no firewall.

---

## 🔑 4. Obtenção e Renovação de Certificados SSL (Certbot)

O script assistente gerado (`setup-nginx-ssl.sh`) automatiza a aquisição de certificados e a modificação das regras do Nginx de forma 100% segura usando o **Certbot**:

### Como o script realiza a configuração sob o capô:
1.  Verifica se o **Nginx** e o **Certbot** estão instalados na máquina host; caso contrário, realiza o download e instalação automáticos.
2.  Copia o arquivo de configuração `nginx-amzx.conf` para `/etc/nginx/sites-available/nginx-amzx-[ChainID].conf`.
3.  Cria o link simbólico na pasta `/etc/nginx/sites-enabled/` para ativar a configuração.
4.  Executa um teste de integridade sintática via `nginx -t` para garantir que o servidor não sofrerá interrupções.
5.  Recarrega o Nginx para carregar os novos subdomínios na porta `80`.
6.  Dispara a solicitação para a autoridade certificadora **Let's Encrypt** para obter certificados SSL válidos de forma gratuita para os 5 subdomínios simultaneamente.
7.  A Let's Encrypt valida o apontamento DNS (verificando que os subdomínios realmente apontam para o IP do seu servidor).
8.  O Certbot reescreve automaticamente o arquivo do Nginx, adicionando as chaves SSL e configurando o redirecionamento automático de `HTTP` para `HTTPS` seguro!

### ⏳ Renovação Automática:
Os certificados da Let's Encrypt são válidos por **90 dias**. O pacote `python3-certbot-nginx` instala automaticamente uma rotina em segundo plano (cronjob/systemd timer) que monitora e renova os certificados de forma 100% automatizada antes do vencimento, garantindo risco zero de expiração da segurança do cliente.
