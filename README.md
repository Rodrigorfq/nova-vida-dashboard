# Painel Financeiro — Igreja

Sistema web para a gestão financeira de uma igreja local: fechamento mensal, entradas semanais, despesas com comprovantes, folha de pagamento e conferência entre o que foi lançado e o saldo real do banco. Funciona para vários anos (2026, 2027, …) e para mais de uma pessoa, cada uma com um papel diferente.

Começou como uma planilha do Google Sheets e evoluiu para um sistema com banco de dados, controle de acesso e histórico de alterações. As planilhas antigas ficam apenas como arquivo; **o painel é o único lugar de lançamento**.

> Este repositório não contém dados reais da igreja nem chaves secretas. O único segredo embutido no HTML é a chave *publishable* do Supabase, que foi feita para ficar no navegador e só dá acesso ao que as políticas de segurança (RLS) permitem.

---

## O que o sistema faz

| Área | O que resolve |
|---|---|
| **Início** | Escolha do ano; resumo; atalho para abrir o ano novo a partir de setembro. |
| **Visão do ano** | Entradas, saídas, saldo e gráficos (Chart.js) por mês e por categoria. |
| **Entradas** | Lançamento semanal (dízimos, ofertas, etc.), com total calculado pela soma dos lançamentos. |
| **Despesas** | Lançamento com categoria padronizada, forma de pagamento, responsável e comprovante. |
| **Folha** | Funcionários, dias trabalhados, vales, parâmetros por ano e fechamento do mês. |
| **Fechamento** | Saldo real do banco informado no fim do mês; o mês fica "fechado". |
| **Conferência** | Compara o saldo calculado com o saldo real e mostra a diferença mês a mês. |
| **Comprovantes** | Foto/PDF enviados ao Google Drive, organizados automaticamente por ano e mês. |
| **Equipe** | Admin cadastra pessoas e define o papel de cada uma. |
| **Exportação** | Planilhas Excel (ano, mês, folha) geradas no navegador. |

### Papéis

- **Administrador** — faz tudo: lança, corrige, exclui, fecha mês, abre ano, gerencia a equipe.
- **Lançador** — lança e corrige despesas e entradas de meses **abertos**. Não exclui, não altera mês fechado, não gerencia equipe.
- **Leitura** — só consulta.

---

## Arquitetura

```
 Navegador (index.html estático, GitHub Pages, Chart.js)
        │  HTTPS + JWT do usuário logado
        ▼
 Supabase
   ├─ Auth (e-mail/senha)
   ├─ PostgreSQL  ← tabelas, views, RLS, funções (RPC) com security definer
   └─ API REST (PostgREST)

 Google Apps Script (apenas para o Google Drive)
   └─ Comprovantes.gs: valida o token do usuário no Supabase,
      guarda o arquivo na pasta  ANO / NN-MÊS  e devolve o link.
```

Decisões principais:

1. **Sem servidor próprio.** O front-end é um único arquivo estático; o banco e a autenticação são do Supabase.
2. **Escrita só por funções (RPC).** O navegador nunca faz `INSERT`/`UPDATE` direto nas tabelas. Toda gravação passa por funções `security definer` que checam papel, mês fechado, normalizam texto e gravam histórico.
3. **Histórico de tudo.** Tabelas `historico_*` guardam quem mudou o quê e quando (visíveis só ao admin).
4. **Exclusão lógica.** Registros apagados recebem `excluido_em`; nada some de verdade.
5. **Multi-igreja pronto.** Toda tabela tem `igreja_id`; as políticas usam `igreja_do_usuario()`.
6. **Multi-ano.** Anos aparecem a partir dos próprios dados; "abrir ano novo" cria o ano no banco e as pastas no Drive de uma vez (operação repetível sem duplicar).

---

## Segurança

- **RLS ligado em todas as tabelas.** Cada pessoa só enxerga a igreja a que pertence e só se estiver em `usuarios_autorizados`.
- **A tabela `usuarios_autorizados` devolve só a linha do próprio usuário.** A lista da equipe vem de uma função específica (`listar_equipe()`), restrita a administradores.
- **Funções de permissão:** `usuario_eh_admin()`, `usuario_pode_lancar()`, `igreja_do_usuario()`.
- **Mês fechado:** `exigir_mes_aberto` bloqueia o lançador (mensagem clara, código 42501); só o admin altera.
- **Chaves:** a chave secreta/`service_role` nunca vai para o navegador nem para o GitHub. Ela existe apenas nas propriedades do script do Apps Script, quando necessário.
- **Sem confiança no front-end:** esconder um botão é só conveniência; a regra real está no banco.
- **Texto vindo do banco ou do Drive não vira HTML.** Categoria, descrição, observação, responsável, nomes de funcionário, mensagens de erro e nomes de arquivo passam por `escHtml()` antes de entrar na página; links de comprovante só são aceitos se começarem com `http://` ou `https://` (`urlSegura()`); os botões dos comprovantes pendentes não levam dados dentro do `onclick`. Há testes automáticos que gravam `<b>`/`<img onerror>` nesses campos e conferem que aparecem como texto.
- **Bibliotecas de terceiros:** o Chart.js é carregado com verificação de integridade (SRI) e `crossorigin="anonymous"`; se o arquivo da CDN for alterado, o navegador recusa.
- **Sessão:** renovação automática do token, uma única vez mesmo com várias chamadas simultâneas; sessão revogada volta ao login.

### Limites conhecidos (documentados de propósito)

- O token da sessão fica no `localStorage` do navegador (padrão de quem usa o Auth do Supabase direto de uma página estática). Mitigações: tokens de vida curta, renovação controlada, e a regra real de acesso ficar no banco (RLS + RPC), não na tela.
- Ainda não há *Content-Security-Policy*: o GitHub Pages não permite cabeçalhos próprios, e o painel usa scripts e estilos embutidos. O escape de texto e o SRI reduzem o risco, mas não o substituem.
- O ExcelJS (exportação) é carregado sob demanda do cdnjs, ainda sem SRI.
- Esconder valores na tela é só conveniência visual (desfoque), não é proteção de dados.

---

## Estrutura dos arquivos

| Arquivo | Função |
|---|---|
| `index.html` | Todo o painel (HTML + CSS + JS). Publicado no GitHub Pages. |
| `schema.sql` | Esquema completo e atual do banco (instalação do zero). |
| `migracao_*.sql` | Passos incrementais para quem já tem uma versão anterior do banco. |
| `Comprovantes.gs` | Apps Script: upload de comprovantes, pastas por ano/mês, "abrir ano". |
| `Importador.gs` e arquivos `.gs` de apoio | Importação histórica das planilhas antigas (uso único na migração). |
| `capturar.html` | Página simples para fotografar/enviar comprovantes pelo celular. |
| `Fechamentos_modelo.xlsx` | Modelo usado no início do projeto (referência). |

---

## Instalação (do zero)

1. **Supabase:** crie o projeto → *SQL Editor* → rode `schema.sql` inteiro.
2. **Primeiro administrador:** crie o usuário em *Authentication → Users* e insira a linha dele em `usuarios_autorizados` com papel `admin` e a `igreja_id` da igreja.
3. **Front-end:** em `index.html`, ajuste `SUPABASE_URL` e `SUPABASE_KEY` (chave **publishable**) e `URL_COMPROV` (endereço do Apps Script). Publique no GitHub Pages.
4. **Comprovantes:** crie o projeto no Apps Script com `Comprovantes.gs`, configure as propriedades do script (URL do Supabase e ID da pasta raiz do Drive), publique como *Web app* e use a URL em `URL_COMPROV`.
5. **Ano:** no painel, o admin abre o ano; o Drive ganha as pastas `ANO/NN-MÊS` e a pasta `PENDENTES` na raiz.

### Atualizando uma instalação existente

Rode **na ordem** apenas os `migracao_*.sql` que ainda não rodou:

1. `migracao_despesas_painel.sql`
2. `migracao_entradas_painel.sql`
3. `migracao_fechamento_painel.sql`
4. `migracao_vinculo_funcionarios.sql`
5. `migracao_padronizar_despesas.sql`
6. `migracao_folha_painel.sql`
7. `migracao_lancador.sql`

Todas são idempotentes (podem ser repetidas sem estragar nada). Depois publique o `index.html` (e dê *Ctrl+F5*) e, se um `.gs` mudou, use *Implantar → Gerenciar implantações → Nova versão* no Apps Script.

---

## Testes

A suíte foi escrita junto com o sistema e roda sem tocar em dados reais:

- **Interface (Playwright):** ~250 verificações com um Supabase simulado (login, sessão, anos, entradas, despesas, folha, papéis, Equipe, abrir ano, exportação, celular, escape de texto).
- **Banco (PostgreSQL embutido):** ~44 verificações das regras reais — papéis, mês fechado, histórico, exclusão, RLS.
- **Segurança:** verificações de que o front-end não contém chave secreta e de que nenhuma tabela aceita escrita direta.
- **Google Drive:** simulação do Apps Script para pastas por ano/mês, PENDENTES e idempotência.

---

## Tecnologias

PostgreSQL · Supabase (Auth, PostgREST, RLS) · JavaScript puro · Chart.js · Google Apps Script · Google Drive · GitHub Pages · Playwright.

## Licença

Projeto interno de uma igreja local, publicado como portfólio. Os dados reais não fazem parte do repositório.
