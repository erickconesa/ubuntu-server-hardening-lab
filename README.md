# Ubuntu Server Hardening Lab

Despliegue y hardening práctico de un servidor **Ubuntu Server** dentro de un entorno de virtualización anidada.

El objetivo del laboratorio no fue únicamente instalar un servidor Linux, sino también aplicar controles de seguridad, resolver incidencias reales de configuración y validar empíricamente cada cambio realizado.

## Descripción del proyecto

Este proyecto incluye:

- Virtualización anidada
- Despliegue de Ubuntu Server
- Gestión de usuarios y privilegios en Linux
- Políticas de bloqueo de cuentas con PAM
- Hardening de OpenSSH
- Autenticación mediante claves Ed25519
- Sincronización horaria con Chrony
- Actualizaciones automáticas de seguridad
- Validación de servicios y superficie de exposición
- Resolución de incidencias y comprobación de configuraciones efectivas

## Arquitectura del laboratorio

```text
Host físico
Windows Server 2022 Core
        │
        ▼
Hyper-V
        │
        ▼
Windows 11
        │
        ▼
VMware Workstation
        │
        ▼
Ubuntu Server
```

La máquina Windows 11 ya formaba parte de mi laboratorio personal, pero **VMware Workstation y la máquina Ubuntu Server utilizada en este proyecto se desplegaron y configuraron desde cero**.

Para poder ejecutar un segundo hipervisor dentro de la VM de Windows fue necesario habilitar y verificar correctamente las extensiones de virtualización anidada.

## Recursos de la máquina virtual

La máquina Ubuntu Server se configuró con:

```text
vCPU:       2
RAM:        2 GB
Disco:      60 GB SCSI
Almacenamiento: LVM
Red:        Bridge
SO:         Ubuntu Server LTS
```

Se mantuvo LVM para disponer de flexibilidad ante futuras ampliaciones del sistema de archivos o creación de nuevos volúmenes lógicos.

## Virtualización anidada

Uno de los primeros retos fue ejecutar VMware Workstation dentro de una máquina Windows ya virtualizada sobre Hyper-V.

El entorno mostró inicialmente problemas de compatibilidad con virtualización.

Para diagnosticarlo se verificó que las extensiones necesarias estaban disponibles dentro de la VM:

```powershell
Get-CimInstance Win32_Processor |
Select-Object Name,
              VirtualizationFirmwareEnabled,
              VMMonitorModeExtensions,
              SecondLevelAddressTranslationExtensions
```

En el host Hyper-V se habilitó la exposición de extensiones de virtualización:

```powershell
Set-VMProcessor -VMName "Win_11" `
    -ExposeVirtualizationExtensions $true
```

También se utilizó temporalmente una VM de prueba dentro de Hyper-V para confirmar que la virtualización anidada funcionaba correctamente.

Una vez validado este punto, VMware Workstation pudo ejecutar la máquina Ubuntu Server con normalidad.

## Gestión de usuarios y privilegios

La configuración del servidor sigue el **principio de mínimo privilegio**.

Las tareas administrativas habituales se realizan con cuentas no root y la elevación de privilegios se gestiona mediante `sudo`.

Se configuraron dos cuentas administrativas nominales para evitar depender del uso directo de `root`.

También se creó una política específica de sudoers:

```text
Defaults timestamp_timeout=5
Defaults passwd_tries=3
```

La sintaxis fue validada antes de utilizarla:

```bash
sudo visudo -cf /etc/sudoers.d/asir-admins
```

## Bloqueo de cuentas con PAM

Se configuró `pam_faillock` para reducir el riesgo de ataques de fuerza bruta sobre cuentas locales.

Política utilizada:

```text
deny = 5
fail_interval = 900
unlock_time = 900
```

Esto implica:

- Máximo de 5 intentos fallidos
- Ventana de observación de 15 minutos
- Bloqueo temporal de 15 minutos

Antes de modificar la pila PAM se realizaron copias de seguridad de los archivos de configuración afectados.

Posteriormente se realizó una prueba controlada de autenticación.

Tras cinco intentos incorrectos, el sistema bloqueó correctamente la cuenta.

En caso necesario, el contador puede restablecerse administrativamente con:

```bash
sudo faillock --user <usuario> --reset
```

## Hardening de SSH

OpenSSH se endureció de forma progresiva para evitar provocar un bloqueo administrativo accidental.

La configuración final quedó establecida con:

```text
Port 48222
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

El acceso SSH se realiza mediante **autenticación con clave pública Ed25519**.

El par de claves se generó desde la estación Windows:

```powershell
ssh-keygen -t ed25519 -a 100
```

La clave privada permanece únicamente en el cliente y el servidor almacena solamente la clave pública autorizada.

## Migración de ssh.socket a ssh.service

Durante la revisión inicial se detectó que SSH estaba funcionando mediante activación por `ssh.socket`.

Para centralizar la configuración en `sshd_config` y controlar directamente el puerto personalizado, se deshabilitó la activación por socket y se habilitó el servicio tradicional:

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
```

## Incidencia con cloud-init y SSH

Uno de los puntos más interesantes del proyecto apareció después de deshabilitar la autenticación SSH mediante contraseña.

En el fichero principal se había definido:

```text
PasswordAuthentication no
```

Sin embargo, la configuración efectiva seguía permitiendo contraseña.

En lugar de asumir que `sshd_config` se estaba aplicando correctamente, se revisó la configuración efectiva:

```bash
sudo sshd -T
```

y se buscaron directivas relacionadas en todos los archivos de configuración:

```bash
sudo grep -Rni \
'^[[:space:]]*PasswordAuthentication' \
/etc/ssh/sshd_config \
/etc/ssh/sshd_config.d/
```

La causa se encontró en:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

donde `cloud-init` estaba sobrescribiendo la configuración principal mediante:

```text
PasswordAuthentication yes
```

Después de corregir el override, se volvió a validar:

```bash
sudo sshd -t
sudo sshd -T
```

Esto permitió confirmar que la **configuración efectiva aplicada por SSH** coincidía realmente con la política de seguridad definida.

## Validación de SSH

La configuración final se comprobó mediante pruebas reales.

### Rechazo de autenticación por contraseña

Se forzó una conexión únicamente mediante contraseña:

```powershell
ssh -o PreferredAuthentications=password `
    -o PubkeyAuthentication=no `
    -p 48222 erick@<servidor>
```

Resultado:

```text
Permission denied (publickey)
```

### Rechazo del acceso root

También se probó el acceso remoto directo como `root`, que fue rechazado.

### Acceso correcto mediante clave pública

La conexión válida se realizó con:

```powershell
ssh -p 48222 erick@<servidor>
```

La sesión se abrió correctamente utilizando la clave Ed25519 y un usuario administrativo no root.

## Banners de acceso

Se configuró un aviso legal previo al inicio de sesión utilizando:

```text
/etc/issue.net
```

y la directiva:

```text
Banner /etc/issue.net
```

También se configuró un mensaje posterior al login mediante:

```text
/etc/motd
```

Ambos fueron validados durante la conexión SSH final.

## Sincronización horaria

La zona horaria del sistema se configuró como:

```text
Europe/Madrid
```

Para la sincronización NTP se utilizó **Chrony**.

Las principales comprobaciones fueron:

```bash
systemctl status chrony
chronyc tracking
chronyc sources -v
```

La validación final mostró:

```text
Stratum 3
Leap status: Normal
Reach: 377
```

con una fuente NTP activa seleccionada.

La sincronización horaria es especialmente importante para poder correlacionar correctamente eventos y logs de SSH, PAM, systemd y otros servicios.

## Actualizaciones automáticas de seguridad

Se configuró y validó `unattended-upgrades` para automatizar la instalación de actualizaciones de seguridad.

Entre las comprobaciones realizadas:

```bash
systemctl status unattended-upgrades
systemctl list-timers --all
```

También se realizó una simulación segura:

```bash
sudo unattended-upgrade --dry-run --debug
```

La prueba finalizó correctamente y sin paquetes rotos.

## Validación de servicios expuestos

La superficie final de exposición se revisó con:

```bash
sudo ss -tulpn
```

El único servicio accesible desde la red quedó siendo OpenSSH en el puerto TCP personalizado.

Los servicios locales de resolución DNS y Chrony permanecieron vinculados únicamente a loopback, evitando una exposición innecesaria a la LAN.

## Estrategia de validación

Uno de los objetivos principales de este laboratorio fue **validar cada medida aplicada en lugar de asumir que una configuración era correcta únicamente porque aparecía escrita en un archivo**.

Se realizaron comprobaciones sobre:

- Extensiones de virtualización anidada
- Funcionamiento de VMware dentro de Hyper-V
- Sistema operativo y kernel
- Estructura LVM
- Conectividad de red
- Sintaxis de sudoers
- Bloqueo real mediante PAM
- Configuración efectiva de SSH
- Rechazo de autenticación por contraseña
- Rechazo de acceso root
- Acceso correcto mediante Ed25519
- Sincronización NTP con Chrony
- Actualizaciones automáticas
- Servicios y puertos en escucha

## Principales aprendizajes técnicos

### Validar la configuración efectiva

Modificar un fichero no garantiza que esa sea la configuración finalmente utilizada por el servicio.

La incidencia con `cloud-init` demostró la importancia de consultar la configuración efectiva:

```bash
sshd -T
```

### Aplicar cambios de seguridad progresivamente

El hardening de SSH se realizó por etapas.

Primero se verificó el acceso mediante clave pública y solamente después se deshabilitó la autenticación por contraseña.

De la misma forma, el nuevo puerto SSH se validó antes de eliminar la vía anterior de acceso.

### Realizar copias antes de modificar autenticación

Antes de modificar PAM se realizaron copias de seguridad.

Los errores en este tipo de configuración pueden impedir incluso el acceso legítimo al sistema.

### Probar los controles de seguridad

Existe una diferencia importante entre configurar una medida y comprobar que realmente funciona.

```text
Configurar bloqueo de cuenta
        ≠
Validar bloqueo de cuenta

Configurar autenticación por clave
        ≠
Demostrar que las contraseñas son rechazadas
```

Las pruebas permiten confirmar que la política de seguridad se está aplicando realmente.

## Competencias trabajadas

Este proyecto demuestra experiencia práctica con:

- Administración de sistemas Linux
- Ubuntu Server
- Hyper-V
- VMware Workstation
- Virtualización anidada
- LVM
- Redes Linux
- OpenSSH
- Criptografía de clave pública
- Ed25519
- PAM
- pam_faillock
- sudo / sudoers
- systemd
- Chrony / NTP
- Actualizaciones automáticas
- Diagnóstico de incidencias
- Validación de configuraciones
- Documentación técnica

## Documentación y evidencias

Puedes consultar información adicional del laboratorio en los siguientes apartados:

- [Validación técnica](docs/validation.md)
- [Evidencias visuales](screenshots/README.md)

## Próximos pasos

Este mismo servidor continuará evolucionando en nuevas fases del laboratorio.

Las siguientes áreas previstas incluyen:

```text
Firewall
Fail2ban
DNS
Servidor web
HTTPS
Let's Encrypt
Renovación automática de certificados
Auditoría de exposición de red
```

Estas nuevas capas de seguridad se apoyarán sobre la base endurecida en este primer proyecto.

---

> Este repositorio documenta un entorno personal de aprendizaje y laboratorio.  
> La documentación pública está deliberadamente saneada y no contiene contraseñas, claves privadas SSH, tokens ni otras credenciales sensibles.
