# 🤖 Chatbot de Engenharia de Prompt — IA Generativa Open Source na Oracle Cloud

> Como coloquei um chatbot de IA generativa no ar, acessível por qualquer navegador, usando um modelo open source da NVIDIA, Python + Streamlit e uma máquina virtual gratuita na Oracle Cloud.

![Python](https://img.shields.io/badge/Python-3.10+-blue) ![Streamlit](https://img.shields.io/badge/Streamlit-app-red) ![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-Always%20Free-orange) ![Modelo](https://img.shields.io/badge/LLM-Llama%203.3%2070B-green)

🔗 **Aplicação em produção:** http://146.235.57.116:8501

---

## Introdução

Modelos de linguagem de grande porte (LLMs) deixaram de ser uma curiosidade de laboratório e viraram ferramenta de trabalho. Mas uma coisa é conversar com um modelo dentro de um playground; outra, bem diferente, é **colocar uma aplicação de IA no ar**, acessível publicamente, rodando de forma estável em um servidor real.

Foi exatamente esse o desafio deste projeto: desenvolver um chatbot baseado em IA generativa e **implantá-lo em produção** em uma máquina virtual na nuvem, deixando-o acessível a qualquer pessoa através de um endereço IP público e um navegador.

### Objetivo da atividade

Desenvolver e implantar um chatbot de IA generativa utilizando um **modelo open source disponibilizado pela NVIDIA**, construído em **Python com Streamlit** e publicado em uma **máquina virtual na Oracle Cloud**, de modo que o chatbot fique acessível via IP público e permita a interação de usuários pelo navegador.

### Visão geral da solução

A solução é um assistente conversacional especializado em **Engenharia de Prompt**. O usuário acessa uma interface web limpa, digita perguntas (sobre LLMs, RAG, agentes, avaliação de prompts etc.) e recebe respostas geradas por um modelo Llama 3.3 de 70 bilhões de parâmetros.

O ponto-chave da arquitetura é que **o modelo não roda dentro da VM**. A inferência acontece nos servidores da NVIDIA (via API compatível com OpenAI), e a máquina virtual fica responsável apenas por servir a interface Streamlit. Isso permite usar um modelo gigante de 70B parâmetros mesmo em uma VM gratuita com apenas 1 GB de RAM.

---

## Infraestrutura

### Configuração da máquina virtual

A aplicação foi publicada em uma instância **Oracle Cloud Infrastructure (OCI)** dentro do plano **Always Free** (gratuito).

| Item | Configuração |
|------|--------------|
| Instância | `instance-20260313-2202` (Always Free) |
| Shape | `VM.Standard.E2.1.Micro` |
| IP público | `146.235.57.116` |
| IP privado | `10.0.0.234` |
| Porta da aplicação | `8501` (Streamlit) |

### Sistema operacional utilizado

**Canonical Ubuntu 22.04 LTS**, com acesso via SSH pelo usuário padrão `ubuntu`.

### Recursos computacionais disponíveis

- **1 OCPU** (processador compartilhado, arquitetura x86 AMD EPYC)
- **1 GB de memória RAM**
- **Armazenamento em block storage**
- **Largura de banda de rede:** ~0,48 Gbps

Esses recursos são propositalmente modestos. Eles seriam insuficientes para rodar um LLM localmente, mas são mais que suficientes para hospedar a interface Streamlit, já que a inferência é delegada à API da NVIDIA.

---

## Modelo Escolhido

### Nome do modelo

**Meta Llama 3.3 70B Instruct** (`meta/llama-3.3-70b-instruct`), consumido através do catálogo de modelos da NVIDIA (`https://integrate.api.nvidia.com/v1`).

### Justificativa da escolha

- **Open source:** a família Llama da Meta é aberta e amplamente adotada, atendendo ao requisito de usar um modelo open source.
- **Qualidade de ponta:** o Llama 3.3 70B entrega qualidade comparável a modelos muito maiores, com ótimo desempenho em raciocínio, instruções e múltiplos idiomas — incluindo português.
- **Custo de infraestrutura zero na VM:** ao usar a API da NVIDIA, eliminamos a necessidade de GPU. Um modelo de 70B exigiria dezenas de GB de VRAM; pela API, ele roda em uma VM de 1 GB de RAM.
- **Compatibilidade com o padrão OpenAI:** a API segue o formato OpenAI, o que simplifica o código e permite trocar de modelo facilmente no futuro.

### Principais características

- 70 bilhões de parâmetros, otimizado para seguir instruções (*instruct*).
- Suporte multilíngue, com bom desempenho em português.
- Janela de contexto ampla, adequada para manter o histórico da conversa.
- Servido com aceleração em GPUs NVIDIA, garantindo baixa latência.

---

## Desenvolvimento

### Arquitetura da aplicação

```
┌──────────────┐      HTTP :8501      ┌───────────────────────────┐      HTTPS      ┌────────────────────┐
│   Navegador  │ ───────────────────▶ │   VM Oracle (Ubuntu 22.04) │ ──────────────▶ │  API NVIDIA        │
│  do usuário  │ ◀─────────────────── │   Streamlit + chatlas      │ ◀────────────── │  Llama 3.3 70B     │
└──────────────┘      resposta        └───────────────────────────┘   inferência    └────────────────────┘
```

O fluxo é simples: o navegador conversa com o Streamlit na porta 8501; o Streamlit monta o contexto (system prompt + histórico) e chama a API da NVIDIA via biblioteca `chatlas`; a resposta volta e é renderizada na tela. O histórico da conversa é mantido em `st.session_state`.

### Bibliotecas utilizadas

- **Streamlit** — interface web e componentes de chat (`st.chat_message`, `st.chat_input`).
- **chatlas** — cliente de alto nível para conversar com LLMs (`ChatOpenAICompletions`).
- **openai** — SDK base usado pelo chatlas para falar com endpoints compatíveis com OpenAI.
- **python-dotenv** — carrega as credenciais do arquivo `.env` para variáveis de ambiente.

### Estratégia de gerenciamento de credenciais

A chave de API nunca é escrita no código. Ela vive em um arquivo **`.env`** local, carregado em tempo de execução por `load_dotenv()` e lido com `os.getenv("NVIDIA_API_KEY")`.

Esse `.env` é explicitamente **ignorado pelo Git** através do `.gitignore`, junto com chaves SSH e qualquer arquivo sensível. O `.env` nunca é versionado: ele é criado manualmente na máquina local e, separadamente, no servidor. Assim, o segredo continua fora do GitHub, mas a aplicação continua funcionando porque o `.env` real permanece em cada ambiente.

---

## Implantação

### Processo de publicação na Oracle Cloud

1. Criação da instância Always Free com Ubuntu 22.04 e geração do par de chaves SSH.
2. Liberação da porta `8501` na **Security List** da rede virtual (VCN) da Oracle.
3. Liberação da mesma porta no firewall do sistema operacional (`iptables`).
4. Acesso à VM via SSH e instalação de Python, `pip` e `venv`.
5. Clonagem do repositório e criação de um ambiente virtual isolado.
6. Instalação das dependências do `requirements.txt` e criação do `.env` diretamente no servidor.
7. Execução do Streamlit em `0.0.0.0:8501`, mantido ativo em segundo plano com `systemd`.

> O passo a passo de produção, comando por comando, está na seção [**Como colocar em produção (Oracle Cloud)**](#como-colocar-em-produção-oracle-cloud) mais abaixo.

### Principais desafios encontrados

- **Camada dupla de firewall:** a Oracle bloqueia portas em *dois* lugares — na Security List da nuvem **e** no `iptables` da própria imagem Ubuntu. Liberar só um não basta; foi preciso abrir a porta nos dois.
- **Memória limitada (1 GB):** exigiu disciplina para manter a VM enxuta. Delegar a inferência à API da NVIDIA foi a decisão de arquitetura que tornou o projeto viável nesse hardware.
- **Acesso externo do Streamlit:** por padrão o Streamlit escuta apenas em `localhost`; foi necessário configurá-lo para escutar em `0.0.0.0` para aceitar conexões externas.
- **Manter o app no ar:** rodar via terminal encerra a aplicação ao fechar a sessão SSH. A solução foi transformá-lo em serviço `systemd`, que reinicia sozinho.
- **Estratégia de Deploy:** É possível enviar os arquivos de um computador para o servidor na nuvem via terminal, porém é mais complicado e difícil de reproduzir. Utilizar o github como repositório para copiar o necessário e futuras melhorias é uma estratégia mais eficiente e otimizada

---

## Discussão

### Lições aprendidas

- **A infraestrutura é metade do trabalho.** Escrever o chatbot foi rápido; deixá-lo no ar de forma estável e acessível foi onde estava o aprendizado real — redes, firewall, processos em segundo plano.
- **Arquitetura inteligente vence hardware caro.** Separar interface (na VM) de inferência (na API) permitiu usar um modelo de 70B em uma máquina gratuita de 1 GB.
- **Segurança de credenciais é um hábito.** Um `.gitignore` bem feito que bloqueia `.env` e chaves é um padrão simples que evita o erro clássico de vazar credenciais no GitHub.

### Possíveis melhorias futuras

- **HTTPS e domínio próprio:** colocar um proxy reverso (Nginx) com certificado Let's Encrypt e um domínio amigável no lugar do IP cru.
- **Respostas em streaming:** exibir o texto token a token (`stream=True`) para uma experiência mais fluida.
- **Containerização:** empacotar tudo em Docker para deploys reproduzíveis.
- **Autenticação e limites de uso:** proteger o acesso e controlar o consumo da API.
- **Memória/contexto inteligente:** resumir conversas longas para otimizar o uso de tokens.

---

## Como executar

### Rodar localmente

```bash
# 1. Clone o repositório
git clone https://github.com/SEU_USUARIO/chatbot-engenharia-prompt.git
cd chatbot-engenharia-prompt

# 2. Crie e ative o ambiente virtual
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Instale as dependências
pip install -r requirements.txt

# 4. Crie o arquivo .env com a sua chave (criado do zero)
echo "NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI" > .env   # Windows (PowerShell): "NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI" | Out-File .env -Encoding utf8

# 5. Rode a aplicação
streamlit run app.py
```

Acesse `http://localhost:8501`.

### Como colocar em produção (Oracle Cloud)

A implantação é parecida com a execução local, mas acontece **dentro da VM** e exige alguns passos extras de servidor e de rede. As diferenças em relação ao local são: (1) você acessa a máquina por **SSH**, (2) precisa **liberar a porta 8501 em duas camadas de firewall**, (3) o Streamlit precisa escutar em **`0.0.0.0`** (e não só em localhost) e (4) o app é mantido no ar por um serviço **`systemd`**.

**1) Conecte na VM via SSH** (rode no seu computador; ajuste o caminho da chave):
```bash
ssh -i caminho/para/ssh-key.key {user_vm_oracle}@{ip_publico}
```
> No Windows, antes de conectar, ajuste a permissão da chave com `icacls`. No macOS/Linux use `chmod 600 caminho/para/ssh--key.key`.

**2) No servidor, instale o básico e clone o repositório:**
```bash
sudo apt update && sudo apt install -y python3-venv python3-pip git
git clone https://github.com/SEU_USUARIO/chatbot-engenharia-prompt.git
cd chatbot-engenharia-prompt
```

**3) Crie o ambiente virtual e instale as dependências:**
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

**4) Crie o `.env` no servidor** (ele não vem do GitHub, pois é ignorado pelo `.gitignore`):
```bash
echo "NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI" > .env
```

**5) Libere a porta 8501 nas DUAS camadas de firewall:**
- **Camada 1 — Oracle:** no console da Oracle Cloud, vá em *Networking → VCN → Security List* e adicione uma **Ingress Rule**: Source `0.0.0.0/0`, protocolo `TCP`, porta de destino `8501`.
- **Camada 2 — Ubuntu (no servidor):**
  ```bash
  sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8501 -j ACCEPT
  sudo netfilter-persistent save
  ```

**6) Rode escutando em todas as interfaces** (`0.0.0.0` é essencial para acesso externo):
```bash
streamlit run app.py --server.address 0.0.0.0 --server.port 8501
```
Teste no navegador: `http://{ip_publico}.116:8501`.

**7) Mantenha o app no ar permanentemente com `systemd`** (continua rodando após fechar o SSH). Crie `/etc/systemd/system/chatbot.service` com:
```ini
[Unit]
Description=Chatbot Streamlit - Engenharia de Prompt
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/chatbot-engenharia-prompt
ExecStart=/home/ubuntu/chatbot-engenharia-prompt/.venv/bin/streamlit run app.py --server.address 0.0.0.0 --server.port 8501
Restart=always

[Install]
WantedBy=multi-user.target
```
E ative o serviço:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now chatbot
sudo systemctl status chatbot     # deve aparecer "active (running)"
```

> Para atualizar a aplicação depois: `git pull` na pasta do projeto, no servidor, e `sudo systemctl restart chatbot`. O `.env` não é afetado pelo `git pull`.

---

## Estrutura do projeto

```
chatbot-engenharia-prompt/
├── app.py              # Aplicação Streamlit (código-fonte)
├── requirements.txt    # Dependências Python
├── .gitignore          # Ignora .env, chaves e arquivos sensíveis
└── README.md           # Este relatório + instruções de instalação e execução

# .env (não versionado) — criado manualmente com a NVIDIA_API_KEY, na máquina local e no servidor
```

---

*Projeto desenvolvido como atividade prática de implantação de chatbot com modelos open source na nuvem.*
