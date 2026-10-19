# Mapa de código de MITTA SSO

## 1. cognito-api

Ruta local:

```text
D:\Analytics Latam\Mitta\cognito-api
```

### Configuración

```text
src/config/aws.js
```

Clientes de Cognito y DynamoDB Document Client.

```text
src/config/sso-application-rules.js
```

Estados, ambientes, scopes y proveedores permitidos.

```text
src/config/sso-transaction-rules.js
```

Estados, patrones y transiciones de transacción.

### Repositorios

```text
src/repositories/sso-applications.repository.js
```

CRUD y consultas del catálogo dinámico.

```text
src/repositories/sso-transactions.repository.js
```

Creación, lectura y transiciones condicionales de transacciones.

```text
src/repositories/sso.repository.js
```

Repositorio legado/compatible de handoffs. Contiene `create`, `findByCodeHash` y `consume`.

```text
src/repositories/sso-finalize.repository.js
```

Crea handoff y cambia `AUTHENTICATED -> HANDOFF_READY` mediante `TransactWriteItems`.

### Servicios

```text
src/services/sso-applications.service.js
```

Reglas de negocio del catálogo y validación de App Clients.

```text
src/services/sso-transactions.service.js
```

Initialize, transactionKey, lectura y transiciones iniciales.

```text
src/services/sso-jwt.service.js
```

Verificación JWT dinámica por Client ID y token use.

```text
src/services/sso-token.service.js
```

AES-256-GCM para cifrar y descifrar el objeto de tokens.

```text
src/services/sso.service.js
```

Finalize dinámico y exchange compatible. El exchange atómico pendiente se implementará aquí o en un servicio específico.

```text
src/services/jwt-auth.service.js
```

Verificación de Access Tokens para endpoints administrativos.

### Rutas

```text
src/routes/sso.js
```

Health, initialize, transiciones, finalize, exchange y diagnóstico administrativo.

```text
src/routes/sso-applications-admin.js
```

API administrativa de aplicaciones.

### Middleware

```text
src/middleware/sso-admin.middleware.js
```

Permisos por grupos Cognito.

### Utilidades

```text
src/utils/sso-application-validation.js
src/utils/sso-initialize-validation.js
src/utils/http-error-response.js
src/utils/sso-service-error.js
src/utils/pkce.js
src/utils/crypto.js
```

### Entrada de aplicación

```text
app.js
```

Express, CORS, rutas, Swagger y acceso protegido a documentación.

## 2. MITTA Frontend

Ruta local:

```text
D:\Analytics Latam\Mitta\CL.Mitta.Cognito\frontend
```

### Inicio SSO

```text
src/app/auth/sso/page.tsx
src/components/auth/SsoContent.tsx
```

Lee parámetros, llama initialize, mueve a AUTHENTICATING y navega al login.

### Login

```text
src/components/auth/LoginForm.tsx
src/services/sso-cognito-auth.service.ts
```

Autentica contra el Client ID dinámico y mueve a AUTHENTICATED.

### Finalización

```text
src/app/auth/sso/complete/page.tsx
src/components/auth/SsoCompleteContent.tsx
```

Envía body reducido a `/sso/finalize` y redirige a la callback autorizada.

### Validación y perfil

```text
src/app/auth/validate/page.tsx
src/components/auth/ValidateContent.tsx
src/components/auth/CompleteProfileForm.tsx
```

Mantienen continuidad de la transacción en flujos auxiliares.

### Persistencia temporal

```text
src/lib/sso-transaction-storage.ts
```

Caché de compatibilidad en `sessionStorage`. Pendiente de reducir.

### Clientes API

```text
src/services/sso-initialize.service.ts
src/services/sso-transaction-state.service.ts
```

## 3. Aplicación B: mitta-sso-demo

Ruta local:

```text
D:\Analytics Latam\Mitta\mitta-sso-demo
```

### Inicio

```text
app/auth/start/page.tsx
```

Genera state, nonce, codeVerifier y codeChallenge. Redirige a MITTA.

### Callback

```text
app/auth/callback/page.tsx
```

Verifica state, recupera codeVerifier, llama exchange y crea sesión.

### PKCE

```text
lib/pkce.ts
```

### Transacción local

```text
lib/sso-transaction-storage.ts
```

Conserva state, codeVerifier, applicationId y datos temporales.

### Sesión final

```text
lib/mitta-session-storage.ts
```

Conserva la sesión final de demo.

## 4. Archivos pendientes de retirar

- Catálogo estático de aplicaciones en MITTA.
- Catálogo estático de aplicaciones en cognito-api.
- `SSO_CLIENT_ID` fijo.
- `NEXT_PUBLIC_SSO_COGNITO_CLIENT_ID` residual.
- Archivos `*.backup.js` dentro de `src`.
- Lógica de exchange legada.
- Campos redundantes de la caché local de MITTA.

## 5. Repositorios Git

### mitta-sso-demo

```text
origin -> GitHub
azure  -> Azure DevOps
```

GitHub:

```text
https://github.com/AnalyticsLatam/mitta-sso-demo.git
```

Azure DevOps:

```text
https://MittaDevOps@dev.azure.com/MittaDevOps/CL.Mitta.Cognito/_git/mitta-sso-demo
```
