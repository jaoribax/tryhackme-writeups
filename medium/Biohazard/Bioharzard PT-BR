# Biohazard — TryHackMe

> **Difficulty:** Medium  
> **Category:** Web Enumeration · Steganography · Cryptography · Privilege Escalation  
> **Room:** Biohazard  
> **Platform:** TryHackMe

---

# Overview

A sala **Biohazard** do TryHackMe utiliza uma temática baseada em Resident Evil para simular uma invasão em uma mansão.

O objetivo é realizar a enumeração inicial, descobrir caminhos escondidos na aplicação web, resolver uma sequência de desafios envolvendo encoding/criptografia, obter acesso ao FTP através de credenciais descobertas, utilizar esteganografia para recuperar credenciais SSH e finalizar realizando uma escalação de privilégio até root.

A máquina não depende de exploits complexos de serviços. O foco principal é:

- Enumeração web
- Análise de código fonte
- Reconhecimento de padrões de encoding
- Steganografia
- Gestão de informações durante um pentest
- Enumeração pós-exploração
- Privilege escalation via má configuração

---

# Reconnaissance

O primeiro passo foi realizar uma enumeração dos serviços disponíveis.

```bash
nmap -sC -sV -p21,22,80 -T4 <TARGET_IP>
```

Resultado:

| Porta | Serviço | Versão |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 22/tcp | SSH | OpenSSH 7.6p1 |
| 80/tcp | HTTP | Apache |

Inicialmente temos três possíveis vetores:

- FTP
- SSH
- Aplicação Web

Como não possuímos credenciais, o foco inicial foi a aplicação HTTP.

---

# Web Enumeration

## /mansionmain/

A página inicial da aplicação apresenta uma referência à mansão.

Ao analisar o código fonte:

```html
<!-- It is in the /diningRoom/ -->
```

Foi identificado o primeiro caminho oculto:

```
/diningRoom/
```

---

# Dining Room

Acessando:

```
http://<TARGET_IP>/diningRoom/
```

Encontramos:

- Emblem flag
- Um texto cifrado
- Uma referência ao usuário:

```
rebecca
```

Esse nome é importante posteriormente.

Também encontramos um hint codificado:

```
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
```

Realizando a decodificação Base64:

```bash
echo "SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=" | base64 -d
```

Resultado:

```
How about the /teaRoom/
```

Novo caminho descoberto:

```
/teaRoom/
```

---

# Tea Room

Na Tea Room encontramos:

- Lockpick flag
- Nova indicação para outra sala

A chave encontrada permite avançar para:

```
/artRoom/
```

---

# Art Room — Map Enumeration

A Art Room contém um mapa da mansão.

Esse mapa revela diversos diretórios:

```
/diningRoom/
/teaRoom/
/artRoom/
/barRoom/
/diningRoom2F/
/tigerStatusRoom/
/galleryRoom/
/studyRoom/
/armorRoom/
/attic/
```

Neste ponto foi possível realizar uma enumeração organizada das salas restantes.

---

# Bar Room

A entrada da Bar Room exige o uso da lockpick encontrada anteriormente.

Dentro dela encontramos uma string codificada.

Identificando o padrão:

```
Base32
```

Decodificação:

```bash
echo "<STRING>" | base32 -d
```

Resultado:

```
Music Sheet
```

A music sheet permite acessar o puzzle do piano.

Ao completar o puzzle:

```
Gold Emblem
```

é obtido.

---

# Gold Emblem

O Gold Emblem deve ser utilizado no slot encontrado anteriormente no Dining Room.

Após inserir o emblema recebemos:

- Blue Gem
- Uma mensagem cifrada

A cifra utilizada era:

```
ROT13
```

Decodificação:

```bash
echo "<TEXT>" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

---

# Obtendo os Crests

A mansão possui quatro crests necessários para continuar.

Cada um utiliza um método diferente de encoding.

---

## Tiger Status Room

Método:

```
Base64
```

---

## Gallery Room

Método:

```
Base58
```

---

## Armor Room

Método:

```
Base64
→ Binary
→ Hex
→ ASCII
```

---

## Attic

Método:

```
Base32
```

---

# Montando os Crests

Após coletar os quatro valores, eles devem ser concatenados.

O resultado é novamente um Base64:

```bash
echo "<CRESTS>" | base64 -d
```

Resultado:

Credenciais FTP.

```
username: hunter
password: <obtida>
```

---

# FTP Enumeration

Com as credenciais:

```bash
ftp hunter@<TARGET_IP>
```

Arquivos encontrados:

```
important.txt
maps.jpg
albert.jpg
```

O arquivo `important.txt` indicava a existência de informações escondidas.

Os arquivos de imagem foram analisados utilizando esteganografia.

---

# Steganography

Ferramenta utilizada:

```bash
steghide
```

Extração:

```bash
steghide extract -sf maps.jpg
```

e:

```bash
steghide extract -sf albert.jpg
```

O arquivo `albert.jpg` solicitava uma passphrase.

Durante a enumeração inicial havia sido encontrado o nome:

```
rebecca
```

Utilizando:

```
rebecca
```

como senha foi possível extrair as informações escondidas.

Resultado:

Credenciais SSH.

```
username: weasker
password: <obtida>
```

---

# Initial Access

Acesso via SSH:

```bash
ssh weasker@<TARGET_IP>
```

Shell obtido:

```
weasker@umbrella_corp:~$
```

---

# Privilege Escalation

Primeiro passo:

```bash
id
```

Resultado:

```
uid=1000(weasker)
gid=1000(weasker)

groups:
sudo
adm
cdrom
dip
plugdev
lpadmin
sambashare
```

O usuário pertence ao grupo:

```
sudo
```

Então verificamos permissões:

```bash
sudo -l
```

Resultado:

```
User weasker may run the following commands on umbrella_corp:

(ALL : ALL) ALL
```

---

# Vulnerability Analysis

A configuração encontrada permite que o usuário execute qualquer comando como qualquer usuário.

Na prática:

```
weasker → qualquer comando → root
```

Essa é uma vulnerabilidade de configuração do sudo.

Classificação:

```
Sudo Privilege Misconfiguration
```

MITRE ATT&CK:

```
T1548.003
Abuse Elevation Control Mechanism: Sudo
```

Impacto:

```
Critical
```

---

# Root Access

Escalação:

```bash
sudo su -
```

Resultado:

```
root@umbrella_corp:~#
```

A flag final estava disponível:

```bash
cat /root/root.txt
```

Resultado:

```
flag:
3c5794a00dc56c35f2bf096571edf3bf
```

---

# Attack Chain

```
Nmap
 |
 |-- FTP
 |-- SSH
 |-- HTTP
        |
        v
/mansionmain/
        |
        v
/diningRoom/
        |
        v
Base64 Hint
        |
        v
/teaRoom/
        |
        v
/artRoom/
        |
        v
Mapa da mansão
        |
        v
Puzzles + Encodings
        |
        v
4 Crests
        |
        v
Credenciais FTP
        |
        v
FTP Enumeration
        |
        v
Steghide
        |
        v
SSH Credentials
        |
        v
SSH Access
        |
        v
sudo -l
        |
        v
sudo su -
        |
        v
Root Flag
```

---

# Lessons Learned

## 1. Informações aparentemente inúteis podem ser essenciais

O nome:

```
rebecca
```

aparece muito cedo na enumeração.

Naquele momento parece apenas uma informação narrativa, porém posteriormente se torna a chave da esteganografia.

Durante um pentest real, qualquer informação coletada deve ser registrada.

---

## 2. Encoding não é criptografia

Durante a máquina foram utilizados:

- Base32
- Base58
- Base64

Esses formatos apenas representam dados de outra maneira.

Eles não fornecem segurança.

---

## 3. A complexidade do ataque não significa que o último passo será complexo

Após uma longa cadeia de enumeração, a escalação final foi extremamente simples.

A vulnerabilidade:

```
(ALL : ALL) ALL
```

entrega privilégios administrativos completos.

---

# Remediation

A configuração insegura:

```
weasker ALL=(ALL:ALL) ALL
```

deve ser evitada.

Aplicar princípio do menor privilégio:

Exemplo:

```
weasker ALL=(root) /usr/bin/systemctl restart apache2
```

Recomendações:

- Revisar regularmente `/etc/sudoers`
- Remover usuários desnecessários do grupo sudo
- Aplicar Least Privilege
- Monitorar alterações administrativas

---

# Final Result

Privilege escalation concluída.

Root obtido.

Flag:

```
3c5794a00dc56c35f2bf096571edf3bf
```
