# Checklist de MVP SaaS

## Produto

- [ ] Problema e usuário principal definidos.
- [ ] Um fluxo de ativação mensurável.
- [ ] Limites do plano explícitos.
- [ ] Cancelamento e exclusão de conta planejados.

## Tenancy e autorização

- [ ] Tenant resolvido de forma central.
- [ ] Queries e policies testadas com dois tenants.
- [ ] Cache, storage, jobs e exports isolados.
- [ ] Operações administrativas auditáveis.

## Billing

- [ ] Estados de assinatura modelados localmente.
- [ ] Webhooks autenticados e idempotentes.
- [ ] Reprocessamento manual seguro.
- [ ] Falha de pagamento com comportamento definido.

## Operação

- [ ] CI executa testes e análise estática.
- [ ] Deploy e rollback documentados.
- [ ] Logs estruturados sem secrets.
- [ ] Alertas acionáveis e runbooks mínimos.
- [ ] Backup restaurado em teste.

## Segurança e privacidade

- [ ] Secrets fora do repositório.
- [ ] Rate limiting nas superfícies públicas.
- [ ] Retenção e exclusão de dados definidas.
- [ ] Canal privado para vulnerabilidades.
