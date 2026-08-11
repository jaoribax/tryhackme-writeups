# Whale Vault — TryHackMe

> **Dificuldade:** Médio  
> **Categoria:** Web Exploitation · Business Logic Flaw · Race Condition (TOCTOU)  
> **Data:** 2025-08-08  
> **Sala:** [https://tryhackme.com/room/whalevault](https://tryhackme.com/room/hh-towelonthesunbed-61271709)

---

## overview

Aplicação web com um sistema de recompensas diárias baseado em tokens. O objetivo é acumular 150 tokens (parâmetro `whaleThreshold`) para desbloquear a flag do Whale Vault. O problema: cada resgate concede apenas 50 tokens, e a lógica da aplicação trava o estado da conta após o primeiro uso. A solução não está em contornar a autenticação — está em explorar uma **Race Condition do tipo TOCTOU (Time-of-Check / Time-of-Use)** para disparar três resgates simultâneos antes que o servidor consiga fechar o acesso.

---

## reconnaissance

Acesso inicial à aplicação em `http://<TARGET_IP>:3000`. Criação de uma conta Guest para interação com a interface.

Configuração do proxy no navegador apontando para `127.0.0.1:8080` (Burp Suite).

Ao clicar em **Claim Daily Reward**, o Burp intercepta a requisição — revelando que o resgate é feito via **método POST**, com os dados trafegando no corpo da mensagem (não expostos na URL).

---

## enumeration

### Análise da lógica de negócio

Com o pacote POST retido no Burp, foi possível mapear a anatomia da requisição:

- A recompensa diária entrega **50 tokens** por resgate
- O `whaleThreshold` exige **150 tokens** para liberar a flag
- Após o primeiro resgate, a conta tem seu estado lógico **travado** — tentativas subsequentes retornam erro
- Resgates em massa são bloqueados por rate limiting (**HTTP 429 Too Many Requests**)

O ataque direto (repetição simples) não funciona. O vetor é outro: explorar a janela entre a verificação de elegibilidade e o bloqueio do estado — **TOCTOU**.

---

## exploitation

### Race Condition — Last-Byte Sync

**Por que o TOCTOU funciona aqui:**

A aplicação verifica se a conta é elegível (Time-of-Check) e só depois registra o uso e trava o estado (Time-of-Use). Se três requisições chegam ao servidor na mesma janela de tempo — antes que qualquer uma delas complete o ciclo de verificação + bloqueio — o banco de dados atesta elegibilidade para todas e libera 50 tokens para cada processo concorrente. Resultado: 150 tokens em um único ciclo.

**Setup no Burp Suite:**

1. Com a requisição POST interceptada, enviar para o **Repeater** (`Ctrl+R`)
2. Duplicar a aba da requisição até ter **exatamente 3 abas idênticas**
3. Selecionar as 3 abas → botão direito → **Add to new tab group** → nomear "Exploit Race"
4. No menu de envio do grupo, alterar a estratégia para **Send group in parallel (last-byte sync)**
5. Disparar o ataque

**Por que last-byte sync:**

O Burp pré-estabelece todas as conexões TCP e envia o conteúdo completo de todas as requisições simultaneamente — segurando apenas o último byte de cada uma. Ao soltar os três bytes finais ao mesmo tempo, as requisições chegam ao processador do servidor no mesmo microciclo, forçando concorrência real.

**Por que exatamente 3 abas:**

3 requisições × 50 tokens = 150 tokens. Usar mais abas geraria tráfego suficiente para acionar o rate limiting (HTTP 429) — o número mínimo necessário é também o número seguro.

**Resultado:**

As três abas retornam **HTTP 200 OK**. Desativando o intercept e atualizando a página na aplicação, o Whale Vault é desbloqueado.

> **Vulnerabilidade:** Race Condition — TOCTOU (CWE-362)  
> **Causa raiz:** Ausência de mecanismo atômico (lock ou transação com isolamento adequado) entre a verificação de elegibilidade e o registro do uso da recompensa.  
> **MITRE ATT&CK:** T1499.004 — Application or System Exploitation

---

## flags

```
Flag: [capturada no Whale Vault após acumular 150 tokens]
```

---

## attack chain

```
Acesso à aplicação → conta Guest criada
          │
          ▼
Burp Suite configurado como proxy (127.0.0.1:8080)
          │
          ▼
Claim Daily Reward → POST interceptado
          │
          ▼
Análise: 50 tokens/resgate · 150 necessários · estado travado após 1 uso
          │
          ▼
Ataque direto bloqueado por rate limiting (HTTP 429)
          │
          ▼
Vetor: TOCTOU — janela entre verificação e bloqueio
          │
          ▼
Burp Repeater → 3 abas idênticas → Tab Group "Exploit Race"
          │
          ▼
Send group in parallel (last-byte sync)
          │
          ▼
3x HTTP 200 OK → 150 tokens acumulados
          │
          ▼
Whale Vault desbloqueado → flag capturada
```

---

## what I learned

Essa sala demonstra um tipo de vulnerabilidade que não aparece em scanners automáticos e não tem um CVE atribuído — é uma **falha de lógica de negócio** explorada via timing preciso.

O conceito de TOCTOU é simples na teoria: existe uma janela entre "verificar se pode" e "registrar que usou". Se você consegue fazer múltiplas operações caberem nessa janela, o sistema autoriza todas antes de fechar o acesso. O desafio prático é sincronizar as requisições com precisão suficiente para que isso aconteça.

O **last-byte sync** do Burp resolve isso de forma elegante: ao pré-estabelecer as conexões e soltar todos os bytes finais simultaneamente, elimina-se o atraso de rede como variável. As requisições chegam ao servidor no mesmo microciclo — não aproximadamente ao mesmo tempo, mas literalmente ao mesmo tempo do ponto de vista do processador.

O detalhe do rate limiting também foi importante: usar mais de 3 abas teria ativado a mitigação do servidor. Entender o threshold do alvo antes de disparar o ataque foi o que tornou a exploração cirúrgica.

**Remediação:**

```sql
-- Operação deve ser atômica: verificar + registrar em uma única transação
BEGIN TRANSACTION;
  SELECT status FROM rewards WHERE user_id = ? FOR UPDATE;
  -- se elegível:
  UPDATE rewards SET status = 'used' WHERE user_id = ?;
  UPDATE wallets SET tokens = tokens + 50 WHERE user_id = ?;
COMMIT;
```

Sem o `FOR UPDATE` (ou equivalente no ORM), a linha não é bloqueada durante a transação — permitindo que múltiplas leituras concorrentes vejam o estado "elegível" antes que qualquer uma delas escreva "usado".
