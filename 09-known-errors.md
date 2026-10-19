# Errores conocidos y soluciones de MITTA SSO

## 1. `ssoApplicationsService.getActiveApplication is not a function`

### Causa

El método se perdió al reemplazar `sso-applications.service.js`.

### Solución

Restaurar `getActiveApplication({ applicationId, environment })` dentro del objeto exportado.

### Verificación

```powershell
node --env-file=.env -e "import('./src/services/sso-applications.service.js').then(({default:s}) => console.log(typeof s.getActiveApplication)).catch(console.error)"
```

Esperado: `function`.

## 2. Validaciones de entrada devolvían HTTP 500

### Causa

`normalizeInitializeRequest()` lanzaba `Error` sin `statusCode`.

### Solución

Capturar el error y convertirlo a `invalid_request`, HTTP 400.

### Casos comprobados

```text
PKCE plain -> 400
codeChallenge abc -> 400
```

## 3. `useSearchParams() should be wrapped in a suspense boundary`

### Causa

Next.js intentó prerenderizar páginas cliente con `useSearchParams` sin Suspense.

### Rutas afectadas

```text
/auth/sso/complete
/auth/validate
```

### Solución

Mover lógica cliente a un componente separado y envolverlo con `<Suspense>` desde `page.tsx`.

## 4. `Invalid ConditionExpression ... :now`

### Causa

La condición usaba `:now` pero `ExpressionAttributeValues` no lo definía.

### Solución

Calcular una sola vez:

```js
const now = Date.now()
```

Y definir:

```js
":now": now
```

## 5. `getTableName is not defined`

### Causa

`sso.repository.js` usa una constante `tableName`, pero un reemplazo parcial llamó `getTableName()`.

### Solución

Usar:

```js
TableName: tableName
```

Y ejecutar `ensureTableConfigured()`.

## 6. `MITTA no devolvió una sesión completa`

### Causa

Finalize cifraba tokens en camelCase:

```text
idToken
accessToken
refreshToken
tokenType
expiresIn
```

Aplicación B esperaba contrato OAuth en snake_case.

### Solución

Cifrar y devolver:

```text
id_token
access_token
refresh_token
token_type
expires_in
scope
```

## 7. `Route not found` al consultar una transacción

### Causa

`/sso/initialize` había fallado y `$initialized.transactionId` estaba vacío.

### Solución

Validar antes de construir URL:

```powershell
if (-not $initialized.transactionId) {
  throw "initialize no devolvió transactionId"
}
```

## 8. JSON inválido en AWS CLI con PowerShell

### Causa

PowerShell procesó comillas y produjo JSON inválido.

### Solución

Usar archivo temporal:

```powershell
'{":id":{"S":"mitta-sso-demo"}}' |
  Set-Content .\query-values.json -Encoding ascii
```

Y luego `file://query-values.json`.

## 9. `$initialized.transactionId = null` falla

### Causa

PowerShell usa `$null`, no `null`.

### Solución

```powershell
$initialized = $null
```

## 10. `.Replace()` no reconocido en PowerShell

### Causa

Se ejecutó `.Replace()` en una línea independiente.

### Solución

```powershell
$payload = $tokenParts[1].Replace("-", "+").Replace("_", "/")
```

## 11. `npm run build` falla en cognito-api

### Causa

`cognito-api` es Express JavaScript y no tiene script build.

### Solución

Usar `node --check` e imports de prueba.

No instalar TypeScript allí.

## 12. `npx tsc` intentó instalar paquete obsoleto en cognito-api

### Causa

El comando se ejecutó en el proyecto incorrecto.

### Solución

Ejecutar TypeScript solo en el frontend Next.js.

## 13. `npm.ps1 ... ejecución de scripts está deshabilitada`

### Causa

Política de ejecución de PowerShell.

### Solución sin cambiar políticas:

```powershell
npm.cmd run build
npm.cmd run dev
npx.cmd tsc --noEmit
```

## 14. `Failed to fetch`

### Causas posibles

- API detenida o reiniciándose.
- CORS.
- URL pública incorrecta.
- Amplify todavía usa localhost.
- API Gateway no permite OPTIONS.

### Diagnóstico

- Verificar `/sso/health`.
- Revisar Network.
- Revisar Origin.
- Revisar variables de Amplify.
- Revisar Lambda y API Gateway CORS.

## 15. `invalid_transaction_key`

### Causas

- Clave alterada.
- Transacción previa al cambio.
- Caché local antigua.

### Solución

Ventana InPrivate nueva y flujo nuevo.

## 16. `invalid_transaction_status`

### Causas

- Repetición de una transición.
- Reutilización de transacción.
- Finalize duplicado.

### Solución

No reintentar con la misma transacción. Crear un flujo nuevo.

## 17. `invalid_grant`

### Causas

- Código inexistente.
- Código expirado.
- Código ya usado.
- PKCE incorrecto.
- applicationId incorrecto.

### Nota

El exchange atómico pendiente debe validar PKCE antes de consumir el handoff.

## 18. Amplify no mostraba variables de entorno

### Solución

Usar AWS CLI:

```powershell
aws amplify update-branch `
  --app-id d1ttj8ckmgmipf `
  --branch-name develop `
  --environment-variables file://amplify-develop-env.json `
  --region us-east-1 `
  --profile Mitta
```

Después redesplegar la rama.

## 19. `git origin` no es un comando

### Causa

`origin` es nombre de remoto, no subcomando.

### Comandos correctos

```text
git remote -v
git remote show origin
git push origin main
```

## 20. Riesgo: backups dentro de `src`

### Problema

Archivos `*.backup.js` pueden confundir búsquedas y tooling.

### Solución

Después de validar la migración, moverlos fuera de `src` o eliminarlos y mantener el historial en Git.
