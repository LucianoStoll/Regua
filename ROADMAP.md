# Plano de aprendizado: SaaS de lembretes de cobrança por e-mail

## Contexto
O usuário (estudante de SI, assistente financeiro) tem um script desktop (`Script.pyw`) que lê uma planilha Excel (aba `DocReceberVencimento80`: N° DOCUMENTO, COD CLIENTE, CLIENTE, DATA VENC, VALOR, EMAIL) e manda um lembrete de vencimento em HTML fixo, usando a planilha como fonte direta dos dados.

O objetivo é transformar isso em um **SaaS multi-empresa na Cloudflare (plano gratuito)**, como projeto de portfólio:
- **O banco de dados (D1) é a fonte principal.** O `.xlsx` serve **apenas para importar**: ao enviar o arquivo, o sistema lê, valida, **verifica duplicidades** e grava no banco. Todo envio de e-mail usa os dados do banco.
- Cada empresa personaliza a marca (logo, cor, contatos) e a mensagem (template com variáveis).
- MVP: base essencial + régua automática (Cron) + descadastro (LGPD). Stack: **TypeScript** (Hono no Worker + React). Repo novo no GitHub do usuário.
- **O usuário quer aprender: ele escreve o código.** Eu não entrego código pronto do projeto. Meu papel é de mentor: explico conceitos, proponho a próxima tarefa, dou dicas e pseudocódigo, mostro exemplos pequenos e genéricos de um conceito (fora do projeto), reviso o código que ele subir, ajudo a interpretar erros e escrevo perguntas e casos de teste para ele resolver.

Regras: só dados fictícios; nenhum segredo no código. A senha de app do Gmail exposta no script atual deve ser revogada.

## Arquitetura alvo
- Um Worker (Hono) com a API em `/api/*` + o frontend React (Vite + `@cloudflare/vite-plugin`, Workers Static Assets)
- **D1** (SQLite) com migrations · **R2** para os logos · **Cron Trigger** diário para a régua
- E-mail via API (Resend, plano gratuito) atrás de uma interface `EmailSender`; chave em `wrangler secret`
- Auth própria: senha com PBKDF2 (WebCrypto) + sessão em cookie httpOnly; toda query filtrada por `tenant_id`

## Modelo de dados (o usuário desenha; esta é a referência para a revisão)
Normalizado: o cliente deixa de se repetir em cada linha, como acontece na planilha.
- `tenants`, `users`, `sessions`
- `clientes` (tenant_id, cod_cliente, nome, email) com **UNIQUE(tenant_id, cod_cliente)**
- `titulos` (tenant_id, cliente_id, documento, vencimento DATE, valor_centavos INTEGER, status aberto|pago|cancelado) com **UNIQUE(tenant_id, documento)**
- `importacoes` (arquivo, total, inseridos, atualizados, ignorados, erros, data) + `importacao_erros` (linha, motivo)
- `templates`, `regras_lembrete` (dias_offset), `envios` com **UNIQUE(titulo_id, regra_id)** (o mesmo lembrete nunca sai duas vezes), `descadastros`

## Fluxo de importação (o coração do pedido)
1. A tela envia o `.xlsx` (multipart) para `POST /api/importacoes`. **O servidor** lê com SheetJS.
2. Mapeamento das colunas da planilha para os campos do sistema (salvo por empresa e reutilizado).
3. Normalização: valor → centavos inteiros; data → `AAAA-MM-DD`; e-mail em minúsculas e sem espaços; documento sem espaços extras.
4. Validação por linha (Zod): e-mail válido, valor > 0, data válida. Linhas inválidas vão para o relatório, sem derrubar a importação.
5. **Duplicidade em 3 níveis**:
   - dentro do próprio arquivo (o mesmo documento aparece 2×);
   - contra o banco: o cliente faz upsert por `cod_cliente` (atualiza nome e e-mail se mudaram); o título com o mesmo documento é **ignorado** se estiver idêntico e **atualizado** se o valor ou o vencimento mudou (só se ainda estiver `aberto`);
   - garantia final no banco: as constraints `UNIQUE` + `INSERT ... ON CONFLICT`.
6. Tudo em `db.batch()` (transação). Resposta: inseridos / atualizados / ignorados / erros, salva em `importacoes`.
7. Opcional: modo "pré-visualizar" mostra o resumo antes de confirmar.

## Etapas (cada uma: o que aprende → o que constrói → pronto quando)
0. **Fundamentos**: TypeScript básico, `async/await`, HTTP/REST, Git. → Revogar a senha do Gmail; criar o repo. → Pronto quando o "hello world" do Worker está no ar com `wrangler deploy`.
1. **Modelagem de dados**: normalização, chaves, UNIQUE, índices. → Desenhar as tabelas e escrever a `0001_init.sql`. → Pronto quando eu revisar o diagrama/SQL e a migration rodar no D1 local.
2. **API com Hono + D1**: rotas, validação com Zod, prepared statements. → CRUD de clientes e títulos. → Pronto quando der para testar as rotas com curl/Thunder Client.
3. **Importação de xlsx** (o fluxo acima). → Pronto quando importar a mesma planilha 2× resultar em "0 inseridos, N ignorados", e uma planilha com o valor alterado em "atualizados". Eu forneço planilhas de teste fictícias com casos difíceis (duplicata interna, e-mail inválido, data em formato estranho).
4. **Testes automatizados**: Vitest + `@cloudflare/vitest-pool-workers`. → Testes da importação e da formatação BR (`Intl.NumberFormat`, que corrige o bug "R$ 9903.04"). → Pronto quando o CI no GitHub Actions estiver verde.
5. **Auth e multi-empresa**: hash de senha, sessão, cookie, isolamento por tenant. → Pronto quando o teste "empresa A não vê dados da B" passar.
6. **Frontend React**: telas de login, configurações (logo no R2, cor, contatos), importação com relatório, lista de títulos.
7. **Templates de e-mail**: variáveis `{{cliente}}` etc., escape de HTML (segurança), layout de e-mail com tabelas, pré-visualização.
8. **Envio**: integração com a API do Resend, `envios` como log, envio manual dos selecionados, link de descadastro + header `List-Unsubscribe`.
9. **Régua com Cron**: `scheduled()`, idempotência via UNIQUE em `envios`, limite diário. Testar com `wrangler dev --test-scheduled`.
10. **Portfólio**: seed fictício, README (problema real que motivou, prints, diagrama, como rodar), deploy com conta demo.

## Como vamos trabalhar
- A cada etapa, eu explico o conceito e passo a tarefa com critérios de "pronto".
- O usuário escreve e faz push. Eu leio o repo (anexado nesta sessão), reviso e aponto melhorias com o porquê.
- Quando travar: primeiro uma dica, depois pseudocódigo, e só então um exemplo genérico isolado. Código do projeto, só se ele pedir explicitamente.
- O plano fica salvo no repo como `ROADMAP.md`. Eu ajudo a escrever as issues do GitHub, uma por etapa.

## Verificação
Cada etapa tem o seu "pronto quando". No geral: testes Vitest passando no CI; importação idempotente comprovada; fluxo completo (cadastro → importar → enviar para `delivered@resend.dev`) funcionando em `npm run dev`; deploy ativo.

## Pendências
- Nome do repositório (`usuario/repo`) e do produto.
- Nível atual em JS/TS (para decidir quanto tempo dar à etapa 0).