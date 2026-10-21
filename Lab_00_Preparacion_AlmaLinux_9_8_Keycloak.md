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

Instala EPEL y Certbot:

```bash
dnf install -y epel-release
dnf install -y certbot
```

Comprueba si tu servidor **ya cuenta** con un certificado de Let's Encrypt:

```bash
certbot certificates
```

Si encuentras un certificado **vigente** que incluya tu FQDN, úsalo y pasa al paso 5. No solicites otro innecesariamente.

Si **no tienes** un certificado para tu FQDN, realiza lo siguiente:

1. Comprueba que el FQDN resuelve a tu IP pública:

```bash
dig +short "$(hostname -f)" A
```

2. Obtén el certificado usando el método `standalone`. Como Apache suele ocupar el puerto 80, detén Apache durante la solicitud y vuelve a iniciarlo después:

```bash
systemctl stop httpd
certbot certonly --standalone -d "$(hostname -f)"
systemctl start httpd
```

Durante la ejecución, Certbot puede solicitar tu correo electrónico y la aceptación de sus condiciones.

> Ejecuta `systemctl start httpd` **aunque Certbot falle**. El método `standalone` necesita disponer temporalmente del puerto 80. No detengas Webmin ni Virtualmin.

3. Verifica el resultado:

```bash
certbot certificates
systemctl is-active httpd
```

Los certificados obtenidos con Certbot se encuentran normalmente en:

```text
/etc/letsencrypt/live/TU_FQDN/fullchain.pem
/etc/letsencrypt/live/TU_FQDN/privkey.pem
```

Usa el **Certificate Name** mostrado por `certbot certificates` para identificar el directorio real; puede no coincidir exactamente con el FQDN.

## Paso 5. Asignar el certificado a Webmin

1. Ingresa en `https://TU_FQDN:10000`.
2. Abre **Webmin → Webmin Configuration → SSL Encryption**.
3. Localiza la configuración de los archivos SSL del servidor Webmin (los nombres exactos pueden variar según la versión).
4. Establece los archivos del certificado ya obtenido:

   - **Private key file:** `/etc/letsencrypt/live/TU_FQDN/privkey.pem`
   - **Certificate file:** selecciona **Separate file** y utiliza `/etc/letsencrypt/live/TU_FQDN/fullchain.pem`.

   Sustituye `TU_FQDN` por el nombre del directorio de certificado **real** que comprobaste en el paso anterior.

5. Guarda la configuración y vuelve a abrir:

```text
https://TU_FQDN:10000
```

**Comprobación:** el navegador muestra un certificado válido para tu FQDN. Si Webmin ya estaba configurado con un certificado válido, conserva esa configuración.

> Acceder por `https://TU_IP_PUBLICA:10000` puede producir una advertencia de nombre de certificado: el certificado emitido para tu FQDN no necesariamente cubre la IP.

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
