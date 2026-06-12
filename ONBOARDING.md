# Guía: TAO CE local con SMOWL integrado por LTI 1.3

Esta guía replica el entorno montado en la EC2 de desarrollo: una instancia de **TAO Community Edition** (contenedor prebuilt) detrás de Apache, con la herramienta **LTI de SMOWL** local integrada como proveedor de proctoring externo de las sesiones. El objetivo es poder levantar entornos de prueba TAO+SMOWL para distintos stages.

> **Dominios usados** (ajústalos si tu stage usa otros):
> - TAO CE: `https://tao-local.smowltech.net`
> - Herramienta LTI de SMOWL: `https://lti-smowl-global-local.smowltech.net` (contenedor `lti-lti-1`, puerto 8080)

---

## 1. Requisitos previos

- EC2 con Docker + Docker Compose y Apache 2.4.47+ (probado con 2.4.58) con `mod_proxy`, `mod_proxy_http` y `mod_ssl`.
- Certificado wildcard `*.smowltech.net` en `/etc/apache2/ssl/` (cert.pem, privkey.pem, chain.pem).
- La herramienta LTI de SMOWL corriendo en la máquina (compose en `/var/www/lti-smowl-global-local.smowltech.net/lti`, expone el puerto 8080).
- Repositorio `tao-ce` clonado (esta guía asume `/var/www/tao-ce`). **No hace falta build desde fuentes**: se usa la imagen publicada `quay.io/tao-ce/tao-ce`.

### 1.1 /etc/hosts

En la **EC2** (sustituye la IP por la de tu máquina, `hostname -I`):

```text
192.168.8.75 tao-local.smowltech.net
192.168.8.75 lti-smowl-global-local.smowltech.net
```


En tu **PC local**, añade las mismas entradas apuntando a la IP de la EC2 para acceder desde el navegador.

---

## 2. Ficheros de configuración local de TAO CE

En la raíz del repo `tao-ce` se crean **tres ficheros untracked** que se montan en el contenedor vía la sección `configs:` de `docker-compose.yml`. Así los cambios sobreviven a recreaciones del contenedor.

### 2.1 `.env`

```bash
TAO_CE_PUBLIC_DOMAIN=tao-local.smowltech.net
```

⚠️ Si la instancia ya se aprovisionó con otro dominio, hay que borrar los volúmenes para re-aprovisionar: `docker compose down && docker volume rm tao-ce_es tao-ce_pgsql tao-ce_redis tao-ce_tao-ce-lib`.

### 2.2 `Caddyfile.local` — TLS interno

El ingress interno del contenedor es Caddy. Con un dominio que no existe en DNS público intenta obtener certificado por ACME/Let's Encrypt, falla con NXDOMAIN y **deja el puerto 443 sin servir TLS**. Solución: certificado de la CA interna de Caddy (Apache termina el TLS real y no verifica el del backend).

```bash
cp etc/tao-ce/config/Caddyfile Caddyfile.local
```

Y añade `tls internal` justo tras abrir el bloque del sitio:

```diff
 {$TAO_CE_PUBLIC_DOMAIN}:443 {
+	tls internal
+
 	@deliver-static path /deliver/sw.js /sw.js /deliver-fe-static/*
```

### 2.3 `environments.libsonnet.local` — proveedor SMOWL

Esta plantilla jsonnet genera la configuración de tenant (feature flags, tools y registrations LTI). Se parte de la del repo:

```bash
cp libexec/setup/apps/files/environments.libsonnet environments.libsonnet.local
```

Se aplican **tres cambios**:

**(a)** En el array `configurations`, tras la entrada `proctoring.registration_id`, añadir el proveedor externo (es lo que hace aparecer SMOWL al asignar proctoring a una sesión — sin esto la lista está vacía y la UI no muestra nada):

```jsonnet
{
  name: 'proctoring.external_providers',
  value: '[{"providerName":"SMOWL","clientId":"smowl-proctoring-client-id","settings":{},"launchClaims":{}}]',
},
```

**(b)** En el array `ltiTools`, añadir la tool de SMOWL (antes de la entrada `proctoring-tool`):

```jsonnet
{
  id: 'smowl-proctoring-tool',
  name: 'SMOWL Proctoring',
  audience: 'https://lti-smowl-global-local.smowltech.net',
  oidcInitiationUrl: 'https://lti-smowl-global-local.smowltech.net/lti/login',
  launchUrl: 'https://lti-smowl-global-local.smowltech.net/lti/launch',
  deepLinkingUrl: '',
},
```

**(c)** En el array `ltiRegistrations`, tras la registration `deliver--proctoring`, añadir:

```jsonnet
{
  id: 'portal--smowl-proctoring',
  clientId: 'smowl-proctoring-client-id',
  tenantId: '1',
  platformId: 'portal-platform',
  toolId: 'smowl-proctoring-tool',
  platformJwksUrl: '%s/.well-known/jwks.json' % setup.apps['environment-management'].auth_server.http.url,
  toolJwksUrl: 'https://lti-smowl-global-local.smowltech.net/lti/jkws/keys',
  platformKeyChain: {},
  toolKeyChain: {},
  deploymentIds: ['1'],
},
```

> Nota: el path del JWKS de la herramienta es `/lti/jkws/keys` (con el typo histórico `jkws`), es el endpoint real del servicio.

### 2.4 `sidecar-config.yaml.local` — whitelist del filtro de envoy

**El hallazgo clave de toda la integración.** El sidecar de environment-management filtra todas las rutas que pasan por envoy con una whitelist (`/opt/tao-ce/environment-management/sidecar/config.yaml`). Incluye `/lti1p3/oidc/authentication` **sin prefijo** porque en el TAO productivo de OAT deliver tiene dominio propio; en TAO CE todo vive bajo `/deliver/` y esa variante no está → el navegador del estudiante recibe "Access Denied" en mitad del handshake OIDC.

Extrae el fichero del contenedor (la primera vez, con el contenedor ya arrancado) y añade la ruta:

```bash
docker exec tao-ce-tao-1 cat /opt/tao-ce/environment-management/sidecar/config.yaml > sidecar-config.yaml.local
```

```diff
       # deliver endpoints with /deliver path prefix (for offline-vm)
       - /deliver/api/v1/asset
       - /deliver/api/v1/attachments
       - /deliver/api/v1/csv
       - /deliver/api/v1/download-asset
+      - /deliver/lti1p3/oidc/authentication
```

### 2.5 `docker-compose.yml`

Cambios sobre el del repo:

```diff
     ports:
-      - 443:443
+      - 8443:443
```

```yaml
services:
  tao:
    configs:
      - source: tao_config
        target: /etc/tao-ce/config/tao.yaml
        mode: 0644
      - source: caddyfile
        target: /etc/tao-ce/config/Caddyfile
        mode: 0644
      - source: environments
        target: /usr/local/libexec/tao-ce/setup/apps/files/environments.libsonnet
        mode: 0644
      - source: sidecar_config
        target: /opt/tao-ce/environment-management/sidecar/config.yaml
        mode: 0644

configs:
  caddyfile:
    file: ./Caddyfile.local
  environments:
    file: ./environments.libsonnet.local
  sidecar_config:
    file: ./sidecar-config.yaml.local
```

---

## 3. Apache (proxy inverso)

### 3.1 Límites globales de cabeceras

TAO mueve JWTs enormes en cookies y URLs (>8 KB, el límite por defecto). **Debe ir en ámbito global** — dentro del vhost no surte efecto porque Apache evalúa estos límites antes de seleccionar el virtual host:

```bash
printf 'LimitRequestFieldSize 32768\nLimitRequestLine 32768\n' | sudo tee /etc/apache2/conf-available/header-limits.conf
sudo a2enconf header-limits
```

### 3.2 Vhost de TAO — `/etc/apache2/sites-available/tao-local.smowltech.net.conf`

```apache
<VirtualHost *:443>
        ServerName tao-local.smowltech.net
        ServerAdmin webmaster@localhost
        SetEnv ENV dev
        SetEnv AWS_REGION eu-west-1

        CustomLog ${APACHE_LOG_DIR}/tao-local.smowltech.net_access.log filebeat
        ErrorLog  ${APACHE_LOG_DIR}/tao-local.smowltech.net_error.log

        SSLEngine on
        SSLCertificateFile      /etc/apache2/ssl/cert.pem
        SSLCertificateKeyFile   /etc/apache2/ssl/privkey.pem
        SSLCertificateChainFile /etc/apache2/ssl/chain.pem
        Header always set Strict-Transport-Security "max-age=15768000; includeSubDomains"

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia On

        # El backend (Caddy) sirve HTTPS con cert autofirmado de su CA interna
        SSLProxyEngine on
        SSLProxyVerify none
        SSLProxyCheckPeerCN off
        SSLProxyCheckPeerName off
        SSLProxyCheckPeerExpire off

        # upgrade=websocket: el test runner usa WebSockets (timers)
        # responsefieldsize: las redirecciones LTI llevan cabeceras Location >10KB
        ProxyPass        / https://127.0.0.1:8443/ upgrade=websocket responsefieldsize=65536
        ProxyPassReverse / https://127.0.0.1:8443/

        Timeout 305
</VirtualHost>
```

### 3.3 Vhost de la herramienta LTI

Al `ProxyPass` existente del vhost de `lti-smowl-global-local.smowltech.net` hay que añadirle `responsefieldsize=65536` (su redirección OIDC lleva una `Location` de ~10,7 KB; sin esto Apache devuelve 502 *"invalid response from upstream"*):

```apache
ProxyPass / http://127.0.0.1:8080/ responsefieldsize=65536
ProxyPassReverse / http://127.0.0.1:8080/
```

Activar todo:

```bash
sudo a2ensite tao-local.smowltech.net.conf
sudo apachectl configtest && sudo systemctl reload apache2
```

---

## 4. Arrancar TAO CE

```bash
cd /var/www/tao-ce
docker compose up -d
```

El primer arranque tarda varios minutos (aprovisiona PostgreSQL, Elasticsearch, usuarios, etc.). Verifica:

```bash
# Config con el dominio correcto
docker exec tao-ce-tao-1 grep publicDomain /etc/tao-ce/config/tao.yaml
# Portal responde
curl -sk -o /dev/null -w "%{http_code}\n" https://tao-local.smowltech.net/portal
```

Login: `admin` / `password` (cámbialo). Test-takers de demo: `demo01`–`demo05` / `password`.

### ⚠️ Carrera de arranque (gotcha importante)

`/etc/tao-ce` persiste entre *restarts* del contenedor y los servicios de environment-management cargan la configuración **en memoria al arrancar**. Si arrancan antes de que `tao-ce.setup.service` re-renderice las plantillas, sirven datos viejos. Tras cualquier cambio en `environments.libsonnet.local` + reinicio del contenedor, ejecuta:

```bash
docker exec tao-ce-tao-1 systemctl restart \
  tao-ce.environment-management.auth-server.service \
  tao-ce.environment-management.auth-server-gateway.service \
  tao-ce.environment-management.sidecar.service \
  tao-ce.portal.backend.service
```

(El `auth-server-gateway` es el que sirve las registrations LTI al portal y a deliver; el portal backend cachea la configuración.)

---

## 5. Alta de la plataforma TAO en la herramienta SMOWL

En el manager de la herramienta LTI, registra la plataforma con estos datos:

| Campo | Valor |
|---|---|
| Issuer / Platform ID | `https://tao-local.smowltech.net/portal-be` |
| **Auth Login URL** | `https://tao-local.smowltech.net/deliver/lti1p3/oidc/authentication` |
| Auth Token URL | `https://tao-local.smowltech.net/auth-server/v1/oauth2/tokens` |
| JWKS URL | `https://tao-local.smowltech.net/auth-server/.well-known/jwks.json` |
| Client ID | `smowl-proctoring-client-id` |
| Deployment ID | `1` |

> ⚠️ **El Auth Login URL es el de `/deliver/`, NO el de `/auth-server/`.** El endpoint del auth-server exige un parámetro `iss` no estándar en la authorization request que nuestra herramienta (correctamente, usa la librería estándar de OAT) no envía. El de deliver implementa el flujo estándar y resuelve la registration desde el `lti_message_hint`, igual que el TAO productivo.

El `id_token` de la plataforma se firma con `kid: oatInternal`, publicado en el JWKS de arriba.

---

## 6. Crear contenido y la sesión con SMOWL

1. **Contenido**: Portal → Content bank → crear items y test → Publish (delivery). Crear un Group con los test-takers y una Session que una delivery + group.
2. **Asignar SMOWL a la sesión**: la UI del portal de TAO CE **no tiene selector** de proctoring externo. El campo `externalProctoring` se asigna por importación CSV de sesiones (columna `session_externalProctoring`) o directamente en Elasticsearch (válido en entorno de pruebas):

```bash
docker exec tao-ce-tao-1 sh -c 'curl -s -X POST "http://es:9200/portal-enrolment/_update_by_query?refresh=true" \
  -H "Content-Type: application/json" \
  -d "{\"query\":{\"match\":{\"sessionId\":\"<SESSION_ID>\"}},\"script\":{\"source\":\"ctx._source.externalProctoring = params.p\",\"params\":{\"p\":\"SMOWL\"}}}"'
```

> El `<SESSION_ID>` está en la URL del portal al abrir la sesión. El update debe afectar a **todos** los documentos de la sesión (sesión + matrículas de usuarios): el claim del launch se lee de la matrícula del usuario, no del documento de la sesión.
> Importante: asigna el proctoring **antes** de que los takers entren; las sesiones creadas después solo necesitan este paso una vez.

3. **Probar**: entra con un taker (p. ej. `demo01`) y arranca el test. Flujo esperado:
   - Portal → deliver → redirección a `https://lti-smowl-global-local.smowltech.net/lti/login` (OIDC initiation, mensaje `LtiStartProctoring`)
   - SMOWL → 302 al Auth Login URL de la plataforma (`/deliver/lti1p3/oidc/authentication`)
   - TAO devuelve el formulario auto-submit con el `id_token` → POST a `/lti/launch` de SMOWL
   - SMOWL inicia el proctoring y redirige al `start_assessment_url` → el taker entra al examen monitorizado

---

## 7. Troubleshooting

| Síntoma | Causa | Solución |
|---|---|---|
| "Client sent an HTTP request to an HTTPS server" | Apache proxifica con `http://` al backend TLS | `ProxyPass https://` + `SSLProxyEngine on` (§3.2) |
| Puerto 8443 no responde (HTTP 000) | Caddy en bucle ACME por dominio sin DNS público | `tls internal` en `Caddyfile.local` (§2.2) |
| "Unexpected error happened during the test" + WS fallan en consola | Apache no tuneliza WebSockets | `upgrade=websocket` en ProxyPass (§3.2) |
| 400 "Size of a request header field exceeds server limit" | Cookies JWT >8KB y límites en el vhost (no aplican) | `header-limits.conf` **global** (§3.1) |
| 502 "invalid response from upstream" en `/lti/login` | `Location` de ~10,7KB supera el buffer del proxy | `responsefieldsize=65536` (§3.3) |
| 400 `INVALID_CONFIG` "Parameters 'iss' and 'client_id' are required" | Auth Login URL apuntando al auth-server | Usar el endpoint de `/deliver/` (§5) |
| "Access Denied" en `/deliver/lti1p3/oidc/authentication` | Ruta no incluida en la whitelist del sidecar | `sidecar-config.yaml.local` (§2.4) + restart sidecar |
| El taker va directo al test, sin SMOWL | La matrícula no tiene `externalProctoring` | Update en ES de **todos** los docs de la sesión (§6.2) |
| La config sirve valores viejos (p. ej. URLs placeholder) | Carrera de arranque: servicios EM cargaron antes del re-render | Restart de los servicios EM (§4) |
| La herramienta LTI no resuelve `tao-local.smowltech.net` | Falta/typo en `/etc/hosts` (y los contenedores no lo heredan) | §1.1; si la herramienta va en Docker, `extra_hosts` o DNS del host |

### Comandos útiles de diagnóstico

```bash
# Logs de todos los servicios de TAO (journald dentro del contenedor)
docker exec tao-ce-tao-1 journalctl --since "-10 min" --no-pager | grep -iE "error|denied"
# Traza de lanzamientos de un delivery
docker exec tao-ce-tao-1 journalctl --no-pager | grep audit_delivery_execution | tail
# Registration LTI que sirve el gateway (¿datos frescos?)
docker exec tao-ce-tao-1 curl -s "http://localhost:21101/api/v1/lti-registrations?platformIssuer=https%3A%2F%2Ftao-local.smowltech.net%2Fportal-be&clientId=smowl-proctoring-client-id"
# Logs de la herramienta LTI de SMOWL
docker logs lti-lti-1 --tail 50
```
