# Biohazard — TryHackMe
 
> **Difficulty:** Medium  
> **Category:** Web Enumeration · Cryptography · Steganography · Privilege Escalation  
> **Date:** 2025-07-12  
> **Room:** https://tryhackme.com/room/biohazard
 
---
 
## overview
 
Sala temática de Resident Evil. A proposta não é explorar vulnerabilidades de serviço — é seguir uma cadeia longa de pistas escondidas em diretórios web, decodificar criptografias encadeadas, extrair dados via steganografia de imagens e só então obter acesso SSH. A escalação final para root vem de uma misconfiguration crítica de sudo. A sala exige atenção a detalhes desde o início: informações coletadas cedo se tornam chaves de decriptação muito mais tarde.
 
---
 
## reconnaissance
 
```bash
nmap -sC -sV -p- <TARGET_IP>
```
 
| porta | serviço | versão |
|---|---|---|
| 21/tcp | ftp | vsftpd 3.0.3 |
| 22/tcp | ssh | OpenSSH 7.6p1 |
| 80/tcp | http | Apache |
 
FTP sem login anônimo. SSH sem credenciais ainda. Ponto de entrada: porta 80.
 
---
 
## enumeration
 
### Porta 80 — /mansionmain/
 
A página inicial tem um link para `/mansionmain/`. Inspecionando o source code:
 
```html
<!-- It is in the /diningRoom/ -->
```
 
A mansão começa a falar. Seguimos.
 
---
 
### /diningRoom/ — emblem flag + hint base64
 
O dining room tem um link para pegar o emblema na parede — isso gera o **emblem flag**.
 
No source code há uma string em Base64:
 
```
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
```
 
Decodificando:
 
```bash
echo "SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=" | base64 -d
# How about the /teaRoom/
```
 
 
 
---
 
### /teaRoom/ — lockpick flag
 
O tea room contém o **lockpick flag** e uma nota indicando que Jill deve visitar o `/artRoom/`.
 
---
 
### /artRoom/ — mapa da mansão
 
O art room revela um mapa com todos os diretórios:
 
```
/diningRoom/    /teaRoom/       /artRoom/      /barRoom/
/diningRoom2F/  /tigerStatusRoom/  /galleryRoom/
/studyRoom/     /armorRoom/     /attic/
```
 
Roadmap completo. A partir daqui, enumeração sistemática sala por sala.
 
---
 
### /barRoom/ — Base32 → music sheet flag → gold emblem
 
A porta do bar room é travada. Usamos o **lockpick flag** para entrar.
 
Dentro há uma nota com texto codificado em **Base32**. Decodificando via CyberChef:
 
```
Base32 → music sheet flag
```
 
Submetendo o music sheet flag no piano da sala, surge um **gold emblem** na parede.
 
Tentando o **gold emblem** diretamente no gold emblem slot não produz resultado. Porém, submetendo o **emblem original do dining room** no mesmo slot redireciona para uma página que revela o nome **"rebecca"** — anotado para uso posterior como chave de decriptação.
 
---
 
### /diningRoom/ — gold emblem slot → Vigenere → shield key
 
Submetendo o **gold emblem** no dining room retorna um texto cifrado. Parece ROT13 mas não é — é **Vigenere Cipher** com a chave **"rebecca"** (obtida na etapa anterior, ao submeter o emblem original no slot do bar room).
 
Decodificando:
 
```
Vigenere (key: rebecca) → "there is a shield key inside the dining room. The html page is called the_great_shield_key"
```
 
Navegamos para `/diningRoom/the_great_shield_key.html` e obtemos o **shield key flag**.
 
---
 
### /diningRoom2F/ — ROT13 → blue jewel
 
Source code da página contém:
 
```
Lbh trg gur oyhr trz ol chfuvat gur fgnghro gb gur ybjre sybbe...
```
 
Aplicando **ROT13**:
 
```
"You get the blue gem by pushing the status to the lower floor. Visit sapphire.html"
```
 
Navegamos para `/diningRoom/sapphire.html` e obtemos o **blue jewel flag**.
 
---
 
### Coletando os 4 Crests
 
Com os itens coletados, as salas restantes se abrem. Cada uma entrega um crest com encoding diferente. Ferramenta usada: **CyberChef**.
 
**Crest 1 — /tigerStatusRoom/**
Blue jewel desbloqueia a sala. Crest codificado duas vezes: **Base64 → Base32**.
 
```
Crest 1: RlRQIHVzZXI6IG
```
 
**Crest 2 — /galleryRoom/**
Examinando a nota da galeria. Crest codificado em **Base32 → Base58**.
 
```
Crest 2: h1bnRlciwgRlRQIHBh
```
 
**Crest 3 — /armorRoom/**
Shield key desbloqueia. Crest codificado três vezes: **Base64 → Binary → Hex → ASCII**.
 
```
Crest 3: c3M6IHlvdV9jYW50X2h
```
 
**Crest 4 — /attic/**
Shield key desbloqueia. Crest codificado em **Base58 → Hex**.
 
```
Crest 4: pZGVfZm9yZXZlcg==
```
 
---
 
### Combinando os 4 Crests → credenciais FTP
 
Concatenando os quatro crests:
 
```
RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==
```
 
Decodificando de **Base64**:
 
```bash
echo "RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==" | base64 -d
# FTP user: hunter, FTP pass: [censurado]
```
 
---
 
## exploitation
 
### FTP — steganografia nas imagens
 
```bash
ftp hunter@<TARGET_IP>
mget *
```
 
Arquivos baixados:
 
```
important.txt
001-key.jpg
002-key.jpg
003-key.jpg
helmet_key.txt.gpg
```
 
`important.txt` — nota de Barry para Jill mencionando um `/hidden_closet/` e que a helmet key está em algum arquivo.
 
As três imagens carregam dados ocultos via steganografia:
 
**001-key.jpg — steghide sem passphrase:**
 
```bash
steghide extract -sf 001-key.jpg
# extrai key-001.txt → cGxhbnQ0Ml9jYW
```
 
**002-key.jpg — strings:**
 
```bash
strings 002-key.jpg | grep -i comment
# Comment: 5fYmVfZGVzdHJveV8
```
 
**003-key.jpg — binwalk (steghide pediu passphrase):**
 
```bash
binwalk -e 003-key.jpg
# extrai key-003.txt → 3aXRoX3Zqb2x0
```
 
Concatenando as três partes e decodificando de **Base64**:
 
```bash
echo "cGxhbnQ0Ml9jYW5fYmVfZGVzdHJveV93aXRoX3Zqb2x0" | base64 -d
# resultado: [censurado]
```
 
Essa é a passphrase para descriptografar o arquivo GPG:
 
```bash
gpg helmet_key.txt.gpg
# passphrase: [censurado]
# resultado: helmet key flag
```
 
---
 
### /studyRoom/ + /hidden_closet/ → credenciais SSH
 
Com o **helmet key flag**, desbloqueamos o Study Room.
 
Dentro há um arquivo `doom.tar.gz` para download:
 
```bash
gunzip doom.tar.gz
tar -xf doom.tar
# extrai eagle_medal.txt → SSH user: umbrella_guest
```
 
No `/hidden_closet/`, usando o helmet key, encontramos o **MO Disk 1** com mensagem codificada em **Vigenere** (chave: albert):
 
```
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
→ albert weasker password: [censurado]
```
 
Ainda no hidden closet, o **wolf medal** revela:
 
```
SSH password: T_virus_rules
```
 
Credenciais SSH obtidas:
 
```
usuário: umbrella_[censurado]
senha:   [censurado]
```
 
---
 
### SSH — acesso inicial e pivô para weasker
 
```bash
ssh umbrella_guest@<TARGET_IP>
```
 
Dentro do home há um diretório oculto `.jailcell` com `chris.txt` — narrativa do CTF. No final do arquivo:
 
```
MO disk 2: albert
```
 
Verificando os outros usuários em `/home`: **hunter** e **weasker**.
 
Mudando para weasker com a senha obtida no MO Disk 1:
 
```bash
su weasker
# senha: [censurado]
```
 
---
 
## privilege escalation
 
### Enumeração pós-acesso
 
```bash
id
```
 
```
uid=1000(weasker) gid=1000(weasker) groups=1000(weasker),4(adm),24(cdrom),27(sudo)...
```
 
Grupo `sudo` presente — verificação imediata:
 
```bash
sudo -l
```
 
```
User weasker may run the following commands on umbrella_corp:
    (ALL : ALL) ALL
```
 
### Análise da vulnerabilidade
 
`(ALL : ALL) ALL` significa: qualquer comando, como qualquer usuário, sem restrição. Weasker tem controle administrativo total. Nenhum exploit adicional necessário — a misconfiguration é o caminho.
 
> **Vulnerability:** Sudo Privilege Misconfiguration  
> **MITRE ATT&CK:** T1548.003 — Abuse Elevation Control Mechanism: Sudo  
> **Severity:** Critical
 
### Execução
 
```bash
sudo su
```
 
```
root@umbrella_corp:~#
```
 
---
 
## flags
 
```bash
cat /root/root.txt
# flag: [censurado]
```
 
---
 
## attack chain
 
```
Nmap → 21 (FTP) · 22 (SSH) · 80 (HTTP)
          │
          ▼
/mansionmain/ → source code → /diningRoom/
          │
          ▼
emblem flag + Base64 → /teaRoom/
          │
          ▼
lockpick flag → /artRoom/ → mapa da mansão
          │
          ▼
/barRoom/ (lockpick) → Base32 → music sheet → gold emblem
          │
          ▼
emblem original no bar room slot → nome "rebecca"
          │
          ▼
gold emblem no diningRoom → Vigenere (key: rebecca) → shield key
          │
          ▼
/diningRoom2F/ → ROT13 → blue jewel
          │
          ▼
4 Crests (Base64·Base32 / Base32·Base58 / Base64·Bin·Hex / Base58·Hex)
          │
          ▼
Crests concatenados → Base64 → FTP: hunter / you_cant_hide_forever
          │
          ▼
FTP → 3 imagens → steghide + strings + binwalk → chave GPG
          │
          ▼
GPG decrypt → helmet key flag
          │
          ▼
/studyRoom/ → doom.tar.gz → SSH user: umbrella_guest
/hidden_closet/ → Vigenere autosolve → weasker password
                → wolf medal → SSH pass: T_virus_rules
          │
          ▼
SSH (umbrella_guest) → .jailcell/chris.txt → MO disk 2: albert
          │
          ▼
su weasker (stars_members_are_my_guinea_pig)
          │
          ▼
sudo -l → (ALL : ALL) ALL → sudo su → root
          │
          ▼
root.txt → flag capturada
```
 
---
 
## what I learned
 
Essa sala tem uma cadeia longa demais para ser resolvida sem anotações. O nome **"rebecca"** aparece logo no início e só se torna relevante como chave Vigenere muito depois — quem não anotou ficou preso. Isso é um comportamento real de reconhecimento: informações que parecem decorativas no início do engagement frequentemente são críticas no final.
 
Outro ponto relevante: as três técnicas de extração usadas nas imagens (steghide, strings, binwalk) serviram para contornar diferentes cenários — steghide pediu passphrase em uma das imagens, o que forçou o uso de binwalk como alternativa. Em pentest real, ter mais de uma ferramenta para o mesmo objetivo é essencial.
 
A escalação via sudo foi trivial depois de toda a complexidade da cadeia. Isso acontece: ambientes com configuração de sudo irrestrita para um usuário comprometido entregam root imediatamente, independente de qual foi o caminho de acesso inicial.
 
**Remediação:**
 
```bash
# Configuração insegura encontrada
weasker ALL=(ALL:ALL) ALL
 
# Correto — princípio do menor privilégio
weasker ALL=(root) NOPASSWD: /usr/bin/systemctl restart apache2
```
 
Revisar `/etc/sudoers` regularmente, remover usuários desnecessários do grupo `sudo` e nunca manter permissões temporárias sem prazo de expiração.
 
