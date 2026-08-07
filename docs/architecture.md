# Arquitetura e tenancy

## Comece pelo limite do tenant

Um tenant pode representar uma empresa, workspace, igreja, escola ou cliente. Escreva essa definição antes de escolher bibliotecas.

O contexto mínimo normalmente inclui:

- identificador imutável;
- status de acesso;
- plano e limites aplicáveis;
- timezone e locale;
- configuração de domínio;
- chaves de cache e storage com namespace.

## Estratégias de dados

| Estratégia | Vantagem | Custo |
|---|---|---|
| Banco compartilhado com `tenant_id` | Operação simples e barata | Exige disciplina em todas as consultas |
| Schema por tenant | Isolamento lógico mais forte | Migrações e conexões mais complexas |
| Banco por tenant | Isolamento e restauração granular | Alto custo operacional |

Para muitos MVPs, banco compartilhado é suficiente se constraints, policies, testes e escopos impedirem consultas sem tenant.

## Contexto em processos assíncronos

Jobs devem carregar o identificador do tenant, reconstruir o contexto no worker e rejeitar tenants suspensos. Nunca dependa da sessão HTTP original.

## Billing como máquina de estados

Modele estados como `trialing`, `active`, `past_due`, `canceled` e `suspended`. Webhooks atualizam esse estado de forma idempotente; a autorização consulta o estado local em vez de chamar o gateway em cada request.

## Decisões registradas

Use ADRs curtos para escolhas difíceis:

1. contexto e problema;
2. decisão;
3. alternativas descartadas;
4. consequências e gatilho de revisão.
