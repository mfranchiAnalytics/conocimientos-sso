# Estado del proyecto MITTA SSO

## Última actualización

19 de octubre de 2026

## 1. Resumen ejecutivo

El núcleo SSO dinámico funciona en pruebas locales. MITTA obtiene aplicaciones desde DynamoDB, crea transacciones del lado servidor, usa Client IDs dinámicos, verifica tokens y genera handoffs. Aplicación B completa el flujo y crea su sesión.

La prueba híbrida con MITTA en Amplify, cognito-api en API Gateway y Aplicación B local está en preparación.

## 2. Fases completadas

### Parte 1. Catálogo dinámico

Estado: **COMPLETADA**

- Tabla `mitta-sso-applications`.
- Índice `origin-index`.
- Repositorio y servicio.
- Validación de Cognito.
- Migración de `mitta-sso-demo`.

### Parte 2. API administrativa

Estado: **COMPLETADA**

- Grupos administrativos.
- JWT administrativo.
- API CRUD.
- Revalidación Cognito.
- Activación y transiciones.
- Bloqueo optimista.

### Parte 3. Initialize y transacciones

Estado: **COMPLETADA**

- Tabla `mitta-sso-transactions`.
- TTL.
- `/sso/initialize`.
- Validaciones de scopes, state, nonce y PKCE.

### Parte 4. MITTA conectado a initialize

Estado: **COMPLETADA**

- `SsoContent` dinámico.
- transactionId generado por backend.
- Client ID dinámico.
- Build Next.js correcto.

### Parte 5B. Transiciones del backend

Estado: **COMPLETADA**

- transactionKey.
- `CREATED -> AUTHENTICATING`.
- `AUTHENTICATING -> AUTHENTICATED`.

### Parte 5C. Transiciones conectadas a MITTA

Estado: **COMPLETADA**

- Flujo real genera versiones 1, 2 y 3.

### Parte 5D. Finalize dinámico

Estado: **FUNCIONAL LOCALMENTE**

- JWT dinámico.
- Handoff transaccional.
- Body reducido.
- Tokens en snake_case.
- `AUTHENTICATED -> HANDOFF_READY`.

## 3. Estado actual exacto

### MITTA Amplify

Variables de la rama `develop` configuradas por AWS CLI:

```text
NEXT_PUBLIC_SSO_INITIALIZE_URL
NEXT_PUBLIC_SSO_FINALIZE_URL
NEXT_PUBLIC_SSO_COGNITO_REGION
```

Pendiente:

- confirmar valores limpios;
- desplegar `develop` con código reciente;
- verificar que no llame a localhost.

### Lambda cognito-api

Pendiente:

- desplegar código reciente;
- configurar `MITTA_FRONTEND_ORIGIN`;
- configurar CORS para Amplify y localhost:3001.

### Aplicación B

Pendiente:

- confirmar `.env.local` híbrido;
- reiniciar;
- completar prueba híbrida.

### Git

`mitta-sso-demo` tiene:

```text
origin -> GitHub
azure  -> Azure DevOps
```

Pendiente:

- revisar estado;
- comprobar si Azure está vacío;
- push de ramas y tags.

## 4. Próximas tareas inmediatas

### Tarea A. Publicar en Azure DevOps

```text
git status --short
git ls-remote azure
git push --dry-run azure --all
git push azure --all
git push azure --tags
git ls-remote --heads azure
git branch -vv
```

Mantener `origin/main` como upstream principal.

### Tarea B. Completar prueba híbrida

1. Redesplegar Amplify `develop`.
2. Actualizar Lambda y CORS.
3. Confirmar código reciente en Lambda.
4. Reiniciar Aplicación B.
5. Ejecutar flujo InPrivate.
6. Confirmar estados HTTP.

### Tarea C. Exchange atómico

Objetivo:

```text
handoff READY -> EXCHANGED
transaction HANDOFF_READY -> COMPLETED
version 4 -> 5
```

Validar PKCE antes de consumir.

## 5. Fases posteriores

- Eliminar catálogos y configuración estática.
- Probar segunda aplicación real.
- Registro, confirmación y recuperación.
- Google y Apple.
- CORS dinámico.
- Auditoría.
- Documentación final.
- Hardening productivo.

## 6. Riesgos abiertos

- Exchange legado consume antes de validar completamente PKCE.
- MITTA conserva caché de compatibilidad en `sessionStorage`.
- CORS dinámico no está cerrado.
- Archivos backup pueden quedar dentro de `src`.
- Variables residuales fijas todavía pueden existir.
- Falta prueba de segunda aplicación sin despliegues centrales.

## 7. Definición de terminado

El proyecto estará terminado cuando una nueva aplicación pueda:

1. Crear su App Client en Cognito.
2. Registrarse y activarse por API.
3. Integrar start, callback y exchange.
4. Completar autenticación con su propio Client ID.
5. Suspenderse o revocarse inmediatamente.

Sin modificar o redesplegar MITTA ni `cognito-api`.
