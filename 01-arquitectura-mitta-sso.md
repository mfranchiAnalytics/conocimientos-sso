# Arquitectura MITTA SSO

## 1. Propósito

MITTA SSO es un servicio central de autenticación reutilizable para múltiples aplicaciones. Una aplicación nueva debe poder integrarse sin modificar ni volver a desplegar MITTA o `cognito-api`.

El alta de una aplicación requiere únicamente:

1. Crear un App Client público en Amazon Cognito.
2. Registrar la aplicación mediante la API administrativa existente.
3. Integrar el protocolo SSO en la aplicación cliente.

## 2. Principios de arquitectura

- Un User Pool central de Amazon Cognito.
- Un App Client independiente por aplicación.
- Los clientes web son públicos y no utilizan Client Secret.
- PKCE S256 es obligatorio.
- DynamoDB es la fuente de autoridad para aplicaciones y transacciones.
- El navegador nunca decide la callback, el Cognito Client ID ni los scopes finales.
- El código temporal de entrega es de un solo uso y tiene vida corta.
- Los tokens se cifran antes de persistirse temporalmente.
- Las altas, suspensiones y revocaciones se realizan por API, no mediante código.
- Las operaciones sensibles usan actualizaciones condicionales o transacciones DynamoDB.

## 3. Componentes

### 3.1 Amazon Cognito

Responsabilidades:

- Autenticar usuarios.
- Emitir ID Token, Access Token y Refresh Token.
- Mantener proveedores locales y sociales.
- Separar aplicaciones mediante App Clients.
- Exponer claims como `sub`, `email`, `given_name`, `family_name` y `custom:RUT`.

### 3.2 cognito-api

Responsabilidades:

- Mantener el catálogo dinámico de aplicaciones.
- Proteger la API administrativa.
- Inicializar transacciones SSO.
- Validar aplicaciones, ambientes, orígenes, scopes y PKCE.
- Controlar la máquina de estados de la transacción.
- Verificar criptográficamente tokens de Cognito.
- Crear códigos temporales de entrega.
- Cifrar y descifrar tokens.
- Canjear códigos de un solo uso.

### 3.3 MITTA Frontend

Responsabilidades:

- Mostrar el login visual central.
- Recibir la solicitud de la aplicación cliente.
- Llamar a `/sso/initialize`.
- Conservar temporalmente `transactionId` y `transactionKey`.
- Iniciar y completar la autenticación en Cognito.
- Llamar a `/sso/finalize` con el contrato reducido.
- Redirigir a la callback autorizada devuelta por el backend.

### 3.4 Aplicación cliente

Responsabilidades:

- Generar `state`, `nonce` y PKCE.
- Conservar localmente el `codeVerifier`.
- Redirigir hacia MITTA.
- Verificar el `state` de regreso.
- Canjear el código temporal en `/sso/exchange`.
- Validar los tokens como defensa adicional.
- Crear y destruir su sesión local.

## 4. Flujo integral

### 4.1 Inicio

```text
Aplicación cliente
  -> genera state, nonce y PKCE
  -> redirige a MITTA /auth/sso
```

### 4.2 Inicialización

```text
MITTA
  -> POST /sso/initialize
  -> cognito-api consulta mitta-sso-applications
  -> valida ACTIVE, ambiente, scopes y S256
  -> crea mitta-sso-transactions
  -> devuelve transactionId y transactionKey
```

Estado:

```text
CREATED, versión 1
```

### 4.3 Inicio de autenticación

```text
MITTA
  -> POST /sso/transactions/:id/start-authentication
```

Estado:

```text
AUTHENTICATING, versión 2
```

### 4.4 Usuario autenticado

```text
Cognito emite tokens
MITTA valida claims básicos
MITTA -> POST /sso/transactions/:id/authenticated
```

Estado:

```text
AUTHENTICATED, versión 3
```

### 4.5 Finalización

```text
MITTA -> POST /sso/finalize
cognito-api:
  - valida transactionKey
  - recupera callback, Client ID, state, nonce y PKCE desde DynamoDB
  - verifica ID Token y Access Token
  - cifra tokens
  - crea handoff
  - devuelve redirectUrl
```

Estado:

```text
HANDOFF_READY, versión 4
```

### 4.6 Canje

```text
Aplicación cliente -> POST /sso/exchange
cognito-api:
  - lee handoff READY
  - valida aplicación, expiración y PKCE
  - consume handoff
  - completa transacción
  - descifra y entrega tokens
```

Estado objetivo:

```text
Handoff: EXCHANGED
Transacción: COMPLETED, versión 5
```

## 5. Límites de confianza

### Datos que puede aportar el navegador

- `applicationId`
- `environment`
- `state`
- `nonce`
- `codeChallenge`
- `codeChallengeMethod`
- scopes solicitados
- `codeVerifier` durante exchange
- tokens emitidos por Cognito durante finalize

### Datos cuya autoridad es el backend

- Estado de la aplicación.
- Ambiente habilitado.
- Callback final.
- Logout URL.
- Allowed origin.
- Cognito Client ID.
- Scopes aceptados.
- Estado y versión de la transacción.
- Expiración de la transacción.
- Asociación entre handoff y transacción.

## 6. Compatibilidad transitoria

Mientras se finaliza la migración:

- MITTA conserva una caché temporal en `sessionStorage`.
- DynamoDB sigue siendo la autoridad real.
- La Aplicación B conserva `state` y `codeVerifier`, lo cual es necesario.
- El exchange atómico y la limpieza definitiva del legado están pendientes.

## 7. Criterio arquitectónico final

Una aplicación nueva debe poder integrarse mediante Cognito y la API administrativa sin:

- editar catálogos estáticos;
- modificar MITTA;
- modificar `cognito-api`;
- cambiar variables centrales por aplicación;
- redesplegar componentes centrales;
- editar DynamoDB manualmente.
