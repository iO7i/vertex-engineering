# Authority boundary diagram source

```text
evidence -> Brain proposal -> application request -> Control Plane decision
                                                    |
                                                    v
                                      bounded durable execution context

proposal != permission
workflow != permission
credential != ambient permission
```

```mermaid
flowchart LR
  B[Brain proposal] --> Q[Action request]
  Q --> C{Current authority?}
  C -->|allowed| W[Bounded workflow]
  C -->|denied or stale| D[No mutation]
```
