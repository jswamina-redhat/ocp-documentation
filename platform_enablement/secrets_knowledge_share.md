# Introduction

Secrets securely store sensitive data like passwords, tokens, or configurations while decoupling credentials from application code.

# Secret Types and Creation

You **must** create a secret before creating the pods that depend on that secret. The value in the `type` field indicates the structure of the secret’s key names and values.
- Opaque
- Docker CFG
- Basic Auth
- SSH Auth
- TLS

### Opaque (`type=opaque`)
- Default secret type for **arbitrary** key-value pairs
- Use When:
  - Storing database credentials
  - API keys for external services
  - Configuration values that shouldn't be hard coded
