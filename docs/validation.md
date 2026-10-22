# Validación técnica

Este documento resume las principales comprobaciones realizadas durante el despliegue y hardening del servidor Ubuntu.

El objetivo no fue únicamente aplicar configuraciones, sino verificar que cada medida funcionaba realmente en el sistema.

## 1. Virtualización anidada

Se comprobó que la máquina Windows 11 recibía correctamente las extensiones necesarias para ejecutar virtualización anidada.

Validación realizada:

```powershell
Get-CimInstance Win32_Processor |
Select-Object Name,
              VirtualizationFirmwareEnabled,
              VMMonitorModeExtensions,
              SecondLevelAddressTranslationExtensions
```

También se verificó desde el host Hyper-V:

```powershell
Get-VMProcessor -VMName "Win_11" |
Select-Object Count, ExposeVirtualizationExtensions
```

Resultado esperado:

```text
ExposeVirtualizationExtensions = True
```

Una VM temporal ejecutada mediante Hyper-V dentro de Windows 11 permitió confirmar que las extensiones de virtualización llegaban correctamente al sistema invitado.

Posteriormente VMware Workstation pudo ejecutar la VM Ubuntu Server.

---

## 2. Validación inicial de Ubuntu Server

Después de completar la instalación se verificaron los datos principales del sistema:

```bash
whoami
hostname
uname -a
cat /etc/os-release
ip -br a
df -h
free -h
sudo -l
```

También se revisó la estructura de almacenamiento:

```bash
lsblk
sudo pvs
sudo vgs
sudo lvs
```

El servidor quedó configurado con LVM y espacio libre dentro del grupo de volúmenes para futuras ampliaciones.

---

## 3. Validación de conectividad

La conectividad se comprobó progresivamente por capas.

### Gateway

```bash
ping 10.10.30.1
```

### Salida a Internet sin depender de DNS

```bash
ping 1.1.1.1
```

### Resolución DNS

```bash
ping ubuntu.com
```

Esta secuencia permitió validar de forma independiente:

```text
Red local
   ↓
Gateway
   ↓
Salida a Internet
   ↓
Resolución DNS
```

---

## 4. Validación de sudoers

La política administrativa se configuró mediante un fichero específico en:

```text
/etc/sudoers.d/asir-admins
```

Antes de utilizarla se comprobó su sintaxis:

```bash
sudo visudo -cf /etc/sudoers.d/asir-admins
```

Resultado:

```text
parsed OK
```

También se verificaron los privilegios efectivos:

```bash
sudo -l
```

---

## 5. Validación de pam_faillock

Se configuró una política de bloqueo con:

```text
deny = 5
fail_interval = 900
unlock_time = 900
```

Se realizaron cinco intentos de autenticación incorrectos sobre una cuenta de prueba.

El estado se revisó mediante:

```bash
sudo faillock --user asiradmin
```

Después de alcanzar el límite, el sistema bloqueó temporalmente la cuenta durante 15 minutos.

Esto permitió comprobar que la política PAM estaba realmente operativa.

---

## 6. Validación de la configuración efectiva de SSH

La configuración de OpenSSH no se validó únicamente revisando el contenido de `sshd_config`.

Se utilizó:

```bash
sudo sshd -T
```

para comprobar la configuración efectiva aplicada por el servicio.

La configuración final debía mostrar:

```text
port 48222
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

Durante esta comprobación se detectó que `cloud-init` estaba sobrescribiendo la directiva:

```text
PasswordAuthentication no
```

mediante:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

Después de corregir el override se volvió a validar:

```bash
sudo sshd -t
sudo sshd -T
```

---

## 7. Rechazo de autenticación mediante contraseña

Desde el cliente Windows se forzó una conexión utilizando únicamente autenticación por contraseña:

```powershell
ssh -o PreferredAuthentications=password `
    -o PubkeyAuthentication=no `
    -p 48222 erick@<servidor>
```

Resultado:

```text
Permission denied (publickey)
```

Esto confirmó que la autenticación mediante contraseña estaba deshabilitada.

---

## 8. Rechazo de acceso SSH como root

También se intentó realizar una conexión SSH directa como `root`.

```powershell
ssh -p 48222 root@<servidor>
```

El acceso fue rechazado, confirmando la aplicación efectiva de:

```text
PermitRootLogin no
```

---

## 9. Acceso mediante clave Ed25519

La conexión válida se realizó mediante:

```powershell
ssh -p 48222 erick@<servidor>
```

El servidor solicitó la passphrase de la clave privada Ed25519 y permitió abrir la sesión con un usuario administrativo no root.

Durante el proceso se mostraron correctamente:

```text
/etc/issue.net
/etc/motd
```

confirmando también el funcionamiento de los banners pre-login y post-login.

---

## 10. Validación de Chrony

La sincronización horaria se comprobó mediante:

```bash
systemctl status chrony
chronyc tracking
chronyc sources -v
```

Entre los valores observados:

```text
Stratum 3
Leap status: Normal
Reach: 377
```

La presencia de una fuente marcada con:

```text
^*
```

confirmó que Chrony disponía de una fuente NTP seleccionada y activa.

---

## 11. Validación de actualizaciones automáticas

El estado de `unattended-upgrades` se verificó mediante:

```bash
systemctl status unattended-upgrades
```

También se revisaron los timers de APT:

```bash
systemctl list-timers --all | grep apt
```

Finalmente se realizó una simulación:

```bash
sudo unattended-upgrade --dry-run --debug
```

La prueba terminó correctamente, sin errores ni paquetes rotos.

---

## 12. Validación final de servicios expuestos

Se realizó una inspección final de sockets:

```bash
sudo ss -tulpn
```

El único servicio expuesto a la red quedó siendo OpenSSH en:

```text
TCP 48222
```

Los servicios locales de DNS y Chrony permanecieron vinculados únicamente a interfaces loopback.

---

## Resultado final

Las pruebas realizadas permitieron confirmar:

```text
Virtualización anidada           OK
Ubuntu Server operativo          OK
LVM                              OK
Conectividad                     OK
sudoers                          OK
pam_faillock                     OK
SSH mediante Ed25519             OK
Contraseña SSH deshabilitada     OK
Acceso root SSH deshabilitado    OK
Puerto SSH personalizado         OK
Banners                          OK
Chrony                           OK
unattended-upgrades              OK
Servicios expuestos revisados    OK
```

La validación confirma que las medidas de hardening configuradas se estaban aplicando realmente en el sistema.
