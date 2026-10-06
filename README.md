# CTF Writeups

> Cada relatório documenta a análise de causa raiz de vulnerabilidades reais (OWASP Top 10), automação de exploração (PoC) e a proposta de remediação de código/infraestrutura.

---

### Autor & Contexto
* **Autor:** Marcos Vinicius
**Vínculo:** HawkSec (Networking) | Engenharia de Computação — UNIFEI

---

### Metodologia de Análise

Cada writeup presente neste repositório não se limita a obter a *flag*, mas segue a estrutura de **Engenharia de Segurança**:

1. **Reconhecimento & Mapeamento:** Identificação de superfície de ataque e inspeção de schemas/rotas.
2. **Análise de Causa Raiz:** Mapeamento do ponto exato de falha no código ou na lógica de negócio.
3. **Recomendações de Correção:** Possíveis correções dessas vulnerabilidades.

---

## Relatórios & Writeups

<details>
<summary><b>2026 (Clique para expandir)</b></summary>

### ☀️ Sunshine CTF 2026
* **[Tomorrow Mart (Web / GraphQL)](./2026/Sunshine-CTF/Web/Tomorrow-Mart)**
  * **Falha:** Introspecção exposta, `vendorTerminalSync` desprotegido e falha de lógica em cupons.
  * **Defesa:** Desativação de `__schema` em produção e validação server-side.
* **[Kidding (Web / JWT)](./2026/Sunshine-CTF/Web/Kidding)**
  * **Falha:** Path Traversal no parâmetro `kid` e dump de chave HMAC em erro.
  * **Defesa:** Validação de `kid` por allowlist e supressão de mensagens de erro detalhadas.
</details>

---

### Tech Stack & Ferramentas Utilizadas

* **Linguagens & Automação:** Python 3, C/C++, Bash
* **APIs & Protocolos:** GraphQL, JWT, REST, HTTP/2
* **Análise & Defesa:** Burp Suite, Semgrep (SAST), Python `requests`/`jwt`, Docker
* **Conceitos:** OWASP Top 10 API, Cryptographic Failures, Broken Object Level Authorization (BOLA/IDOR)

---

### Contato Técnico
* **LinkedIn:** [linkedin.com/in/marcmacedo](https://linkedin.com/in/marcmacedo)
* **Email:** marcosmacedo-cs@proton.me
