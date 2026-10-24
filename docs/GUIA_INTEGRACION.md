# ChameleonOps — Guía de Integración y Despliegue

**Materia:** Aplicación de Sistemas Operativos  
**Sistema operativo:** openSUSE Leap 16.0  
**Tecnología:** Ansible, SSH, Git y GitHub  
**Repositorio:** https://github.com/Ghostcrist/ChameleonOps

## 1. Objetivo del proyecto

Implementar una infraestructura de cuatro máquinas virtuales administradas de manera centralizada mediante Ansible.

La VM 01 actuará como nodo de control (Master), ejecutando los playbooks y roles que instalan y configuran automáticamente los servicios de las otras tres máquinas.

El proyecto debe demostrar automatización, seguridad, colaboración mediante GitHub, reutilización de configuraciones y optimización de los despliegues.

## 2. Arquitectura de máquinas virtuales

| Máquina         | Grupo Ansible   | Servicios                                                    |
|-----------------|-----------------|--------------------------------------------------------------|
| VM01—Master     | Nodo de control | Ansible, Git, inventario, roles, templates y playbooks       |
| VM02—Web        | `web`           | Apache, PHP, phpMyAdmin, MariaDB y librería PHP de PostgreSQL|
| VM03—Archivos   | `archivos`      | Samba, FTP, usuarios, permisos y recursos compartidos        |
| VM04—PostgreSQL | `postgres`      | PostgreSQL, usuarios, base de datos, acceso remoto y TLS     |

**Importante:** Todas las máquinas utilizan openSUSE Leap 16.0 y deben tener conectividad entre sí.
 cada VM debe tener su propia dirección IP.

## 3. Requisitos previos

Antes de ejecutar cualquier despliegue, verificar:

- Las cuatro máquinas están encendidas y tienen acceso a la red.
- VMware está configurado con adaptador Bridged.
- Cada máquina posee una dirección IP válida y diferente.
- SSH y Python están disponibles en las máquinas destino.
- Existe un usuario `ansible` con permisos administrativos adecuados en los destinos.
- La Master tiene acceso mediante claves SSH.
- La Master contiene la versión actualizada del repositorio ChameleonOps.

### 3.1. Verificación del sistema operativo

Ejecutar en cada VM:

`cat /etc/os-release`

Comprobar que `VERSION_ID` corresponde a `16.0`.

### 3.2. Verificación de direcciones IP

En cada VM:

`ip -4 addr`

Registrar las direcciones reales:

| Máquina | Dirección IP  |
|---------|---------------|
| VM 01   | Por confirmar |
| VM 02   | Por confirmar |
| VM 03   | Por confirmar |
| VM 04   | Por confirmar |

Las direcciones pueden cambiar cuando se utiliza DHCP. Siempre deben comprobarse antes de actualizar el inventario.

## 4. Conectividad y autenticación SSH

### 4.1. Comprobar conectividad de red

Desde la Master:

`ping -c 4 IP_VM02`

`ping -c 4 IP_VM03`

`ping -c 4 IP_VM04`

Sustituir cada marcador por la dirección IP real.

Si alguna máquina no responde, comprobar red, adaptador VMware y firewall. El bloqueo de ICMP no siempre implica que SSH esté inaccesible.

### 4.2. Preparar el servicio SSH

En cada destino, verificar:

`sudo systemctl status sshd`

El servicio debe estar activo y accesible desde la Master.

### 4.3. Comprobar autenticación mediante clave pública

La Master utiliza la clave pública del usuario local `cristian`.

Si es necesario instalarla en un nuevo destino:

`ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@IP_DESTINO`

Antes de aceptar por primera vez la identidad del servidor, contrastar su huella SSH con la proporcionada por el responsable de esa máquina.

Comprobar el ingreso utilizando únicamente la clave:

`ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no ansible@IP_DESTINO`

Una vez dentro, verificar:

`whoami`

`hostname`

El usuario remoto debe ser `ansible`.

Si el servidor solicita la contraseña del usuario remoto o rechaza la autenticación por clave, revisar `authorized_keys`, permisos, configuración efectiva de SSH y registros de `sshd`.

No desactivar la verificación de claves del servidor para resolver problemas de autenticación.

## 5. Preparación de la Master

El proyecto oficial está almacenado en:

`~/proyectos/Ansible_ASO/ChameleonOps_actual`

Entrar al proyecto:

`cd ~/proyectos/Ansible_ASO/ChameleonOps_actual`

Actualizar la rama principal:

`git switch main`

`git pull --ff-only origin main`

Comprobar Ansible:

`ansible --version`

Comprobar la configuración utilizada:

`ansible-config dump --only-changed`

### 5.1. Configurar el inventario

Editar:

`nano hosts`

Asignar las direcciones IP reales a los grupos:

```
[web]
vm02 ansible_host=IP_VM02

[archivos]
vm03 ansible_host=IP_VM03

[postgres]
vm04 ansible_host=IP_VM04
```

Los marcadores deben sustituirse antes de ejecutar las conexiones remotas.

Verificar la estructura:

`ansible-inventory --graph --ask-vault-pass`

### 5.2. Variables sensibles

Las credenciales deben manejarse con Ansible Vault y no guardarse en texto visible en el inventario.

Ejemplo de edición de un archivo cifrado:

`ansible-vault edit host_vars/vm02/vault.yml`

No publicar contraseñas reales, claves privadas ni archivos locales de Vault excluidos por `.gitignore`.

## 6. Pruebas iniciales de integración

Las pruebas deben ejecutarse desde la VM 01, después de comprobar SSH.

### 6.1. Prueba de conectividad Ansible

Ejecutar:

`ansible-playbook 00_conectividad.yml --ask-vault-pass`

**Resultado esperado:** tres respuestas `pong`, correspondientes a `vm02`, `vm03` y `vm04`.

### 6.2. Verificación de privilegios sudo

Ejecutar inicialmente de forma individual:

`ansible-playbook 01_permisos_sudo.yml --limit vm02 -K --ask-vault-pass`

Repetir para `vm03` y `vm04`.

Si las credenciales de elevación ya están definidas correctamente en Vault, la opción `-K` puede no ser necesaria.

**Resultado esperado:** el usuario remoto puede elevar privilegios y ejecutar `id -u` con resultado `0`.

### 6.3. Diagnóstico del sistema

Ejecutar:

`ansible-playbook 02_diagnostico.yml --ask-vault-pass`

**Resultado esperado:** información de distribución, versión, arquitectura, alias e IP de inventario de los tres servidores.

Las pruebas de sintaxis y `--list-hosts` no sustituyen estas pruebas remotas.

## 7. Despliegue de servicios

El archivo principal del proyecto es:

`site.yml`

Este playbook ejecuta los roles según el grupo de destino.

### 7.1. Validar la sintaxis

`ansible-playbook site.yml --syntax-check`

La validación sintáctica no garantiza que los servicios se instalen correctamente.

### 7.2. Despliegue individual

Primero desplegar y verificar cada máquina por separado.

Servidor web:

`ansible-playbook site.yml --limit vm02 -K --ask-vault-pass`

Servidor de archivos:

`ansible-playbook site.yml --limit vm03 -K --ask-vault-pass`

Servidor PostgreSQL:

`ansible-playbook site.yml --limit vm04 -K --ask-vault-pass`

Las opciones de autenticación deben ajustarse a las credenciales configuradas para cada destino.

### 7.3. Despliegue completo

Una vez probados todos los roles y resueltas las credenciales administrativas:

`ansible-playbook site.yml --ask-vault-pass -K`

Si las VM utilizan contraseñas `sudo` diferentes, configurar antes su gestión individual mediante Vault.

No realizar este despliegue completo hasta que los roles de los compañeros hayan sido revisados.

## 8. Validación de los servicios

Después del despliegue, los responsables deben demostrar:

**VM 02 — Servidor web**

- Apache funciona y responde a solicitudes HTTP.
- PHP funciona con el servidor web.
- phpMyAdmin está instalado y accesible de forma controlada.
- MariaDB está activo y configurado.
- La librería PHP para PostgreSQL está disponible.

**VM 03 — Servidor de archivos**

- Samba está activo.
- Los usuarios y grupos necesarios existen.
- Los recursos compartidos tienen permisos adecuados.
- La configuración incluye `create mask`.
- FTP permite transferencias autorizadas.

**VM 04 — PostgreSQL**

- PostgreSQL está instalado y activo.
- Existen la base de datos y el usuario solicitados.
- Las conexiones remotas están limitadas a clientes autorizados.
- La autenticación utiliza SCRAM-SHA-256 según lo configurado.
- TLS está configurado y las conexiones cifradas se verifican.
- La VM 02 puede realizar una conexión autorizada a PostgreSQL.

### 8.1. Validación automática de servicios

El proyecto incorpora el playbook `03_validacion_servicios.yml` para comprobar el estado de los servicios de las máquinas destino.
Este playbook utiliza `ansible.builtin.service_facts` y no realiza instalaciones ni modificaciones.

**Ejecución informativa:**

```bash
ansible-playbook 03_validacion_servicios.yml --ask-vault-pass -K
```

Muestra el estado de Apache y MariaDB en VM 02, Samba y FTP en VM 03, y PostgreSQL en VM 04.

**Ejecución estricta:**

```bash
ansible-playbook 03_validacion_servicios.yml --ask-vault-pass -K -e validacion_estricta=true
```

En modo estricto, la validación falla si alguno de los servicios esperados no está activo.

Si los destinos utilizan contraseñas `sudo` diferentes, se deben ejecutar las pruebas por separado utilizando `--limit vm02`,
`--limit vm03` o `--limit vm04`, o configurar las credenciales individuales mediante Vault.

Los nombres de los servicios deben confirmarse con los roles implementados en cada máquina.

Esta validación comprueba el estado de los servicios, pero no sustituye las pruebas funcionales de PHP, MariaDB, recursos Samba,
transferencias FTP ni conexiones TLS de PostgreSQL.

## 9. Optimización e idempotencia

Una vez que el despliegue completo funcione, ejecutarlo nuevamente.

Comparar los valores reportados por Ansible:

- `ok`
- `changed`
- `failed`
- `unreachable`

La segunda ejecución debería evitar modificaciones innecesarias cuando los servidores ya están correctamente configurados.

Los roles deben utilizar handlers para reiniciar servicios solo cuando sus configuraciones cambien.

También se deben evitar actualizaciones repetidas e innecesarias de Zypper y comprobar los posibles bloqueos provocados por PackageKit.

Registrar los tiempos de ejecución para evaluar las mejoras.

## 10. Trabajo colaborativo con GitHub

Cada integrante trabaja en su rama:

| Rama               | Responsabilidad                              |
|--------------------|----------------------------------------------|
| `main`             | Versión oficial e integrada                  |
| `feature/master`   | Administración e integración desde la Master |
| `feature/web`      | Apache, PHP y MariaDB                        |
| `feature/archivos` | Samba y FTP                                  |
| `feature/postgres` | PostgreSQL                                   |

Procedimiento de colaboración:

1. Actualizar `main`.
2. Trabajar en la rama correspondiente.
3. Implementar y comprobar los cambios.
4. Crear un commit.
5. Publicar la rama.
6. Abrir un Pull Request hacia `main`.
7. Revisar y fusionar los cambios.
8. Actualizar la Master desde `main`.

No subir contraseñas, claves privadas ni configuraciones sensibles en texto plano.

## 11. Evidencias para la presentación

Se recomienda registrar:

- Estado de las cuatro máquinas y sus direcciones IP.
- Comunicación SSH y prueba de autenticación por clave.
- Inventario Ansible con los tres destinos.
- Ejecución de los playbooks de conectividad y diagnóstico.
- Ejecución del playbook principal.
- Comprobación de Apache, MariaDB, Samba, FTP y PostgreSQL.
- Conexión PostgreSQL remota cifrada.
- Comparación entre la primera y segunda ejecución.
- Historial de commits y Pull Requests del repositorio.

## 12. Criterio de finalización

El proyecto se considerará listo para su presentación cuando los cinco roles estén implementados,
los tres destinos puedan ser administrados desde la Master y se haya verificado el funcionamiento de los servicios solicitados.

**Estado actual:** infraestructura Master y repositorio preparados. 
