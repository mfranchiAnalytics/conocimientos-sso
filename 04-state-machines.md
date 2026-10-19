# Máquinas de estados de MITTA SSO

## 1. Estados de aplicaciones

```text
DRAFT
ACTIVE
SUSPENDED
REVOKED
ARCHIVED
```

### Transiciones permitidas

```text
DRAFT -> ACTIVE
DRAFT -> ARCHIVED

ACTIVE -> SUSPENDED
ACTIVE -> REVOKED

SUSPENDED -> ACTIVE
SUSPENDED -> REVOKED
SUSPENDED -> ARCHIVED

REVOKED -> ARCHIVED

ARCHIVED -> ninguna
```

### Significado

- `DRAFT`: configuración creada, aún no autorizada.
- `ACTIVE`: puede iniciar nuevas transacciones.
- `SUSPENDED`: bloqueo temporal de nuevas operaciones.
- `REVOKED`: autorización retirada.
- `ARCHIVED`: cierre lógico definitivo.

### Reglas

- La creación siempre termina en `DRAFT`.
- La activación revalida Cognito.
- Una aplicación archivada no puede modificarse.
- Toda actualización usa `version`.

## 2. Estados de transacciones SSO

```text
CREATED
AUTHENTICATING
AUTHENTICATED
HANDOFF_READY
COMPLETED
CANCELLED
EXPIRED
FAILED
```

### Camino feliz

```text
CREATED
  -> AUTHENTICATING
  -> AUTHENTICATED
  -> HANDOFF_READY
  -> COMPLETED
```

Versiones esperadas:

```text
CREATED          versión 1
AUTHENTICATING   versión 2
AUTHENTICATED    versión 3
HANDOFF_READY    versión 4
COMPLETED        versión 5
```

### Transiciones permitidas

```text
CREATED -> AUTHENTICATING
CREATED -> CANCELLED
CREATED -> FAILED

AUTHENTICATING -> AUTHENTICATED
AUTHENTICATING -> CANCELLED
AUTHENTICATING -> FAILED

AUTHENTICATED -> HANDOFF_READY
AUTHENTICATED -> FAILED

HANDOFF_READY -> COMPLETED
HANDOFF_READY -> FAILED

COMPLETED -> ninguna
CANCELLED -> ninguna
EXPIRED -> ninguna
FAILED -> ninguna
```

### Reglas condicionales

Cada transición exige:

- `transactionId` existente.
- transacción no expirada.
- estado actual esperado.
- versión actual esperada.
- `transactionKey` válida para operaciones desde MITTA.
- aplicación todavía disponible cuando corresponda.

## 3. Estados de handoff

```text
READY
EXCHANGED
```

### Camino feliz

```text
READY -> EXCHANGED
```

### Reglas

- El código público nunca se almacena en claro.
- Se guarda `codeHash`.
- El handoff expira rápidamente.
- El canje es de un solo uso.
- PKCE debe validarse antes del consumo definitivo.
- El handoff debe estar vinculado a `transactionId`.

## 4. Bloqueo optimista

Atributo:

```text
version
```

Patrón:

```text
leer versión N
actualizar con ConditionExpression version = N
incrementar a N + 1
```

Errores esperados:

```text
version_conflict
transaction_conflict
invalid_transaction_status
```

## 5. TTL y expiración

### Transacción

```text
expiresAt: milisegundos Unix
 ttl: segundos Unix
```

### Handoff

```text
expiresAt: milisegundos Unix
 ttl: segundos Unix
```

La aplicación siempre valida `expiresAt`. DynamoDB TTL solo realiza limpieza eventual.

## 6. Diseño atómico objetivo para exchange

Operación final pendiente:

```text
Validar handoff READY
Validar PKCE
TransactWriteItems:
  - handoff READY -> EXCHANGED
  - transaction HANDOFF_READY -> COMPLETED
```

Si cualquier condición falla, ninguna actualización debe aplicarse.
