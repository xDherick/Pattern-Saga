# Decisões Arquiteturais (ADR) - Saga de Compra

Documento técnico complementar ao [README](../README.md). Registra as decisões, os riscos mitigados e as estratégias de resiliência da Saga orquestrada (Order → Inventory → Payment).

## 1. Decisões

| ID | Decisão | Alternativa descartada | Motivo |
|---|---|---|---|
| ADR-01 | Saga **orquestrada**, orquestrador no Order Service | Coreografia pura | Fluxo explícito, compensação controlada, estado auditável (ver README §2). |
| ADR-02 | Comunicação **assíncrona** por broker entre serviços; síncrona só cliente→API | Chamadas HTTP síncronas encadeadas | Evita acoplamento temporal e falha em cascata. |
| ADR-03 | **Transactional Outbox** em todos os serviços | Publicar direto no broker após o commit | Atualizar banco e publicar mensagem são duas operações; sem outbox, uma falha entre elas perde ou duplica mensagens. |
| ADR-04 | **Idempotent Consumer** (`processed_messages`) | Confiar em entrega exactly-once | Brokers entregam at-least-once; exactly-once ponta a ponta não é garantido. |
| ADR-05 | Pagamento é a transação **pivô** | Pagamento antes do estoque | Passos compensáveis antes do pivô, retentáveis depois; evita cobrar para depois não ter estoque. |
| ADR-06 | Estado da saga **persistido** (`saga_state`) | Estado em memória | Retomada após crash e escala horizontal do orquestrador. |
| ADR-07 | Particionamento por `orderId` | Fila única sem chave | Garante ordem das mensagens de um mesmo pedido. |

## 2. Riscos mitigados

| Risco | Impacto | Mitigação |
|---|---|---|
| Mensagem duplicada | Estoque reservado ou cobrança em dobro | Idempotent Consumer + idempotency key no gateway. |
| Perda de mensagem (crash entre commit e publish) | Saga travada, estado inconsistente | Transactional Outbox + relay com retentativa. |
| Mensagens fora de ordem | Transição de estado inválida | Chave de partição `orderId`; máquina de estados rejeita transições inválidas; `version` otimista. |
| Timeout no gateway de pagamento | Cobrança feita mas pedido cancelado (ou o inverso) | Em timeout, **consultar** o status por idempotency key antes de decidir. |
| Falha na compensação | Estoque preso em reserva | Compensação retentável + DLQ + alerta + status `COMPENSATION_FAILED`. |
| Reserva órfã (saga abandonada) | Estoque indisponível indevidamente | Reserva com **TTL**; *saga reaper* compensa sagas vencidas. |
| Falta de isolamento entre sagas | Leituras sujas (ex.: pedido `PENDING` exibido como confirmado) | *Semantic lock*: `PENDING` explícito; só `APPROVED` dispara efeitos externos. |
| Queda do orquestrador | Sagas paradas | Réplicas stateless + retomada a partir de `saga_state`. |
| Falha em cascata do gateway externo | Esgotar threads/conexões do Payment Service | Circuit Breaker + timeout + bulkhead. |
| Mensagem "veneno" | Loop infinito de reprocessamento | Limite de tentativas + DLQ. |

## 3. Estratégias de resiliência

### 3.1 Retry

- **Onde:** reenvio de comandos pelo orquestrador, relay do outbox e chamada ao gateway.
- **Política:** backoff exponencial com **jitter** (ex.: 1s, 2s, 4s, 8s, até um teto), número máximo de tentativas por passo (ex.: 5).
- **Somente se for seguro repetir:** todo comando é idempotente (ver README §5.1). Erros de negócio (pagamento recusado, estoque insuficiente) **não** são retentados; são respostas válidas que disparam compensação.
- Erros transitórios (timeout, 5xx, conexão) são retentados; erros permanentes (4xx de validação) vão direto para tratamento de falha.

### 3.2 Circuit Breaker

Aplicado na chamada do Payment Service ao gateway financeiro externo.

| Estado | Comportamento |
|---|---|
| **Closed** | Chamadas normais; monitora taxa de falhas em janela deslizante. |
| **Open** | Acima do limiar (ex.: 50% de falhas na janela), chamadas falham rápido, sem tocar o gateway. A mensagem volta para retentativa posterior (não é recusa de pagamento). |
| **Half-open** | Após o tempo de espera, libera poucas chamadas de teste; sucesso fecha o circuito, falha reabre. |

Complementos: **timeout** curto por chamada (ex.: 3-5s) e **bulkhead** (pool de threads/conexões dedicado ao gateway) para que sua lentidão não derrube o serviço inteiro. Com o circuito aberto, a saga não é compensada imediatamente: ela aguarda dentro do `deadline_at` e só compensa se o prazo vencer.

### 3.3 Dead-Letter Queue (DLQ)

- Cada fila/tópico de entrada possui uma DLQ (`inventory.commands.dlq`, `payment.commands.dlq`, `saga.replies.dlq`).
- **Entra na DLQ** a mensagem que esgotou as tentativas, que não pode ser desserializada (contrato inválido) ou que gera erro não retentável.
- **Conteúdo preservado:** payload original, `messageId`, `sagaId`, motivo do erro, stack resumida, número de tentativas e timestamp.
- **Alerta:** profundidade da DLQ > 0 aciona alarme (ex.: PagerDuty/Slack) com o `sagaId`.
- **Tratamento:** triagem operacional e *redrive* (reprocessar após corrigir a causa) é seguro por causa da idempotência. Se a mensagem era uma compensação, a saga permanece `COMPENSATION_FAILED` até resolução, e **nunca** é descartada em silêncio.

### 3.4 Timeouts e prazos

Cada passo grava `deadline_at` em `saga_state`. O *saga reaper* trata os vencidos conforme a tabela do README §5.3. Os valores são parametrizáveis por passo (ex.: estoque 30s, pagamento 2min, compensação 5min) e devem ser calibrados com métricas reais.

### 3.5 Observabilidade

- **Correlation ID:** `sagaId`/`orderId` em todos os logs, mensagens e traces (OpenTelemetry).
- **Métricas:** sagas por status, duração fim a fim, taxa de compensação, tentativas por passo, profundidade de DLQ, estado do Circuit Breaker, idade da saga não terminal mais antiga.
- **Alertas:** sagas em `COMPENSATION_FAILED`, DLQ não vazia, sagas não terminais além do prazo.

## 4. Matriz de decisão: duplicatas, reentregas e timeouts

| Evento observado | Decisão | Efeito colateral evitado |
|---|---|---|
| `messageId` já em `processed_messages` | Ignorar, ack e reenviar resposta anterior se necessário | Reserva/cobrança duplicada |
| `ReserveStock` repetido para o mesmo `sagaId` | No-op (reserva existente) | Baixa de estoque em dobro |
| `ReleaseStock` repetido | No-op (reserva já `RELEASED`) | Devolução de estoque em dobro |
| `ProcessPayment` repetido | Gateway devolve o resultado da mesma idempotency key | Cobrança em dobro |
| Resposta chega com saga já `COMPENSATED`/`COMPLETED` | Descartar e registrar | Transição inválida |
| Timeout sem resposta de pagamento | Consultar status antes de decidir | Cancelar pedido já pago |
| Timeout sem resposta de estoque | Reenviar comando e, esgotado, compensar C1 | Pedido preso em `PENDING` |
| Falha ao publicar no broker | Outbox mantém a mensagem e o relay tenta de novo | Mensagem perdida |

## 5. Pontos em aberto / evolução

- Definir política de **reembolso** como processo de negócio caso o pedido precise ser cancelado após o pivô (ex.: fraude detectada depois).
- Avaliar **Outbox com CDC** (ex.: Debezium) em vez de polling para reduzir latência.
- Calibrar timeouts, limiares do Circuit Breaker e TTL das reservas com dados de produção.
- Versionamento de contratos de mensagens (schema registry) para evolução sem quebrar consumidores.
