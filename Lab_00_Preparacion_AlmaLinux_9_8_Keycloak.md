# Laboratorio 00 — Preparación del servidor AlmaLinux 9.8

**Curso:** Administración Keycloak  

**Creado por:** Patricio Canelo  

**Modalidad:** Laboratorio individual  
**Servidor:** AlmaLinux 9.8 · 6 vCPU · 16 GB RAM  
**Accesos disponibles:** SSH, Webmin y Virtualmin  
**Estado inicial:** dispones de la cuenta `root`, la IP pública y el FQDN de tu servidor.

## Objetivo

Dejar tu servidor preparado para los siguientes laboratorios del curso: actualizar AlmaLinux, verificar el nombre DNS, trabajar con HTTPS en Webmin, revisar Apache/Virtualmin, crear un usuario administrativo y utilizar **Docker Engine y Docker Compose sin `sudo`** desde ese usuario.

> **Cómo trabajar:** realiza los pasos en orden. Los comandos de los pasos 1 a 7 se ejecutan como `root`, salvo indicación contraria. A partir del paso 8 trabajarás con tu nuevo usuario. No instales Keycloak todavía.

## Paso 1. Acceder al servidor

1. Abre la terminal de tu equipo.
2. Conéctate por SSH (reemplaza `TU_IP_PUBLICA`):

```bash
ssh root@TU_IP_PUBLICA
```

3. Abre Webmin en tu navegador usando el FQDN o la IP y el puerto `10000`:

```text
https://TU_FQDN:10000
https://TU_IP_PUBLICA:10000
```

4. Accede con tus credenciales. Desde Webmin puedes pasar a Virtualmin usando su menú.
5. Si aún conservas la contraseña inicial de `root`, cámbiala:

```bash
passwd
```

**Comprobación:** puedes iniciar sesión por SSH y entrar en Webmin.

## Paso 2. Actualizar AlmaLinux

Ejecuta:

```bash
cat /etc/almalinux-release
uname -a
dnf update -y
```

Si se actualiza el kernel, reinicia:

```bash
reboot
```

Vuelve a conectarte por SSH y comprueba:

```bash
uname -r
cat /etc/almalinux-release
```

> La actualización puede llevar AlmaLinux a una versión menor posterior a 9.8 si los repositorios ya la publican. Para este laboratorio no necesitas mantener obligatoriamente el número 9.8 después de actualizar.

## Paso 3. Verificar o configurar el FQDN

Ya conoces el FQDN asignado a tu servidor. Verifica cómo está configurado:

```bash
hostnamectl
hostname -f
```

Si el FQDN mostrado **ya es el tuyo**, no lo cambies. Si es distinto, ejecuta el siguiente comando sustituyendo el valor de ejemplo:

```bash
hostnamectl set-hostname alumno00.planeta0.com
```

> En todos los comandos de este laboratorio, sustituye `alumno00.planeta0.com` por **tu FQDN real**. No copies literalmente el FQDN de ejemplo.

Ahora inspecciona el archivo de nombres locales:

```bash
cat /etc/hosts
```

Si falta una entrada que relacione tu IP pública con tu FQDN, edita el archivo:

```bash
nano /etc/hosts
```

Como referencia, conserva `localhost` y añade **una sola línea** con tu IP y tu FQDN:

```text
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
TU_IP_PUBLICA   alumno00.planeta0.com alumno00
```

**Importante:** no dupliques el FQDN en `127.0.0.1` y en la IP pública. Conserva otras entradas existentes que sean necesarias para servicios instalados. Si la asociación correcta ya existe, déjala intacta.

Guarda con `Ctrl+O`, `Enter` y `Ctrl+X`. Verifica:

```bash
hostname -f
getent hosts "$(hostname -f)"
exec bash
```

Comprueba también la resolución DNS pública (se instalará `dig` si hace falta):

```bash
dnf install -y bind-utils
dig +short "$(hostname -f)" A
```

**Resultado esperado:** el FQDN es el que tienes asignado y el registro DNS `A` devuelve la IP pública del servidor. Si aparecen varias direcciones, revisa que esté la asignada.

## Paso 4. Instalar Certbot y comprobar el certificado HTTPS
## Paso 4. Instalar Certbot y comprobar el certificado HTTPS

Instala EPEL y Certbot:

```bash
dnf install -y epel-release
dnf install -y certbot
```

Comprueba si el servidor ya cuenta con un certificado de Let's Encrypt:

```bash
certbot certificates
```

Si existe un certificado vigente que incluya tu FQDN, utilízalo. No solicites otro innecesariamente.

Si no existe un certificado para tu FQDN:

1. Comprueba que el FQDN resuelve a tu IP pública:

```bash
dig +short "$(hostname -f)" A
```

2. Obtén el certificado mediante el método `standalone`.

Este procedimiento interrumpe temporalmente Apache y requiere que Let's Encrypt pueda acceder al puerto TCP 80 del servidor.

```bash
systemctl stop httpd
certbot certonly --standalone -d "$(hostname -f)"
systemctl start httpd
```

**Importante:** ejecuta `systemctl start httpd` incluso si Certbot falla. No detengas Webmin ni Virtualmin.

3. Verifica el resultado:

```bash
certbot certificates
systemctl is-active httpd
```

Los certificados se encuentran normalmente en:

```text
/etc/letsencrypt/live/TU_FQDN/fullchain.pem
/etc/letsencrypt/live/TU_FQDN/privkey.pem
```

Utiliza el **Certificate Name** mostrado por Certbot para identificar el directorio real.

## Paso 5a. Asignar el certificado a Virtualmin y Apache

Virtualmin administra los certificados de los servidores virtuales y sus correspondientes VirtualHosts de Apache.

Para instalar el certificado existente de Let's Encrypt, ejecuta:

```bash
virtualmin install-cert \
  --domain "$(hostname -f)" \
  --cert "/etc/letsencrypt/live/TU_FQDN/fullchain.pem" \
  --key "/etc/letsencrypt/live/TU_FQDN/privkey.pem"
```

Sustituye `TU_FQDN` por el directorio real identificado con `certbot certificates`. El nombre utilizado en `--domain` debe corresponder al servidor virtual configurado en Virtualmin.

Este comando:

- Importa el certificado firmado por Let's Encrypt.
- Importa su clave privada correspondiente.
- Actualiza los archivos administrados por Virtualmin.
- Integra el certificado con la configuración del servidor virtual.

**Verifica Apache:**

```bash
apachectl configtest
```

Si el resultado es `Syntax OK`, ejecuta:

```bash
systemctl reload httpd
```

**Comprueba el certificado utilizado por Apache:**

```bash
echo | openssl s_client \
  -connect 127.0.0.1:443 \
  -servername "$(hostname -f)" \
  2>/dev/null | openssl x509 \
  -noout -subject -issuer -dates
```

El emisor debe corresponder a Let's Encrypt.

Comprueba también la cadena de confianza:

```bash
openssl s_client \
  -connect 127.0.0.1:443 \
  -servername "$(hostname -f)" \
  -verify_return_error </dev/null 2>&1 | \
  grep -E 'Verify return code|Verification error|depth='
```

El resultado esperado es:

```text
Verify return code: 0 (ok)
```

Finalmente, comprueba el acceso:

```text
https://TU_FQDN/
```

> **Nota:** es normal que Virtualmin siga mostrando certificados almacenados en `/etc/ssl/virtualmin/`. Virtualmin mantiene sus propios archivos y no necesita conservar las rutas originales de Certbot.

## Paso 5b. Asignar el certificado a Webmin

Webmin utiliza su propio servicio HTTPS, normalmente en el puerto 10000, independiente de Apache.

1. Ingresa en:

```text
https://TU_FQDN:10000
```

2. Abre **Webmin → Webmin Configuration → SSL Encryption**.

3. Localiza la configuración de los archivos SSL del servidor Webmin.

4. Establece los siguientes archivos:

   - **Private key file:** `/etc/letsencrypt/live/TU_FQDN/privkey.pem`
   - **Certificate file:** selecciona **Separate file** y utiliza `/etc/letsencrypt/live/TU_FQDN/fullchain.pem`.

5. Guarda la configuración y comprueba nuevamente el acceso:

```text
https://TU_FQDN:10000
```

**Comprobación:** el navegador debe mostrar un certificado válido para el FQDN.

Si Webmin ya utiliza correctamente un certificado válido, conserva su configuración.

> Acceder mediante `https://TU_IP_PUBLICA:10000` puede provocar una advertencia, porque un certificado emitido para un FQDN no necesariamente incluye la dirección IP.

## Consideración importante: renovación automática

Certbot y Virtualmin tienen mecanismos de administración y renovación de certificados independientes.

El comando `virtualmin install-cert` importa el certificado existente, pero no garantiza que las copias administradas por Virtualmin se actualicen cuando Certbot renueve el certificado original.

Por lo tanto, antes de utilizar esta configuración de forma permanente, se debe establecer una estrategia de renovación:

- **Alternativa recomendada:** permitir que Virtualmin gestione la emisión y renovación del certificado desde **Setup SSL Certificate → SSL Providers**, evitando administraciones independientes.
- **Alternativa con Certbot:** conservar Certbot como responsable de la renovación y configurar un *deploy hook* que vuelva a ejecutar `virtualmin install-cert` después de cada renovación exitosa. También debe comprobarse la recarga de los servicios correspondientes.

No es necesario solicitar un certificado nuevo para realizar la instalación inicial descrita anteriormente.



## Paso 6. Revisar Apache y deshabilitar el listado de directorios

Verifica que Apache funciona:

```bash
systemctl is-active httpd
httpd -S
```

Revisa si existe alguna directiva que habilite el listado de directorios:

```bash
grep -RIn 'Options.*Indexes' /etc/httpd/conf /etc/httpd/conf.d 2>/dev/null
```

Si una configuración aplicable al sitio usa:

```apache
Options Indexes FollowSymLinks
```

edítala para retirar únicamente `Indexes`:

```apache
Options FollowSymLinks
```

Por ejemplo, si la directiva está en el archivo principal:

```bash
nano /etc/httpd/conf/httpd.conf
```

Antes de aplicar los cambios comprueba la sintaxis:

```bash
httpd -t
```

Si aparece `Syntax OK`, recarga Apache:

```bash
systemctl reload httpd
```

> No sustituyas ni elimines directivas `Options` de otros bloques sin comprobar a qué directorio corresponden. Si no se encuentra una directiva con `Indexes` para tu sitio, continúa.

## Paso 7. Verificar el servidor virtual en Virtualmin

1. Entra a Webmin y abre **Virtualmin**.
2. Busca en el selector de dominios un servidor virtual correspondiente a **tu FQDN**.
3. Si ya existe, **utilízalo**: no necesitas crearlo de nuevo.
4. Si no existe, selecciona **Create Virtual Server**, completa:
   - **Domain name:** tu FQDN.
   - **Administration password:** una contraseña segura para el usuario del dominio.
5. Pulsa **Create Server**.
6. Comprueba que tu dominio aparece en la lista y que Apache sigue activo.

Desde SSH ejecuta:

```bash
httpd -S
systemctl is-active httpd
```

**Resultado esperado:** tu servidor virtual está identificado en Virtualmin y su configuración de Apache está operativa. Virtualmin puede gestionar la configuración web y los certificados del dominio; no necesitas crear un segundo virtual host manualmente.

## Paso 8. Crear un usuario de trabajo con sudo

Hasta aquí has trabajado como `root`. Crea ahora el usuario con el que realizarás las actividades de Docker. Sustituye `alumno` por el nombre de usuario que elijas y ejecútalo **siempre de forma consistente** en este paso:

```bash
useradd -m -s /bin/bash alumno
passwd alumno
usermod -aG wheel alumno
```

Comprueba la pertenencia al grupo `wheel`:

```bash
id alumno
```

En AlmaLinux, el grupo `wheel` suele estar autorizado para usar `sudo`. Confírmalo sin modificar archivos innecesariamente:

```bash
grep -E '^%wheel[[:space:]]+ALL=' /etc/sudoers
```

Si esa línea no está habilitada, ejecuta:

```bash
visudo
```

Y habilita esta entrada (quitando `#` si existe):

```text
%wheel ALL=(ALL) ALL
```

Si utilizas autenticación SSH por clave y ya existe `/root/.ssh/authorized_keys`, permite el acceso con esa misma clave a tu usuario:

```bash
mkdir -p /home/alumno/.ssh
cp /root/.ssh/authorized_keys /home/alumno/.ssh/authorized_keys
chown -R alumno:alumno /home/alumno/.ssh
chmod 700 /home/alumno/.ssh
chmod 600 /home/alumno/.ssh/authorized_keys
```

> Omite la copia si `/root/.ssh/authorized_keys` no existe o te conectas exclusivamente con contraseña.

Ahora abre **otra sesión SSH** con el usuario nuevo:

```bash
ssh alumno@TU_IP_PUBLICA
```

Verifica el acceso a `sudo`:

```bash
whoami
sudo whoami
```

**Resultado esperado:** `whoami` devuelve tu usuario y `sudo whoami` devuelve `root`.

## Paso 9. Instalar Docker Engine y Docker Compose

**Desde este punto utiliza la sesión SSH de tu usuario, no la sesión de `root`.** Los comandos de instalación llevan `sudo`; después configurarás Docker para ejecutarlo sin `sudo`.

1. Actualiza los paquetes disponibles:

```bash
sudo dnf update -y
```

2. Comprueba si existen instalaciones previas:

```bash
rpm -qa | grep -Ei 'docker|podman|containerd|runc' || true
```

No elimines paquetes automáticamente. Si ya existe Docker Engine funcional y `docker compose` está disponible, puedes pasar directamente al paso 10.

3. Instala la utilidad para administrar repositorios:

```bash
sudo dnf install -y dnf-plugins-core
```

4. Agrega el repositorio oficial de Docker para RHEL:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

5. Instala Docker Engine, containerd, Buildx y el plugin de Compose:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

6. Activa e inicia Docker:

```bash
sudo systemctl enable --now docker
```

7. Verifica las versiones:

```bash
docker --version
docker compose version
sudo systemctl is-active docker
```

8. Ejecuta la primera prueba:

```bash
sudo docker run --rm hello-world
```

**Resultado esperado:** aparece un mensaje de bienvenida de Docker indicando que el motor pudo descargar y ejecutar una imagen.

## Paso 10. Utilizar Docker sin sudo

Tu usuario debe ejecutar Docker directamente, sin escribir `sudo`.

1. Crea el grupo `docker` si todavía no existe:

```bash
getent group docker || sudo groupadd docker
```

2. Añade tu usuario actual al grupo:

```bash
sudo usermod -aG docker "$(whoami)"
```

3. Cierra la sesión SSH y vuelve a conectarte **con tu usuario**. Alternativamente, ejecuta:

```bash
newgrp docker
```

4. Comprueba los grupos y ejecuta Docker **sin sudo**:

```bash
id -nG
docker ps
docker compose version
docker run --rm hello-world
```

**Resultado esperado:** los tres comandos de Docker funcionan sin `sudo`, y `id -nG` incluye `docker`.

> Para este laboratorio temporal se permite pertenecer al grupo `docker`. Ten presente que este grupo concede privilegios efectivos equivalentes a `root` sobre el servidor: utilízalo exclusivamente en tu VPS del curso.

## Paso 11. Reiniciar y comprobar la preparación del servidor

Reinicia desde tu usuario:

```bash
sudo reboot
```

Reconecta por SSH con **tu usuario operativo** y realiza el control final:

```bash
whoami
hostname -f
cat /etc/almalinux-release
systemctl is-active httpd
systemctl is-active docker
id -nG
docker --version
docker compose version
docker ps
docker run --rm hello-world
```

Abre también en el navegador:

```text
https://TU_FQDN:10000
```

Verifica que puedes acceder a Webmin y Virtualmin.

### Lista de comprobación final

- [ ] Accedí por SSH a mi servidor.
- [ ] Actualicé AlmaLinux y comprobé la versión instalada.
- [ ] Verifiqué o corregí mi FQDN y `/etc/hosts`.
- [ ] Mi FQDN resuelve a la IP pública asignada.
- [ ] Certbot está instalado y comprobé el certificado de mi FQDN.
- [ ] Webmin es accesible por HTTPS con mi FQDN.
- [ ] Apache está activo y revisé el listado de directorios.
- [ ] Tengo identificado mi servidor virtual en Virtualmin.
- [ ] Creé un usuario personal que puede ejecutar `sudo`.
- [ ] Docker Engine y Docker Compose están instalados.
- [ ] Mi usuario puede ejecutar `docker ps` y `docker compose version` **sin sudo**.
- [ ] Reinicié el VPS y todo sigue funcionando.

## Resultado del laboratorio

Tu servidor individual está preparado para los laboratorios de Keycloak. Continuarás trabajando con tu usuario personal y utilizarás Docker y Docker Compose para desplegar los servicios del curso.

**Fin del Laboratorio 00.**


