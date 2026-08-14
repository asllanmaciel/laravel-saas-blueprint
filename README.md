# Laravel SaaS Blueprint

[![Release](https://img.shields.io/github/v/release/asllanmaciel/laravel-saas-blueprint?display_name=tag&sort=semver)](https://github.com/asllanmaciel/laravel-saas-blueprint/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-0f766e.svg)](LICENSE)

Um mapa arquitetural independente de fornecedor para planejar SaaS multi-tenant em Laravel sem transformar o primeiro release em um monólito impossível de operar.

Este projeto é documentação comunitária. Ele não contém código, regras de negócio ou infraestrutura de produtos comerciais.

> **Não é um starter kit Laravel.** O objetivo é ajudar a tomar decisões e definir limites arquiteturais antes de escolher pacotes, providers ou escrever a implementação específica do produto.

## Para quem serve

- Pessoas desenhando seu primeiro SaaS em Laravel.
- Times migrando um sistema single-tenant para multi-tenant.
- Tech leads revisando isolamento, billing, filas e observabilidade.
- Founders que precisam separar MVP de complexidade prematura.

## Como usar este blueprint

Uma sequência prática para um projeto novo ou uma revisão arquitetural:

1. **Escolha o modelo de isolamento** em [tenant-isolation-matrix.md](docs/tenant-isolation-matrix.md).
2. **Defina as fronteiras de segurança** em [security.md](docs/security.md), incluindo cache, storage, exports e operações administrativas.
3. **Modele billing e entitlements** sem acoplar regras de negócio ao provider em [billing-adapters.md](docs/billing-adapters.md).
4. **Defina idempotência e contexto de tenant** para jobs/webhooks em [jobs-and-webhooks.md](docs/jobs-and-webhooks.md).
5. **Planeje diagnóstico e operação** em [observability.md](docs/observability.md) e [operations.md](docs/operations.md).
6. **Valide recuperação de dados** com [backup-and-tenant-restore.md](docs/backup-and-tenant-restore.md).
7. **Corte complexidade para o primeiro release** usando [mvp-checklist.md](docs/mvp-checklist.md).

O resultado esperado não é “copiar a arquitetura inteira”, mas sair com decisões explícitas, trade-offs documentados e uma implementação mínima que preserve isolamento e operabilidade.

## Visão de arquitetura

```mermaid
flowchart TB
    U[Usuários e integrações] --> E[Edge: TLS, rate limit, WAF]
    E --> A[Aplicação Laravel]
    A --> T[Contexto do tenant]
    T --> D[(Banco de dados)]
    T --> C[(Cache)]
    A --> Q[Fila]
    Q --> W[Workers idempotentes]
    A --> B[Billing adapter]
    A --> O[Logs, métricas e traces]
    W --> O
```

## Princípios

1. Resolva o tenant uma vez e carregue o contexto explicitamente.
2. Aplique isolamento também em filas, cache, storage, logs e exports.
3. Trate billing como estado de negócio, não como um único webhook.
4. Torne jobs e webhooks idempotentes desde o início.
5. Defina limites e observabilidade antes de precisar investigar um incidente.
6. Entregue o MVP com a arquitetura mais simples que preserve esses limites.

## Guias

- [Arquitetura e decisões de tenancy](docs/architecture.md)
- [Matriz de decisão para isolamento de tenants](docs/tenant-isolation-matrix.md)
- [Segurança e isolamento](docs/security.md)
- [Billing adapters e entitlements](docs/billing-adapters.md)
- [Jobs, webhooks e idempotência](docs/jobs-and-webhooks.md)
- [Observabilidade multi-tenant](docs/observability.md)
- [Backup e restore por tenant](docs/backup-and-tenant-restore.md)
- [Operação e deploy](docs/operations.md)
- [Checklist de MVP](docs/mvp-checklist.md)

## O que este blueprint não prescreve

Ele não obriga banco por tenant, Kubernetes, microserviços ou um provedor específico de pagamento. Essas decisões dependem do risco, escala, equipe e requisitos regulatórios do produto.

Também não substitui documentação oficial do Laravel, requisitos regulatórios, threat modeling específico do produto ou testes reais de isolamento. O blueprint organiza decisões; a prova continua sendo da implementação.

## Contribuindo

Proponha melhorias por issue ou pull request. Decisões devem explicar contexto, alternativa e trade-off; evite receitas universais. Antes de participar, leia o [guia de contribuição](CONTRIBUTING.md) e o [código de conduta](CODE_OF_CONDUCT.md).

Para relatar conteúdo que possa incentivar uma prática insegura, siga a [política de segurança](.github/SECURITY.md) e não publique dados de sistemas reais.

- [Escolha uma tarefa para primeira contribuição](https://github.com/asllanmaciel/laravel-saas-blueprint/issues?q=is%3Aissue%20state%3Aopen%20label%3A%22good%20first%20issue%22)
- [Veja todas as tarefas que precisam de ajuda](https://github.com/asllanmaciel/laravel-saas-blueprint/issues?q=is%3Aissue%20state%3Aopen%20label%3A%22help%20wanted%22)

## Licença

[MIT](LICENSE).
