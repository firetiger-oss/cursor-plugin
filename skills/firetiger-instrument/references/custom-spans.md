# Custom spans, log correlation & attribute hygiene

Auto-instrumentation covers common libraries. Add manual spans for business-critical operations, and correlate
logs with traces so Firetiger can join them.

## Custom spans

### Node.js
```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('your-service-name');

async function processOrder(orderId: string, customerId: string) {
  return await tracer.startActiveSpan('processOrder', async (span) => {
    span.setAttribute('order.id', orderId);
    span.setAttribute('customer.id', customerId);
    try {
      const result = await executeOrder(orderId);
      span.addEvent('order.completed', { 'order.total': result.total });
      return result;
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Python
```python
from opentelemetry import trace

tracer = trace.get_tracer("your-service-name")

def process_order(order_id: str, customer_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("customer.id", customer_id)
        try:
            result = execute_order(order_id)
            span.add_event("order.completed", {"order.total": result.total})
            return result
        except Exception as e:
            span.set_status(trace.StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

## Log correlation

Inject `trace_id` / `span_id` into logs so `"opentelemetry/logs/{service}"` rows join to their traces.

### Node.js (Winston)
```typescript
import winston from 'winston';
import { trace } from '@opentelemetry/api';

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format((info) => {
      const span = trace.getActiveSpan();
      if (span) {
        const ctx = span.spanContext();
        info.trace_id = ctx.traceId;
        info.span_id = ctx.spanId;
      }
      return info;
    })(),
    winston.format.json()
  ),
  transports: [new winston.transports.Console()],
});
```

### Python (structlog)
```python
import structlog
from opentelemetry import trace

def add_trace_context(logger, method_name, event_dict):
    span = trace.get_current_span()
    if span.is_recording():
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict

structlog.configure(processors=[add_trace_context, structlog.processors.JSONRenderer()])
```

## Attribute hygiene

- **Semantic conventions:** `http.method`, `http.status_code`, `http.route`; `db.system`, `db.statement`,
  `db.name`; `rpc.service`, `rpc.method`. Prefix custom business attributes with `app.`.
- **Cardinality:** high-cardinality *values* (user IDs) are fine; high-cardinality attribute *names* are not.
- **Never** put secrets, tokens, passwords, or PII in attributes.
- These attributes become queryable columns in Firetiger (`attributes.app.order_id`), inferred to depth 2 —
  see the `firetiger-query` schema reference.

## Performance

- Use `BatchSpanProcessor` in production (not `SimpleSpanProcessor`).
- Enable `gzip` compression on exporters.
- Size batches to your traffic; set queue limits to bound memory.
- Add manual spans for checkout/payment and anything measured for SLOs.

## Where to go next

- **Query your custom spans and correlated logs** — the attributes you set become columns; see the `firetiger-query` skill (and its schema reference for the depth-2 inference rule).
- **Investigate with them** — once you have rich spans, `firetiger-investigate` can reason over them to diagnose incidents.
