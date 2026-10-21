# Paso 2. Configurar Apache Reverse Proxy para Keycloak

## Objetivo

Publicar Keycloak 26.7.4 mediante Apache HTTPS en la dirección:

**`https://key00.planeta0.com/auth0/`**

Keycloak ya está ejecutándose en Docker y responde localmente en **`http://127.0.0.1:8084/auth0/`**. Apache será el punto de entrada por el puerto **443**, sin necesidad de modificar los registros DNS ni de exponer el puerto 8084 a los navegadores.

> **Importante:** este procedimiento modifica **Configure Apache SSL Website** (`key00.planeta0.com:443`). **No** modifiques **Configure Apache Website** (`key00.planeta0.com:80`).

## 2.1. Comprobar Keycloak antes de modificar Apache

Desde una sesión SSH en AlmaLinux, ejecuta:

```bash
curl -I http://127.0.0.1:8084/auth0/
```

La respuesta que ya se obtuvo en este laboratorio fue:

```http
HTTP/1.1 302 Found
Location: https://key00.planeta0.com/auth0/admin/
```

**Interpretación:** Keycloak responde en el puerto 8084 y redirige hacia la URL pública prevista. Si ahora no responde, comprueba primero el contenedor:

```bash
docker ps --filter name=keycloak-dev
docker logs --tail 80 keycloak-dev
```

No continúes con la configuración del proxy si Keycloak no está disponible localmente.

## 2.2. Abrir la configuración HTTPS correcta en Virtualmin

1. Abre **`https://key00.planeta0.com:10000/`** en el navegador e inicia sesión en Webmin/Virtualmin.
2. Selecciona la pestaña **Virtualmin**.
3. En el panel izquierdo, selecciona el servidor virtual **`key00.planeta0.com`**.
4. Despliega **Web Configuration**.
5. Selecciona **Configure Apache SSL Website**. **No** selecciones **Configure Apache Website**, porque corresponde al puerto 80.
6. En la pantalla de configuración SSL de Apache, abre **Edit Directives**.
7. Comprueba el encabezado: debe indicar **`For key00.planeta0.com:443`**. Si dice `:80`, vuelve atrás y selecciona la opción SSL.

## 2.3. Incorporar las directivas del proxy

En el editor de directivas del servidor HTTPS, localiza la directiva existente:

```apache
ProxyPass /.well-known !
```

**Inmediatamente debajo**, agrega el siguiente bloque, sin borrar las líneas que ya existen:

```apache
# Keycloak 26.7.4 - Apache Reverse Proxy
# URL publica: https://key00.planeta0.com/auth0/
# Backend local: http://127.0.0.1:8084/auth0/

ProxyPreserveHost On

RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

ProxyPass        /auth0/ http://127.0.0.1:8084/auth0/
ProxyPassReverse /auth0/ http://127.0.0.1:8084/auth0/

RedirectMatch 302 ^/auth0$ /auth0/
```

**Asegúrate de que:**

- Estás editando el servidor HTTPS **`:443`**, no el HTTP **`:80`**.
- Las directivas quedan **dentro del VirtualHost HTTPS existente**; no agregues otro bloque `<VirtualHost>`.
- Conservas las directivas de certificados, redirecciones y demás servicios que ya configuró Virtualmin.
- El destino del proxy es `127.0.0.1:8084` y **no** `127.0.0.1:8080`: el puerto 8084 es el publicado por Docker en el host.
- Mantienes la barra final `/` en ambos extremos de las directivas `ProxyPass` y `ProxyPassReverse`.

Guarda los cambios desde Virtualmin.

## 2.4. Validar y recargar Apache

En tu sesión SSH, ejecuta:

```bash
sudo apachectl configtest
```

El resultado correcto es:

```text
Syntax OK
```

**Si no aparece `Syntax OK`, no recargues Apache.** Revisa el mensaje de error y corrige las directivas en Virtualmin.

Si la sintaxis es correcta, recarga Apache:

```bash
sudo systemctl reload httpd
```

Comprueba que continúe activo:

```bash
systemctl is-active httpd
```

Debe responder:

```text
active
```

## 2.5. Comprobar el acceso externo por HTTPS

Desde el servidor, ejecuta:

```bash
curl -I https://key00.planeta0.com/auth0/
```

Se espera una respuesta HTTP de redirección, similar a:

```http
HTTP/1.1 302 Found
Location: https://key00.planeta0.com/auth0/admin/
```

Después abre en el navegador:

**`https://key00.planeta0.com/auth0/admin/`**

Confirma que carga la consola de administración de Keycloak. Utiliza las credenciales de administración que definiste al crear el contenedor.

> Si la validación TLS falla por un problema de confianza del certificado, investígalo por separado. Como diagnóstico puntual puedes repetir `curl` con `-k`, pero la prueba definitiva debe funcionar sin omitir la validación TLS.

## 2.6. Diagnóstico si no funciona

### Apache devuelve 503 Service Unavailable

Verifica de nuevo el backend:

```bash
curl -I http://127.0.0.1:8084/auth0/
```

Inspecciona el registro de errores correspondiente al sitio:

```bash
sudo tail -n 60 /var/log/virtualmin/key00.planeta0.com_error_log
```

Si Keycloak responde localmente, pero Apache no puede conectarse, comprueba SELinux:

```bash
getenforce
getsebool httpd_can_network_connect
```

Si SELinux está en `Enforcing` y los registros indican una denegación de conexión de Apache hacia el backend, habilita la conexión saliente:

```bash
sudo setsebool -P httpd_can_network_connect on
```

Vuelve a probar la URL HTTPS.

### Apache devuelve 404 Not Found

Comprueba que las directivas se guardaron en **Configure Apache SSL Website**, no en la configuración HTTP. Revisa que las rutas sean exactamente `/auth0/`.

### Apache devuelve un error de sintaxis

Ejecuta:

```bash
sudo apachectl configtest
```

Corrige el error indicado antes de recargar el servicio.

### Keycloak redirige a una URL inesperada

Comprueba que el contenedor se haya iniciado con las opciones de ruta y hostname apropiadas para este laboratorio:

```text
--http-relative-path=/auth0
--hostname=https://key00.planeta0.com/auth0
--proxy-headers=xforwarded
```

No cambies estos parámetros si la respuesta local ya redirige correctamente y el acceso externo funciona.

## 2.7. Resultado esperado

El flujo de acceso debe quedar así:

```text
Navegador del alumno
    |
    | HTTPS https://key00.planeta0.com/auth0/
    v
Apache (VirtualHost HTTPS, puerto 443)
    |
    | ProxyPass /auth0/
    v
127.0.0.1:8084 (puerto publicado en AlmaLinux)
    |
    v
Contenedor Docker Keycloak (puerto interno 8080)
```

**Criterio de término:** desde un navegador se puede abrir `https://key00.planeta0.com/auth0/admin/` y acceder a la consola de Keycloak, sin indicar el puerto 8084 en la URL pública.

---

**Para otros servidores del curso:** sustituye `key00.planeta0.com` por el FQDN que corresponda a cada servidor y verifica que el contenedor de ese servidor publique el mismo puerto local antes de replicar la configuración.
