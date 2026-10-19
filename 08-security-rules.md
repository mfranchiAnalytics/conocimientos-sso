# Reglas de seguridad de MITTA SSO

## 1. Información que nunca debe compartirse

- Contraseñas.
- Access Tokens.
- ID Tokens.
- Refresh Tokens.
- `transactionKey`.
- `codeVerifier`.
- `handoff_code`.
- `SSO_TOKEN_ENCRYPTION_KEY`.
- AWS Secret Access Keys.
- Azure DevOps PAT.
- Client Secrets.
- Cookies de sesión.

Si algún secreto se expone, debe rotarse y tratarse como comprometido.

## 2. Variables públicas

Toda variable `NEXT_PUBLIC_*` es visible en el navegador.

Puede contener:

- URLs públicas.
- región AWS.
- applicationId público.
- Client ID público cuando sea necesario.

No puede contener:

- contraseñas;
- claves de cifrado;
- PAT;
- Client Secret;
- AWS credentials;
- tokens.

## 3. Cognito App Clients

Para aplicaciones web:

- Public client.
- Sin Client Secret.
- PKCE S256.
- Callback exacta.
- Logout URL exacta.
- Mínimos flujos y atributos necesarios.
- Token revocation habilitado cuando corresponda.

## 4. PKCE

- Método obligatorio: `S256`.
- `codeVerifier` solo vive en la aplicación cliente.
- Solo `codeChallenge` viaja al inicio.
- El exchange valida PKCE antes del consumo definitivo.
- `codeVerifier` no debe registrarse en logs.

## 5. State y nonce

- Deben ser aleatorios y de un solo flujo.
- El cliente valida `state` antes del exchange.
- El backend conserva el state autoritativo.
- `nonce` debe asociarse a la transacción y al ID Token cuando aplique.
- No deben aparecer completos en logs o documentos.

## 6. transactionKey

- Generada aleatoriamente por backend.
- Entregada una sola vez a MITTA.
- Guardada temporalmente en `sessionStorage`.
- Nunca incluida en URLs.
- Nunca registrada en logs.
- DynamoDB guarda solo SHA-256.
- Comparación en tiempo constante.

## 7. Handoff code

- Aleatorio.
- Vida corta.
- Un solo uso.
- DynamoDB guarda solo `codeHash`.
- Vinculado a `transactionId`.
- Debe consumirse atómicamente con la finalización de la transacción.

## 8. Tokens

- Verificar firma.
- Verificar issuer.
- Verificar expiración.
- Verificar `token_use`.
- Verificar `aud` del ID Token.
- Verificar `client_id` del Access Token.
- Verificar `sub` consistente.
- Cifrar en reposo temporal con AES-256-GCM.
- Nunca incluir tokens en query strings.
- Nunca registrar tokens completos.

## 9. DynamoDB

- Usar condiciones de estado y versión.
- Usar `TransactWriteItems` para cambios entre tablas.
- Validar `expiresAt` en la aplicación.
- TTL solo realiza limpieza eventual.
- Consultas de soporte deben proyectar campos no sensibles.

No mostrar:

```text
transactionKeyHash
state
nonce
codeChallenge
codeHash
encryptedTokens
```

## 10. Logs

Permitido:

- código de error;
- ruta;
- método;
- applicationId;
- environment;
- estado;
- versión;
- timestamps;
- request ID.

Prohibido:

- passwords;
- tokens;
- transactionKey;
- codeVerifier;
- handoff code;
- claves;
- datos completos de usuario.

## 11. CORS

- No confiar solo en CORS como control de seguridad.
- Permitir orígenes explícitos.
- Separar rutas administrativas y públicas.
- Resolver `allowedOrigin` desde catálogo dinámico.
- Evitar `*` cuando existan credenciales o datos sensibles.

## 12. API administrativa

- Access Token válido.
- `token_use=access`.
- Client ID administrativo esperado.
- Grupo Cognito requerido.
- Control de versión.
- Auditoría de cambios.

## 13. Git y documentación

Nunca versionar:

```text
.env
.env.local
archivos con tokens
archivos de contexto con secretos
backups con credenciales
```

Mantener `.env.example` sin valores reales.

## 14. Sesión de aplicaciones

Demo:

- `sessionStorage` aceptable para pruebas.

Producción recomendada:

- Backend for Frontend.
- Tokens del lado servidor.
- Cookie `HttpOnly`.
- `Secure`.
- `SameSite` apropiado.
- Rotación y revocación.

## 15. Respuesta ante exposición

1. Revocar o rotar la credencial.
2. Cerrar sesiones si aplica.
3. Eliminar el dato de logs e historial.
4. Revisar Git y cachés.
5. Documentar incidente.
6. Confirmar que el valor anterior ya no funciona.
