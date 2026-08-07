# Segurança e isolamento

## Controles essenciais

- Negue acesso quando o tenant não puder ser resolvido de forma inequívoca.
- Use policies para autorização de recurso, mesmo após filtrar por tenant.
- Inclua o tenant em chaves de cache, caminhos de storage e nomes de export.
- Não aceite `tenant_id` arbitrário do cliente como prova de autorização.
- Valide assinatura e idempotência de webhooks.
- Rotacione secrets e mantenha ambientes separados.
- Registre ações administrativas sem armazenar dados sensíveis desnecessários.

## Testes de isolamento

Cada recurso crítico deve ter testes que criem dois tenants e confirmem que:

1. o tenant A acessa seu próprio recurso;
2. o tenant A não lê, altera ou exclui o recurso do tenant B;
3. buscas, exports e jobs mantêm o mesmo limite;
4. cache e arquivos não vazam entre tenants.

## Dados pessoais

Mapeie finalidade, retenção, exportação e exclusão. Criptografia ajuda, mas não substitui minimização, autorização e processo operacional.

## Resposta a incidentes

Defina responsável, canal, evidências preservadas, rotação de credenciais, comunicação e post-mortem antes do primeiro incidente real.
