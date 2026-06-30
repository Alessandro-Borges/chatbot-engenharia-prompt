# 🚀 Guia Passo a Passo — Deploy do Chatbot na Oracle Cloud

Guia completo e detalhado, do zero até o chatbot rodando em produção e acessível pelo navegador.
Pensado para quem tem **pouca experiência**: cada passo está numerado, com os comandos prontos para **Windows** e para **macOS/Linux**.

> **Como usar este guia:** sempre que houver dois blocos de comando — um marcado **🪟 Windows (PowerShell)** e outro **🍎 macOS / 🐧 Linux** — escolha o do **seu** sistema operacional. Os comandos rodados **dentro do servidor Oracle** são sempre Linux (porque o servidor é Ubuntu), independentemente do seu computador.

---

## 📋 Dados da sua infraestrutura (já preenchidos)

| Item | Valor |
|------|-------|
| IP público da VM | `146.235.57.116` |
| Usuário SSH | `ubuntu` |
| Sistema operacional do servidor | Ubuntu 22.04 LTS |
| Arquivo da chave privada SSH | `xxxxxxx.key` |
| Porta da aplicação | `8501` |
| URL final (no navegador) | `http://146.235.57.116:8501` |

---

## 🗺️ Visão geral do fluxo (o caminho completo)

```
   SEU COMPUTADOR                         GITHUB                    SERVIDOR ORACLE (Ubuntu)
 ┌─────────────────┐               ┌─────────────────┐            ┌──────────────────────────┐
 │ 1. Preparar     │               │                 │            │                          │
 │ 2. Testar local │── git push ──▶│  Repositório    │── clone ──▶│ 4. Instalar + configurar │
 │    (Partes 1-4) │   (Parte 5)   │  (código limpo, │  (Parte 8) │ 5. Liberar porta 8501    │
 │                 │               │  SEM segredos)  │            │ 6. Rodar (Partes 9-11)   │
 └─────────────────┘               └─────────────────┘            │ 7. Manter no ar (12)     │
                                                                   └──────────────────────────┘
                                                                              │
                                                                   http://146.235.57.116:8501
                                                                              ▼
                                                                       🌐 Navegador
```

**Resumo da estratégia:** você testa na sua máquina, envia o código para o GitHub (sem segredos), e no servidor você **clona** o repositório e cria o `.env` na hora. Assim, só o que é necessário vai para o servidor.

---

## ⚠️ PARTE 0 — Aviso de segurança (leia antes de tudo)

Os arquivos que você forneceu (`app.py`, a **chave SSH privada**, etc.) estavam salvos como **páginas do GitHub**. Isso sugere que podem já estar em um repositório **público**. Uma chave SSH privada exposta publicamente está **comprometida**.

**0.1.** Gere uma **nova chave de API da NVIDIA** em https://build.nvidia.com (menu → *Get API Key*) e descarte a antiga.

**0.2.** Nunca envie a chave SSH (`*.key`) nem o arquivo `.env` para o GitHub. Este projeto já vem com um `.gitignore` que bloqueia esses arquivos.

**0.3.** Se você tem um repositório antigo com esses arquivos, considere **apagá-lo** e criar um novo limpo (veja a Parte 5).

> Sobre o `.env`: você mencionou que **não tem** um `.env` originalmente — e está correto. **Não existe um `.env` pronto neste projeto.** Você vai **criá-lo do zero** duas vezes: uma na sua máquina (Passo 4.4) e outra no servidor (Passo 9.4). E o arquivo `.gitignore`, que impede o `.env` e as chaves de irem para o GitHub, também será **criado do zero pelo terminal** na Parte 5.

---

# PARTE 1 — Pré-requisitos no seu computador

Antes de tudo, você precisa de três coisas instaladas: **Python**, **Git** e um **cliente SSH** (que no Windows 10/11 e no Mac já vem embutido).

## 1.1 Verifique o Python

### 🪟 Windows (PowerShell)
```powershell
python --version
```
- Se aparecer `Python 3.10` (ou maior), está pronto.
- Se der erro ou abrir a Microsoft Store, instale o Python em https://www.python.org/downloads/ e, **durante a instalação, marque a caixa "Add Python to PATH"**. Depois feche e reabra o PowerShell.

### 🍎 macOS / 🐧 Linux
```bash
python3 --version
```
- Se aparecer `Python 3.10` (ou maior), está pronto.
- No macOS, se não tiver, instale com `brew install python` (precisa do [Homebrew](https://brew.sh)). No Linux: `sudo apt install python3 python3-venv python3-pip`.

## 1.2 Verifique o Git

### 🪟 Windows (PowerShell)
```powershell
git --version
```
- Se der erro, baixe e instale em https://git-scm.com/download/win. Durante a instalação, aceite as opções padrão. **Isso também instala o "Git Bash"**, um terminal útil para comandos Linux no Windows.

### 🍎 macOS / 🐧 Linux
```bash
git --version
```
- No macOS, se pedir, aceite instalar as "Command Line Tools". No Linux: `sudo apt install git`.

## 1.3 Verifique o SSH

### 🪟 Windows (PowerShell)
```powershell
ssh
```
- Se aparecer a ajuda do comando (texto com "usage: ssh"), está pronto. O Windows 10/11 já vem com SSH.
- Se não tiver, instale em *Configurações → Aplicativos → Recursos opcionais → Adicionar recurso → "Cliente OpenSSH"*.

### 🍎 macOS / 🐧 Linux
```bash
ssh -V
```
- Já vem instalado em praticamente todos os sistemas.

---

# PARTE 2 — Conhecendo os arquivos do projeto

A pasta `chatbot-engenharia-prompt/` que você recebeu contém:

| Arquivo | Para que serve | Vai para o GitHub? | Vai para o servidor? |
|---------|----------------|:------------------:|:--------------------:|
| `app.py` | A aplicação Streamlit (código-fonte) | ✅ Sim (entregável) | ✅ Sim (essencial) |
| `requirements.txt` | Lista de bibliotecas Python | ✅ Sim (entregável) | ✅ Sim (essencial) |
| `.gitignore` | Diz ao Git o que ignorar (você cria na Parte 5) | ✅ Sim (entregável) | ➖ Indiferente |
| `README.md` | Relatório + instruções de instalação/execução | ✅ Sim (entregável) | ➖ Opcional |
| `GUIA_PASSO_A_PASSO.md` | Este guia | ➖ Não (uso pessoal / consulta) | ➖ Opcional |
| `.env` | **Suas chaves secretas** (você cria nos Passos 4.4 e 9.4) | ❌ **NUNCA** | ✅ Sim (criado no servidor) |

> 🎯 **O entregável final** é um **repositório público** contendo **somente 4 arquivos**: `app.py`, `requirements.txt`, `.gitignore` e `README.md`. O `.env` e a chave SSH ficam de fora (protegidos pelo `.gitignore`), e este guia fica só na sua máquina, para consulta.

> ℹ️ **Por que você talvez não veja `.gitignore` e `.env` no Finder/Explorador?** Arquivos que começam com **ponto** (`.`) são "ocultos" por padrão. No macOS, aperte `Cmd + Shift + .` no Finder para exibi-los; no Windows, ative *Exibir → Itens ocultos* no Explorador. De qualquer forma, neste guia você vai **criá-los pelo terminal**, então não precisa se preocupar em vê-los.

> **Os únicos arquivos realmente necessários para a aplicação rodar são `app.py`, `requirements.txt` e o `.env`.** O resto é documentação.

---

# PARTE 3 — Abrir o terminal na pasta do projeto

Todos os comandos das próximas partes (no **seu computador**) devem ser executados **de dentro da pasta do projeto**. Faça isto primeiro:

## 3.1 Abra o terminal

### 🪟 Windows
- Abra o **PowerShell**: tecle `Win`, digite `PowerShell` e abra.

### 🍎 macOS / 🐧 Linux
- Abra o app **Terminal**.

## 3.2 Entre na pasta do projeto

Substitua o caminho pelo local onde a pasta `chatbot-engenharia-prompt` está no seu computador.

### 🪟 Windows (PowerShell)
```powershell
cd "C:\Users\SEU_USUARIO\Documents\chatbot-engenharia-prompt"
```

### 🍎 macOS / 🐧 Linux
```bash
cd ~/Documents/chatbot-engenharia-prompt
```

## 3.3 Confirme que está no lugar certo
```powershell
# Windows
dir
```
```bash
# macOS / Linux
ls -a
```
Você deve ver `app.py`, `requirements.txt`, `.gitignore`, etc. listados.

---

# PARTE 4 — Testar a aplicação no seu computador (local)

Garantir que funciona localmente **antes** de mexer no servidor evita debugar dois problemas ao mesmo tempo.

## 4.1 Crie o ambiente virtual (venv)

O *venv* é uma "caixa isolada" onde as bibliotecas do projeto ficam separadas do resto do sistema.

### 🪟 Windows (PowerShell)
```powershell
python -m venv .venv
```

### 🍎 macOS / 🐧 Linux
```bash
python3 -m venv .venv
```

## 4.2 Ative o ambiente virtual

### 🪟 Windows (PowerShell)
```powershell
.\.venv\Scripts\Activate.ps1
```
> **Se aparecer um erro de "execução de scripts desabilitada"**, rode o comando abaixo uma vez e tente ativar de novo:
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> ```

### 🍎 macOS / 🐧 Linux
```bash
source .venv/bin/activate
```

✅ **Como saber se ativou:** aparece `(.venv)` no começo da linha do terminal.

## 4.3 Instale as dependências

(Com o venv ativado — os comandos abaixo valem para todos os sistemas.)
```bash
pip install --upgrade pip
pip install -r requirements.txt
```
> Isso instala Streamlit, chatlas, openai e python-dotenv. Pode levar 1–2 minutos.

## 4.4 Crie o arquivo `.env` (com a sua chave)

Este é o passo em que **criamos o `.env` do zero**, no momento em que ele é necessário. Ele guarda sua chave secreta da NVIDIA.

### 🪟 Windows (PowerShell)
```powershell
# Cria o arquivo .env já com a variável (troque pela sua chave real)
"NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI" | Out-File -FilePath .env -Encoding utf8
```
Para conferir o conteúdo: `type .env`

### 🍎 macOS / 🐧 Linux
```bash
# Cria o arquivo .env já com a variável (troque pela sua chave real)
echo "NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI" > .env
```
Para conferir o conteúdo: `cat .env`

> 🔑 **Onde conseguir a chave:** https://build.nvidia.com → faça login → *Get API Key*. A chave começa com `nvapi-`.

## 4.5 Rode a aplicação localmente
```bash
streamlit run app.py
```
- O terminal mostra um endereço `Local URL: http://localhost:8501`.
- O navegador deve abrir sozinho. Se não abrir, acesse `http://localhost:8501` manualmente.

## 4.6 Teste

Digite uma pergunta no chat (ex.: *"O que é engenharia de prompt?"*). Se o bot responder, **está funcionando**. 🎉

## 4.7 Pare a aplicação

No terminal, pressione `Ctrl + C`.

> ✅ **Só avance para o servidor depois que o teste local funcionar.**

---

# PARTE 5 — Criar o repositório PÚBLICO final no GitHub (entregável)

> 🎯 **Objetivo desta parte:** criar um **repositório público novo** contendo **apenas 4 arquivos**: o código-fonte (`app.py`), a lista de dependências (`requirements.txt`), o `.gitignore` e o `README.md`. Tudo o que é segredo — o `.env` com a `NVIDIA_API_KEY`, a chave SSH, etc. — **fica de fora**, e é o `.gitignore` que garante isso automaticamente na hora do commit.

Como o repositório é **público**, qualquer pessoa pode ver o conteúdo. Por isso o `.gitignore` precisa estar configurado **antes** do primeiro commit. A ordem é: (1) criar o `.gitignore`, (2) adicionar **só** os arquivos do entregável, (3) conferir, (4) enviar.

> 💡 **Sobre os arquivos:** o `.env` (com a chave) e a chave SSH **continuam existindo na sua máquina** e a aplicação continua usando o `.env` normalmente — eles só não são enviados ao GitHub. O acesso à VM Oracle continua sendo feito da mesma forma de sempre (via a chave SSH, que permanece local). Já o `GUIA_PASSO_A_PASSO.md` (este arquivo) é de uso pessoal e **não** entra no repositório.

## 5.1 Entenda: o que NÃO pode ir para o GitHub e por quê

Alguns arquivos contêm segredos. Se forem para um repositório (ainda mais público), qualquer pessoa pode copiá-los:

| Arquivo | O que tem dentro | Risco se vazar |
|---------|------------------|----------------|
| `.env` | A sua `NVIDIA_API_KEY` | Terceiros usam sua chave, gastam seu limite e podem gerar custos em seu nome |
| `ssh-key-2026-03-14.key` | Chave **privada** SSH do servidor | Qualquer um entra na sua VM da Oracle |
| `.venv/` | O ambiente virtual (centenas de arquivos) | Não é segredo, mas é lixo pesado e desnecessário no repositório |

> 💡 **O detalhe-chave que você pediu:** o `.gitignore` faz o Git **ignorar** o `.env`, mas o arquivo **continua no seu disco** e a aplicação **continua usando ele** normalmente (via `load_dotenv()`). Ou seja: o segredo fica fora do GitHub, **mas o app roda perfeitamente**, porque ler um arquivo do disco não tem nada a ver com o que o Git versiona.

## 5.2 Crie o arquivo `.gitignore` (do zero, pelo terminal)

> Faça isto **na pasta do projeto, no seu computador**.

**Primeiro, entenda o que o `.gitignore` vai bloquear e por quê:**
- `.env` e `.env.*` → o arquivo com a `NVIDIA_API_KEY` (e variações como `.env.local`). **É o mais importante.**
- `*.key`, `*.pem`, `*.ppk`, `*.pub`, `ssh-key-*`, `id_rsa*` → **qualquer** chave SSH ou certificado.
- `.venv/`, `venv/`, `env/` → o ambiente virtual (não é segredo, mas é lixo pesado).
- `__pycache__/`, `*.py[cod]` → arquivos temporários do Python.
- `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/`, `*.log` → lixo do sistema operacional e de editores.

**Agora, crie o `.gitignore` com UM único comando** (copie a linha inteira e cole no terminal):

### 🍎 macOS / 🐧 Linux
```bash
printf '.env\n.env.*\n*.key\n*.pem\n*.ppk\n*.pub\nid_rsa*\nssh-key-*\n.venv/\nvenv/\nenv/\n__pycache__/\n*.py[cod]\n.streamlit/secrets.toml\n.DS_Store\nThumbs.db\n.idea/\n.vscode/\n*.log\n' > .gitignore
```

### 🪟 Windows (PowerShell)
```powershell
".env`n.env.*`n*.key`n*.pem`n*.ppk`n*.pub`nid_rsa*`nssh-key-*`n.venv/`nvenv/`nenv/`n__pycache__/`n*.py[cod]`n.streamlit/secrets.toml`n.DS_Store`nThumbs.db`n.idea/`n.vscode/`n*.log" | Out-File .gitignore -Encoding utf8
```

**Confira que o arquivo foi criado** (deve listar as regras acima):
```bash
cat .gitignore        # macOS / Linux
```
```powershell
type .gitignore       # Windows
```

## 5.3 Inicialize o repositório e adicione SOMENTE os 4 arquivos do entregável

Estes comandos transformam a pasta em um repositório Git e selecionam **apenas** os arquivos que devem ir para o repositório público. Valem para todos os sistemas.
```bash
git init
git add app.py requirements.txt .gitignore README.md
```
- `git init` → transforma a pasta em um repositório Git (cria a pasta oculta `.git`).
- `git add app.py requirements.txt .gitignore README.md` → adiciona **explicitamente só esses 4 arquivos**. Diferente de `git add .` (que pegaria tudo na pasta), aqui você controla exatamente o que entra. Assim, o `GUIA_PASSO_A_PASSO.md` fica de fora (uso pessoal) e nenhum arquivo extra é incluído por engano.

> ✅ Mesmo que você usasse `git add .`, o `.gitignore` impediria o `.env` e as chaves de entrarem. Usar a lista explícita é uma **camada extra de segurança** e garante o repositório enxuto que o entregável pede.

## 5.4 ⚠️ CONFIRA o que será enviado (passo crítico de segurança)

Antes de qualquer envio, verifique o que entrou:
```bash
git status
```
- **DEVE aparecer** (em verde / "Changes to be committed"), e **somente** isto: `app.py`, `requirements.txt`, `.gitignore`, `README.md`.
- **NÃO PODE aparecer**: `.env`, `ssh-key-2026-03-14.key` (nem qualquer `.key`), `.venv/`. (O `GUIA_PASSO_A_PASSO.md` vai aparecer como "Untracked" — está correto, ele fica de fora de propósito.)

Garantia extra — o comando abaixo deve **imprimir** `.env`, confirmando que ele está sendo ignorado:
```bash
git check-ignore .env
```
> 🚨 Se `.env` ou qualquer `.key` aparecerem entre os arquivos a serem commitados, **PARE**. Revise o `.gitignore` (Passo 5.2) antes de continuar. Não faça o commit enquanto eles aparecerem.

## 5.5 Faça o commit (registra a versão localmente)
```bash
git commit -m "Chatbot de Engenharia de Prompt - IA generativa na Oracle Cloud"
```

## 5.6 Crie o repositório PÚBLICO vazio no site do GitHub
1. Acesse https://github.com e faça login.
2. Clique no botão **New** (ou no `+` no topo → **New repository**).
3. **Repository name:** `chatbot-engenharia-prompt`.
4. Marque a opção **Public** ✅ (este é o requisito do entregável).
5. **NÃO** marque "Add a README", "Add .gitignore" nem "Choose a license" — já temos os nossos (e marcar criaria conflito no primeiro push).
6. Clique em **Create repository**.
7. Na página seguinte, **copie a URL** que aparece (algo como `https://github.com/SEU_USUARIO/chatbot-engenharia-prompt.git`).

## 5.7 Conecte sua pasta ao GitHub e faça o upload (push)

Estes comandos ligam a pasta local ao repositório do GitHub e enviam os arquivos. Troque `SEU_USUARIO` pelo seu usuário.
```bash
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/chatbot-engenharia-prompt.git
git push -u origin main
```
- `git branch -M main` → nomeia a branch principal como `main`.
- `git remote add origin ...` → registra o endereço do seu repositório no GitHub.
- `git push -u origin main` → **envia (faz o upload)** os arquivos para o GitHub.

> 🔐 **Vai pedir login?** O GitHub **não aceita mais a senha normal** no push. Use seu usuário e, no lugar da senha, um **Personal Access Token (PAT)**. Para criar: *GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token*, marque o escopo `repo`, gere e cole no lugar da senha.

## 5.8 Confirme no navegador (verificação final de segurança)
Abra `https://github.com/SEU_USUARIO/chatbot-engenharia-prompt` no navegador e confira a lista de arquivos:
- ✅ Devem estar lá (e somente estes): `app.py`, `requirements.txt`, `.gitignore`, `README.md`.
- ❌ **NÃO** podem estar lá: `.env`, `ssh-key-2026-03-14.key` (nem qualquer `.key`).

> Se por acidente um segredo subiu: apague o arquivo, faça novo commit, **e troque a credencial exposta** (gere nova `NVIDIA_API_KEY` e nova chave SSH). Apagar não basta — o histórico do Git guarda versões antigas, então a credencial deve ser considerada comprometida.

## 5.9 Atualizações futuras (como reenviar mudanças)
Sempre que mudar algo no projeto, o ciclo de upload é (adicione só os arquivos do entregável para manter o repositório enxuto):
```bash
git add app.py requirements.txt .gitignore README.md
git commit -m "descreva a mudança aqui"
git push
```
> Confirme sempre com `git status` antes do commit, e nunca commite o `.env` nem chaves.

---

# PARTE 6 — Conectar no servidor Oracle via SSH

Agora vamos entrar no servidor. Os comandos abaixo rodam **no seu computador**.

## 6.1 Localize a chave privada

Anote o caminho completo até o arquivo `ssh-key-2026-03-14.key`. Exemplos:
- 🪟 Windows: `C:\Users\SEU_USUARIO\Downloads\ssh-key-2026-03-14.key`
- 🍎 macOS / 🐧 Linux: `~/Downloads/ssh-key-2026-03-14.key`

## 6.2 Ajuste a permissão da chave (passo obrigatório)

O SSH **recusa** chaves que estejam "abertas demais" para outros usuários. Cada sistema corrige de um jeito.

### 🪟 Windows (PowerShell)
No Windows não existe `chmod`; usamos `icacls`. Troque o caminho pelo seu:
```powershell
icacls "C:\Users\SEU_USUARIO\Downloads\ssh-key-2026-03-14.key" /inheritance:r
icacls "C:\Users\SEU_USUARIO\Downloads\ssh-key-2026-03-14.key" /grant:r "$($env:USERNAME):R"
```
> O primeiro comando remove permissões herdadas; o segundo dá acesso de leitura só para você.

### 🍎 macOS / 🐧 Linux
```bash
chmod 600 ~/Downloads/ssh-key-2026-03-14.key
```

## 6.3 Conecte ao servidor

Troque o caminho da chave pelo seu.

### 🪟 Windows (PowerShell)
```powershell
ssh -i "C:\Users\SEU_USUARIO\Downloads\ssh-key-2026-03-14.key" {user}@{public_ip}
```

### 🍎 macOS / 🐧 Linux
```bash
ssh -i ~/Downloads/ssh-key-2026-03-14.key {user}@{public_ip}
```

- Na primeira conexão aparece `Are you sure you want to continue connecting (yes/no)?` → digite `yes` e Enter.
- Se aparecer um prompt como `ubuntu@instance-...:~$`, **você está dentro do servidor!** 🎉

> ❌ **Erro `Permission denied (publickey)`?** Causas comuns: caminho da chave errado, faltou ajustar a permissão (6.2), ou usuário diferente de `ubuntu`.

---

# PARTE 7 — Preparar o servidor

> ⚠️ **A partir daqui, os comandos rodam DENTRO do servidor** (depois do `ssh`). O prompt começa com `ubuntu@...`. São todos comandos Linux.

## 7.1 Atualize o sistema
```bash
sudo apt update && sudo apt upgrade -y
```
> Pode demorar alguns minutos. Se aparecer alguma tela roxa pedindo confirmação, aceite as opções padrão (Enter).

## 7.2 Instale Python, pip, venv e git
```bash
sudo apt install -y python3 python3-pip python3-venv git
```

## 7.3 Confirme as versões
```bash
python3 --version
git --version
```

---

# PARTE 8 — Migrar APENAS os arquivos necessários para o servidor

Aqui você tem duas opções. A **Opção A (clonar do GitHub)** é a recomendada: traz só os arquivos versionados (e o `.env`, que é ignorado, naturalmente fica de fora). A **Opção B (SCP)** copia arquivos específicos direto do seu PC.

## Opção A — Clonar do GitHub (recomendado) ✅

> Vantagem: traz exatamente os arquivos do repositório (código limpo, sem `.env`, sem `.venv`). É a forma mais limpa de "replicar" o projeto no servidor.

**8.A.1.** Dentro do servidor, vá para a pasta home e clone (troque `SEU_USUARIO`):
```bash
cd ~
git clone https://github.com/SEU_USUARIO/chatbot-engenharia-prompt.git
```

**8.A.2.** Entre na pasta:
```bash
cd ~/chatbot-engenharia-prompt
ls -a
```
Você verá `app.py`, `requirements.txt`, etc. — mas **não** o `.env` (ele será criado no Passo 9.4).

## Opção B — Copiar arquivos específicos com SCP

> Use esta opção se não quiser usar o GitHub. O SCP copia arquivos do seu PC direto para o servidor. Aqui copiamos **somente os dois arquivos essenciais**.

> ⚠️ Estes comandos rodam **no seu computador** (abra um **novo** terminal, sem estar dentro do SSH).

**8.B.1.** Crie a pasta no servidor (rode do seu PC):

🪟 Windows (PowerShell):
```powershell
ssh -i "C:\caminho\ssh-key-2026-03-14.key" {user}@{public_ip} "mkdir -p ~/chatbot-engenharia-prompt"
```
🍎 macOS / 🐧 Linux:
```bash
ssh -i ~/Downloads/ssh-key-2026-03-14.key {user}@{public_ip} "mkdir -p ~/chatbot-engenharia-prompt"
```

**8.B.2.** Copie **apenas** `app.py` e `requirements.txt` (rode do seu PC, de dentro da pasta do projeto):

🪟 Windows (PowerShell):
```powershell
scp -i "C:\caminho\ssh-key-2026-03-14.key" app.py requirements.txt {user}@{public_ip}:~/chatbot-engenharia-prompt/
```
🍎 macOS / 🐧 Linux:
```bash
scp -i ~/caminho/ssh-key-2026-03-14.key app.py requirements.txt {user}@{public_ip}:~/chatbot-engenharia-prompt/
```

> 💡 Note que **não copiamos o `.env`** nem a chave SSH. O `.env` será criado direto no servidor no Passo 9.4. Isso é o "migrar somente o necessário".

**8.B.3.** Volte para o terminal que está conectado ao servidor (ou conecte de novo via SSH) e entre na pasta:
```bash
cd ~/chatbot-engenharia-prompt
ls -a
```

---

# PARTE 9 — Instalar e configurar a aplicação no servidor

> Comandos **dentro do servidor**, na pasta `~/chatbot-engenharia-prompt`.

## 9.1 Crie o ambiente virtual
```bash
cd ~/chatbot-engenharia-prompt
python3 -m venv .venv
```

## 9.2 Ative o ambiente virtual
```bash
source .venv/bin/activate
```
✅ Deve aparecer `(.venv)` no início da linha.

## 9.3 Instale as dependências
```bash
pip install --upgrade pip
pip install -r requirements.txt
```

## 9.4 Crie o arquivo `.env` no servidor (com a sua chave)

Como o `.env` não veio (nem do GitHub, nem do SCP), criamos ele aqui — no momento necessário. Vamos usar o editor `nano`:
```bash
nano .env
```
Na tela do editor, digite (troque pela sua chave real):
```
NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI
```
Salve e saia: pressione `Ctrl + O`, depois `Enter`, depois `Ctrl + X`.

Confirme que ficou certo:
```bash
cat .env
```

> 💡 **Alternativa em uma linha** (sem abrir o editor):
> ```bash
> echo "NVIDIA_API_KEY=nvapi-SUA_CHAVE_AQUI" > .env
> ```

---

# PARTE 10 — Liberar a porta 8501 (as DUAS camadas de firewall)

A Oracle bloqueia portas em **dois lugares diferentes**. Você precisa abrir a porta `8501` nas **duas**, ou o navegador nunca vai conectar.

## 10.1 Camada 1 — Security List da Oracle (no painel web do navegador)

> Este passo é feito no **site da Oracle Cloud**, não no terminal.

**10.1.1.** Acesse https://cloud.oracle.com e faça login.

**10.1.2.** No menu (☰) vá em **Networking → Virtual Cloud Networks**.

**10.1.3.** Clique na sua **VCN** (rede virtual).

**10.1.4.** No menu lateral, clique em **Security Lists** e abra a **Default Security List**.

**10.1.5.** Clique em **Add Ingress Rules** e preencha:
- **Stateless:** deixe desmarcado
- **Source Type:** `CIDR`
- **Source CIDR:** `0.0.0.0/0`  *(qualquer IP pode acessar)*
- **IP Protocol:** `TCP`
- **Destination Port Range:** `8501`
- **Description:** `Streamlit chatbot`

**10.1.6.** Clique em **Add Ingress Rules** para salvar. A regra deve aparecer na lista.

## 10.2 Camada 2 — Firewall do Ubuntu (iptables)

> Este passo roda **dentro do servidor** (no SSH). A imagem Ubuntu da Oracle vem com `iptables` bloqueando portas novas.

**10.2.1.** Libere a porta 8501:
```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8501 -j ACCEPT
```

**10.2.2.** Salve a regra para não se perder ao reiniciar:
```bash
sudo netfilter-persistent save
```
> Se der "command not found", instale e tente de novo:
> ```bash
> sudo apt install -y iptables-persistent
> sudo netfilter-persistent save
> ```

---

# PARTE 11 — Rodar e testar no navegador

## 11.1 Inicie a aplicação (teste rápido)

> Dentro do servidor, na pasta do projeto, com o venv ativado.
```bash
cd ~/chatbot-engenharia-prompt
source .venv/bin/activate
streamlit run app.py --server.address 0.0.0.0 --server.port 8501
```
> 🔑 O `--server.address 0.0.0.0` é **essencial**: faz o Streamlit aceitar conexões **de fora** (por padrão ele só aceita conexões locais e o navegador não conseguiria acessar).

## 11.2 Acesse pelo navegador

No **seu computador**, abra o navegador e acesse:
```
http://{public_ip}:8501
```
> ⚠️ Use `http://` (e **não** `https://`). Se o chatbot aparecer e responder, **está no ar!** 🎉

## 11.3 Se não abrir, verifique nesta ordem:
1. A regra de Ingress da Oracle foi salva? (10.1)
2. A regra do iptables foi aplicada? (10.2)
3. O Streamlit está rodando com `--server.address 0.0.0.0`? (11.1)
4. Você usou `http://` e a porta `:8501`?

## 11.4 Pare o teste

`Ctrl + C` no terminal do servidor. (Vamos deixá-lo permanente na próxima parte.)

---

# PARTE 12 — Deixar a aplicação no ar para sempre (systemd)

Com o método do Passo 11, o app **morre** quando você fecha o SSH. Vamos transformá-lo em um serviço que roda sozinho e reinicia automaticamente.

## 12.1 Crie o arquivo do serviço
```bash
sudo nano /etc/systemd/system/chatbot.service
```

## 12.2 Cole exatamente este conteúdo:
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
Salve e saia: `Ctrl + O`, `Enter`, `Ctrl + X`.

## 12.3 Ative e inicie o serviço
```bash
sudo systemctl daemon-reload
sudo systemctl enable chatbot
sudo systemctl start chatbot
```

## 12.4 Confirme que está rodando
```bash
sudo systemctl status chatbot
```
> Deve aparecer `active (running)` em verde. Pressione `q` para sair da tela de status.

## 12.5 Teste de novo no navegador
Acesse `http://{public_ip}:8501`. Agora ele continua no ar mesmo que você feche o SSH. ✅

## 12.6 Comandos úteis do serviço
```bash
sudo systemctl restart chatbot     # reiniciar (use após atualizar o código)
sudo systemctl stop chatbot        # parar
sudo journalctl -u chatbot -f      # ver os logs ao vivo (Ctrl+C para sair)
```

---

# PARTE 13 — Como atualizar a aplicação depois (manutenção)

Quando você mudar o código no futuro:

**13.1.** No seu PC, faça as alterações, teste localmente e envie ao GitHub:
```bash
git add .
git commit -m "descrição da mudança"
git push
```

**13.2.** No servidor (via SSH), puxe a atualização e reinicie o serviço:
```bash
cd ~/chatbot-engenharia-prompt
git pull
sudo systemctl restart chatbot
```
> O `.env` no servidor **não é afetado** pelo `git pull` (ele é ignorado pelo Git), então suas chaves permanecem intactas.

---

# ✅ PARTE 14 — Checklist final de entrega

- [ ] App responde em `http://{public_ip}:8501` pelo navegador
- [ ] App rodando como serviço `systemd` (continua no ar com o SSH fechado)
- [ ] Repositório no GitHub contém: `app.py`, `requirements.txt`, `.gitignore`, `README.md`
- [ ] Repositório **NÃO** contém `.env` nem a chave SSH (`.key`)
- [ ] `README.md` tem o relatório completo (Introdução, Infra, Modelo, Desenvolvimento, Implantação, Discussão)
- [ ] Chave de API da NVIDIA rotacionada (se a antiga foi exposta)
- [ ] Link do GitHub + link do IP enviados ao professor

---

# 🔧 PARTE 15 — Solução de problemas (troubleshooting)

| # | Problema | Causa provável | Solução |
|---|----------|----------------|---------|
| 15.1 | `Permission denied (publickey)` ao conectar | Permissão da chave ou caminho errado | Refaça o Passo 6.2 (`chmod 600` ou `icacls`); confira o caminho e use o usuário `ubuntu` |
| 15.2 | No Windows, `Activate.ps1` dá erro de script | Política de execução do PowerShell | Rode `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` (Passo 4.2) |
| 15.3 | Navegador não abre o `:8501` | Porta bloqueada | Revise a Security List (10.1) **e** o iptables (10.2) |
| 15.4 | Página abre mas dá erro ao responder | `.env` ausente ou chave inválida | Confira o `.env` no servidor (`cat .env`) e a `NVIDIA_API_KEY` |
| 15.5 | App cai ao fechar o SSH | Rodando direto no terminal | Configure o serviço `systemd` (Parte 12) |
| 15.6 | `ModuleNotFoundError` | venv não ativado | Rode `source .venv/bin/activate` antes de `streamlit run` |
| 15.7 | Mudei o código e não atualizou no site | Serviço usando versão antiga | `git pull` + `sudo systemctl restart chatbot` (Parte 13) |
| 15.8 | `git push` pede senha e recusa | GitHub não aceita mais senha | Use um Personal Access Token como senha (Passo 5.6) |
| 15.9 | `netfilter-persistent: command not found` | Pacote não instalado | `sudo apt install -y iptables-persistent` e salve de novo |
```
