# Decisões Arquiteturais

Este documento complementa o README com os contratos de mensagens, o estado persistido da Saga, idempotência e tratamento de falhas. Trata-se de um blueprint; os componentes descritos são propostas arquiteturais, não código implementado neste repositório.

## ADR-001 — Saga orquestrada

- **Status:** aceita.
- **Contexto:** Order, Inventory e Payment possuem bancos independentes; uma transação ACID única entre eles não é viável.
- **Decisão:** orquestrador persistente, comunicação assíncrona por broker e transações locais por serviço.
- **Motivo:** fluxo explícito, auditoria, timeouts por passo e compensações em ordem inversa.
- **Consequências:** adiciona um componente operacional; mitigar com múltiplas instâncias, estado durável, bloqueio otimista e monitoramento.

## 1. Contratos de mensagens

Todas as mensagens devem incluir um envelope comum:

```json
{
  "messageId": "uuid",
  "sagaId": "uuid",
  "correlationId": "orderId",
  "type": "ReserveStock",
  "occurredAt": "2026-10-23T14:00:00Z",
  "attempt": 1,
  "payload": {}
}
```

| Mensagem | Origem → destino | Dados principais |
|---|---|---|
| `OrderCreated` | Order → Orchestrator | `orderId`, `items`, `amount`, `customerId` |
| `ReserveStock` | Orchestrator → Inventory | `orderId`, itens e quantidades |
| `StockReserved` / `StockRejected` | Inventory → Orchestrator | reserva ou motivo da recusa |
| `ProcessPayment` | Orchestrator → Payment | `orderId`, valor, `paymentId` |
| `PaymentApproved` / `PaymentRejected` | Payment → Orchestrator | resultado e referência do gateway |
| `ReleaseStock` / `StockReleased` | Orchestrator ↔ Inventory | identificador da reserva |
| `CancelOrder` / `OrderCancelled` | Orchestrator ↔ Order | pedido e motivo |
| `ApproveOrder` / `OrderApproved` | Orchestrator ↔ Order | identificador do pedido |

Tópicos sugeridos: `saga.commands.inventory`, `saga.commands.payment`, `saga.commands.order` e `saga.replies`, com DLQ por tópico. Particionar por `orderId` ajuda a preservar a ordem das mensagens de um mesmo pedido.

## 2. Persistência e máquina de estados

A tabela `saga_state` deve registrar `saga_id`, `order_id`, `status`, `current_step`, `payload`, `failure_reason`, `retry_count`, `timeout_at`, `version`, `created_at` e `updated_at`.

Tabelas auxiliares propostas:

- **`saga_log`:** trilha append-only de mensagens recebidas e emitidas.
- **`processed_messages`:** deduplicação por `messageId` em cada consumidor.
- **`outbox`:** mensagens a publicar, gravadas na mesma transação local do efeito de negócio.

Estados principais: `STARTED`, `STOCK_RESERVED`, `PAYMENT_APPROVED`, `COMPLETED`, `COMPENSATING`, `COMPENSATED` e `COMPENSATION_FAILED`. Veja a máquina de estados em [diagrams/saga-state-machine.svg](../diagrams/saga-state-machine.svg).

## 3. Idempotência (Idempotent Consumer)

Com entrega *at-least-once*, a mesma mensagem pode ser recebida mais de uma vez. Cada consumidor deve:

1. Abrir uma transação local.
2. Registrar o `messageId` em `processed_messages`; se já existir, não repetir o efeito de negócio.
3. Executar a operação e persistir a resposta na outbox na mesma transação.
4. Confirmar a mensagem no broker somente após o commit.

Garantias adicionais:

- `POST /orders` aceita `Idempotency-Key` para evitar pedidos duplicados.
- `paymentId` é usado como chave de idempotência no gateway, quando suportado.
- `ReleaseStock` e `CancelOrder` devem ser idempotentes.
- `version` na Saga aplica controle otimista para evitar que duas instâncias avancem o mesmo estado.
- Respostas tardias ou fora de ordem são registradas. Se houver risco de reserva tardia após timeout, emitir uma liberação idempotente.

## 4. Matriz de falhas e decisões

| Cenário | Detecção | Comportamento |
|---|---|---|
| Mensagem duplicada | `messageId` já processado | Não repetir o efeito; devolver resposta registrada |
| Estoque insuficiente | `StockRejected` | Cancelar pedido; não liberar reserva inexistente |
| Pagamento recusado | `PaymentRejected` | `ReleaseStock` e depois `CancelOrder` |
| Timeout do Inventory | `timeout_at` vencido | Reenviar comando idempotente até o limite; depois compensar |
| Timeout do Payment | Sem resposta no prazo | Consultar o gateway por `paymentId` antes de decidir compensar |
| Resposta tardia | Passo atual não corresponde | Registrar e reconciliar; liberar reserva tardia se necessário |
| Falha após commit e antes da publicação | Outbox pendente | Relay republica; consumidor deduplica |
| Queda do orquestrador | Estado persistido | Outra instância retoma do passo atual |
| Compensação esgota retries | Limite de tentativas | DLQ, `COMPENSATION_FAILED`, alerta e intervenção |
| Gateway indisponível | Erros e timeouts | Circuit Breaker; reconciliar resultado de pagamentos incertos |

## 5. Timeouts, retry e DLQ

- Definir timeout para cada passo (por exemplo, Inventory 10 s, Payment 30 s e compensações 60 s); os valores são parâmetros iniciais e devem ser ajustados com métricas.
- Aplicar retry com backoff exponencial e jitter apenas para falhas transitórias; recusa de negócio não deve ser repetida como erro técnico.
- Para pagamento sem resposta, consultar o gateway antes de reenviar ou compensar, evitando cobrança duplicada.
- Encaminhar mensagens inválidas ou que esgotaram tentativas para uma Dead-Letter Queue (DLQ), preservando identificadores, exceção e número de tentativas.
- Alertar sobre DLQs com mensagens, sagas presas em compensação e idade da mensagem mais antiga.
- Um timeout scanner deve reivindicar cada Saga de forma concorrente-segura (por exemplo, bloqueio de linha ou lease) antes de processá-la.

## 6. Estratégias de resiliência

| Mecanismo | Aplicação | Objetivo |
|---|---|---|
| Outbox + relay | Todos os serviços | Evitar perda entre commit e publicação |
| Idempotência | Consumidores, API e pagamento | Tornar reentregas seguras |
| Retry + backoff + jitter | Erros transitórios | Evitar tempestade de retentativas |
| Circuit Breaker | Gateway financeiro | Evitar chamadas contínuas a dependência degradada |
| Bulkhead | Pools separados | Isolar saturação de dependências |
| Réplicas do orquestrador | Orquestração | Disponibilidade com estado persistido |
| TTL de reserva | Inventory | Evitar estoque preso após falhas prolongadas |
| Logs, métricas e traces | Todos | Correlacionar eventos por `sagaId` e `correlationId` |

## 7. Riscos mitigados e limites

| Risco | Mitigação |
|---|---|
| Divergência entre pedido e estoque | Compensações em ordem inversa e estado persistido |
| Cobrança duplicada | Chave idempotente no gateway e reconciliação |
| Perda de mensagem | Outbox e confirmação após commit |
| Mensagens venenosas | DLQ, alerta e reprocessamento controlado |
| Orquestrador indisponível | Réplicas e retomada pelo estado persistido |
| Estoque preso | TTL e timeout scanner |
| Falta de rastreabilidade | Log de Saga e IDs de correlação |

A Saga oferece **consistência eventual**, não isolamento ACID global. Entre etapas, o pedido pode estar `PENDING` e o estoque `RESERVED`. As interfaces devem tratar esses estados intermediários. Compensação também não apaga efeitos irreversíveis (por exemplo, uma notificação já enviada); por isso, notificar o cliente somente após o estado final. Versionar contratos de mensagens e manter compatibilidade retroativa durante evoluções.
