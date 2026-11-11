# Rezol Pay

Painel independente de pagamentos da Rezol. Hoje ele oferece uma fundação local de produto e uma simulação persistente; **não é uma instituição financeira, não processa cobranças e não movimenta fundos**.

## Relação com o shell desktop Rezol

A direção futura da Rezol é deixar o Electron e avaliar uma janela desktop
compilada com Rust ou tecnologia nativa equivalente, que apresente cada
projeto em uma aba isolada. Este painel continuará sendo aplicação e processo
independentes; não deve compartilhar origem, proxy, host, porta ou rotas com
ASCA ou ServerLabs. A janela não deve reescrever URLs nem converter a interface
do painel em rotas internas de outro servidor. O shell ainda não foi migrado
nem teve sua tecnologia escolhida.

## Executar localmente

Requisitos: Node.js 24 e pnpm 10.10.

```powershell
corepack pnpm install
corepack pnpm dev
```

O servidor fica limitado a `127.0.0.1`. Abra `http://127.0.0.1:3000`. O SQLite é criado automaticamente em `.data/rezol-pay.sqlite`, com dados demonstrativos fictícios. A pasta está excluída do Git. Para usar outro arquivo local, defina `REZOL_PAY_DB_PATH` antes de iniciar o processo.

Se usar uma porta ou URL diferente, defina `REZOL_PAY_BASE_URL` com a origem completa usada pelo navegador e pelos compradores (por exemplo, `http://127.0.0.1:3001`). A configuração é usada para montar links hospedados de checkout; sem ela, o sandbox deriva a origem de uma requisição de loopback. Não exponha este servidor à internet: a variável não transforma a demonstração em serviço seguro de produção.

```powershell
corepack pnpm test
corepack pnpm typecheck
corepack pnpm build
```

### Depurar no VS Code

Abra `Rezol.code-workspace` na raiz Rezol, selecione **Rezol Pay: debug full stack** e pressione F5. O perfil usa `127.0.0.1:3002`, habilita breakpoints do servidor na porta `9232`, abre o Chrome e grava em um SQLite de depuração separado (`%LOCALAPPDATA%\Rezol\RezolPay-Debug\rezol-pay.sqlite`). Use **Rezol Pay: servidor (sem navegador)** para não abrir o Chrome. Coloque breakpoints em rotas da API ou módulos do servidor. Para depurar a interface React, use os DevTools do navegador aberto. Se a porta 9232 estiver em uso, encerre o processo depurado anterior antes de iniciar outra sessão. Configuração baseada no [guia oficial de depuração do Next.js](https://nextjs.org/docs/app/guides/debugging).

Para iniciar o estúdio de voz pelo mesmo workspace principal, escolha **Rezol Voice Debug** na lista de configurações de execução e pressione F5. O perfil executa o clone em `apps/elevenlabs-clone/elevenlabs-clone-frontend`, inicia ou reutiliza o Piper local e abre `http://127.0.0.1:3011`. Os caminhos pressupõem que os projetos estejam lado a lado como na estrutura padrão de `E:\rezol`.

## O que funciona nesta versão

- Banco SQLite local com migração versionada, clientes fictícios, pagamentos, eventos imutáveis e trilha de auditoria.
- `GET /api/dashboard?period=7d|30d|90d`: séries de recebimento e indicadores agregados do banco.
- `GET /api/health`: readiness local, conectividade do SQLite e versão do schema; responde `503` se a consulta ao banco falhar.
- `POST /api/payments`: grava uma cobrança demonstrativa como pendente. Valida nome, e-mail, valor em centavos e meio de pagamento; requer `Idempotency-Key`.
- Repetir a mesma chave e o mesmo corpo devolve a cobrança original; reutilizar a chave com conteúdo diferente retorna conflito.
- `GET /integracoes`: portal local para criar e revogar chaves sandbox. O segredo `rz_test_...` aparece somente uma vez; o banco guarda seu hash e um prefixo mascarado. Chaves revogadas não autenticam chamadas.
- `POST /api/v1/checkout-sessions`: cria uma sessão hospedada sandbox. Requer `Authorization: Bearer <chave>` e `Idempotency-Key`; aceita `amount_cents` (100–100.000.000), `currency: "BRL"`, `payment_method` (`pix`, `card` ou `boleto`), `customer` (`name`, `email`), `description` opcional e `return_url` HTTP(S) opcional.
- `GET /api/v1/checkout-sessions/{id}`: consulta sessão com a mesma chave sandbox ativa.
- `GET /checkout/{id}`: checkout demonstrativo hospedado. O comprador pode clicar em “Simular pagamento aprovado”, que altera somente dados fictícios locais; não captura cartão, Pix ou boleto.
- Sessões são válidas por 30 minutos. Ao consultar ou tentar concluir uma sessão após o prazo, a sessão muda para `expired`, o pagamento pendente para `failed` e o banco registra evento/auditoria uma única vez.
- `GET /api/checkout/{id}` e `POST /api/checkout/{id}`: leitura pública mínima da página e ação de simulação protegida contra requisições cross-origin.
- A repetição da chave de idempotência com os mesmos dados devolve a sessão original (`200`); outra carga útil para a mesma chave retorna `409`. Uma chave ausente/inválida/revogada retorna `401`.
- Métricas: volume aprovado, ticket médio, taxa de aprovação, pendências, estornos, falhas, disputas, clientes e mix de meios de pagamento.
- Máquina de estados e contrato para o futuro adaptador de PSP, sem cliente de provedor nem chamadas financeiras reais.

Os dados iniciais usam e-mails reservados no domínio `.test`. As cobranças criadas pela interface permanecem no arquivo local até que ele seja removido deliberadamente. Não grave informações pessoais reais neste protótipo.

### Chamada de integração sandbox

Gere uma chave em **Integrações**, guarde o segredo no servidor da loja (por exemplo, na variável de ambiente `REZOL_PAY_API_KEY`) e faça a chamada do backend. Nunca inclua uma chave secreta no navegador ou repositório:

```bash
curl -X POST http://127.0.0.1:3000/api/v1/checkout-sessions \
  -H "Authorization: Bearer $REZOL_PAY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: pedido-123" \
  -d '{
    "amount_cents": 12990,
    "currency": "BRL",
    "payment_method": "pix",
    "description": "Pedido 123",
    "customer": { "name": "Ana Silva", "email": "ana@example.test" }
  }'
```

A resposta inclui `id`, `url`, `status` (`open`) e o estado inicial `pending` do pagamento. Redirecione o comprador para `url`; consulte a sessão com `GET /api/v1/checkout-sessions/{id}` para ver o estado persistido. Somente a UI de demonstração permite simular a conclusão. `return_url` é atualmente validada e persistida como metadado de sessão, mas não provoca redirecionamento automático.

O schema SQLite atual é a migração v2: além das tabelas-base, inclui `api_keys`, `checkout_sessions` e `webhook_deliveries` para registrar eventos demonstrativos. As chaves e sessões pertencem ao tenant sintético `rezol-demo`; não existe isolamento/autorização multi-tenant para comerciantes reais.

## Arquitetura proposta para a gateway

### Fronteira de negócio

A decisão inicial é começar como **camada técnica sobre um PSP/adquirente autorizado**, sem custodiar, liquidar nem prometer disponibilidade de saldo. O provedor segue responsável pelo processamento e pela liquidação. O provedor ainda não foi escolhido. O onboarding de comerciantes, a contratação do PSP e a análise jurídica/regulatória são pré-requisitos para qualquer produção.

### Componentes

1. **Dashboard / BFF**: experiência do comerciante; API autenticada e autorizada por tenant.
2. **API de pagamentos**: intents/sessões, consultas, cancelamentos e pedidos de estorno; valores inteiros em centavos, moeda explícita, chave de idempotência e máquina de estados.
3. **Adaptador de PSP**: interface interna comum com implementações por provedor, credenciais apenas no servidor e ambiente sandbox/live segregado. `lib/gateway/provider.ts` é apenas o contrato tipado; a implementação aguarda a seleção do provedor.
4. **Checkout hospedado/tokenizado**: coleta de cartão dentro da experiência fornecida pelo PSP; a Rezol não deve receber nem armazenar PAN/CVV.
5. **Webhooks**: validar assinatura e janela temporal conforme o PSP, deduplicar evento, persistir inbox, responder rapidamente e processar de forma assíncrona/repetível.
6. **Persistência de produção**: PostgreSQL gerenciado, migrações controladas, isolamento por `merchant_id`, transações e backup com teste de restauração. SQLite é somente para este protótipo de uma instância local.
7. **Conciliação e relatórios**: importar liquidações e tarifas do PSP, reconciliar pagamentos/estornos/disputas e separar volume bruto, taxa, líquido previsto e valor efetivamente liquidado.
8. **Operação**: logs sem dados de cartão/segredos, métricas e alertas de falhas, fila parada, atraso de webhook, conciliação divergente e disponibilidade do PSP.

### Modelo de domínio a evoluir

`merchants`, `users`, `memberships/roles`, `customers`, `payment_intents`, `payment_attempts`, `provider_events` (inbox e deduplicação), `refunds`, `disputes`, `settlements`, `ledger_entries` de dupla entrada, `idempotency_records`, `audit_log`, `api_keys` (segredos armazenados apenas como hashes), `webhook_endpoints`, `webhook_deliveries` e `reconciliation_runs`.

O SQLite atual inclui apenas `merchants`, `customers`, `payments`, `payment_events` e `audit_log`, com `provider='demo'`; não representa um livro-caixa nem um saldo sacável. Um sistema real deve separar estados da tentativa do PSP e registros contábeis append-only. Saldo não deve ser inferido apenas da soma de cobranças pagas.

### Fluxo seguro de uma cobrança real

1. O servidor autenticado valida tenant, valor e referência comercial; exige `Idempotency-Key`.
2. Cria intent local como `created` e registra a tentativa de forma transacional.
3. O adaptador cria sessão no sandbox/PSP; cliente paga no checkout hospedado ou tokenizado do provedor.
4. O navegador mostra somente estado informativo. A confirmação financeira vem da consulta ao PSP ou de webhook autenticado, não do redirecionamento do navegador.
5. O webhook é validado, deduplicado e enfileirado; o consumidor aplica transições permitidas e registra evento/auditoria.
6. Jobs conciliam eventos, taxas, reembolsos, disputas e liquidações; a UI separa autorizado, capturado, recebido e liquidado.

### Endpoints sugeridos

- `POST /v1/checkout-sessions` — criar intent/sessão idempotente.
- `GET /v1/payments/{id}` — consultar estado canônico.
- `POST /v1/payments/{id}/capture` e `/cancel` — somente se o PSP e o fluxo suportarem.
- `POST /v1/payments/{id}/refunds` — pedido idempotente de estorno.
- `POST /v1/webhooks/{provider}` — recepção autenticada de eventos.
- Rotas privadas para conciliação e relatórios; não expor operações financeiras no endpoint público do dashboard.

### Métricas úteis no produto

- **Operação**: volume bruto aprovado; líquido liquidado e previsto; ticket médio; contagem por estado; taxa de aprovação por tentativa, PSP, método, emissor/BIN quando permitido e faixa de valor; abandono entre checkout iniciado e pago; taxa de pendência e tempo até confirmação.
- **Risco e suporte**: falhas por código normalizado, estornos e taxa de estorno, disputas/chargebacks e taxa sobre vendas, reembolsos em aberto, eventos/webhooks atrasados ou repetidos.
- **Financeiro**: tarifas do provedor, custo efetivo, valor a liquidar por data, divergências de conciliação e prazo médio de liquidação.
- **Produto**: novos clientes, recorrência e receita por período, com filtros de data e comparação equivalente ao período anterior.

Os indicadores mostrados hoje são somente o conjunto calculável a partir da amostra local: volume bruto e aprovação incluem pagamentos `succeeded`, `refunded` e `disputed`; a taxa de aprovação exclui pendências do denominador. O valor estornado é mostrado separadamente. A máquina simplificada da demonstração não representa o livro financeiro, liquidação ou estado de captura do PSP.

## Segurança, conformidade e produção

- Não receber PAN, CVV, PIN, senha bancária nem guardar dados de autenticação sensíveis. Checkout hospedado reduz escopo, mas **não elimina automaticamente** obrigações PCI DSS; o escopo depende da arquitetura e deve ser confirmado com o PSP/adquirente e um assessor qualificado.
- TLS, gestão/rotação de segredos, MFA, RBAC, segregação entre tenants e ambientes, proteção contra replay, idempotência e rate limits fazem parte do desenho de produção.
- LGPD: definir finalidade/base legal, minimização, retenção, direitos do titular, contratos de operador/controlador, resposta a incidente e avaliação do fluxo internacional com assessoria adequada.
- Confirmar com advogado e o Banco Central a classificação da atividade, arranjos, papel contratual, regras do Pix/participação indireta e eventuais autorizações aplicáveis. Usar um PSP autorizado não dispensa análise do modelo concreto.
- Habilitar live somente após homologação do PSP, testes de webhook e conciliação, análise de segurança, política de chargeback/estorno, suporte operacional e aprovação jurídica.

## Pesquisa de referência (fontes oficiais consultadas em 23/10/2026)

- [PCI Security Standards Council — PCI DSS](https://www.pcisecuritystandards.org/standards/pci-dss/): escopo do padrão inclui entidades que armazenam, processam ou transmitem dados de conta e entidades que podem afetar o ambiente de dados do cartão.
- [Stripe — Checkout](https://docs.stripe.com/payments/checkout): opções hospedadas/integradas e recomendação de sessão de checkout para reduzir esforço e manutenção.
- [Mercado Pago — Checkout API / Orders](https://www.mercadopago.com.br/developers/pt/docs/checkout-api-orders/overview): checkout brasileiro, meios locais e configuração de notificações e testes de integração.
- [Banco Central do Brasil — Pix](https://www.bcb.gov.br/estabilidadefinanceira/pix): referência institucional sobre o arranjo Pix; regras e participação devem ser confirmadas para o modelo escolhido.
- [Lei 13.709/2018 — LGPD](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm): texto legal para revisão com assessoria.

As fontes orientam o desenho, não constituem parecer jurídico, certificação PCI nem aprovação do Banco Central. O uso de um PSP ainda não selecionado é deliberadamente simulado.
