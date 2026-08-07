# Operação e observabilidade

## Ambientes

Mantenha desenvolvimento, staging e produção isolados. Dados reais não devem ser copiados para desenvolvimento sem anonimização e justificativa.

## Deploy

Um deploy previsível deve incluir:

- build reproduzível;
- migrações compatíveis com a versão anterior;
- health check da aplicação e dos workers;
- rollback conhecido;
- reinício controlado de filas;
- verificação pós-deploy.

## Sinais mínimos

- taxa de erro e latência por rota;
- profundidade e idade da fila;
- falhas e tentativas de jobs;
- webhooks recebidos, rejeitados e duplicados;
- uso por tenant sem dados sensíveis em labels;
- eventos de billing e mudança de plano.

## Runbooks

Crie procedimentos curtos para fila parada, webhook atrasado, banco saturado, storage indisponível, vazamento de secret e rollback.

## Backups

Backup só é confiável após um teste de restauração. Defina RPO, RTO, retenção, criptografia e responsável por executar testes periódicos.
