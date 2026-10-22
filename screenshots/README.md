# Evidencias visuales

Esta carpeta contiene una selección de las evidencias más representativas del laboratorio de despliegue y hardening de Ubuntu Server.

El objetivo es mostrar de forma visual que las configuraciones documentadas fueron comprobadas realmente en el entorno de laboratorio.

## 1. Virtualización anidada

![Virtualización anidada](01_virtualizacion_anidada.png)

Verificación de que la máquina Windows 11 recibe correctamente las extensiones necesarias para ejecutar virtualización anidada.

---

## 2. VMware ejecutando Ubuntu Server

![VMware Ubuntu](02_vmware_ubuntu_operativo.png)

VMware Workstation ejecutando correctamente la máquina Ubuntu Server dentro del entorno virtualizado.

---

## 3. Bloqueo de cuenta con pam_faillock

![PAM faillock](03_pam_faillock_bloqueo.png)

Validación real de la política de bloqueo configurada mediante `pam_faillock`.

Tras alcanzar el número máximo de intentos fallidos, la cuenta queda temporalmente bloqueada.

---

## 4. Configuración efectiva de SSH

![Configuración SSH](04_ssh_configuracion_efectiva.png)

Comprobación de la configuración realmente aplicada por OpenSSH.

Entre las medidas validadas:

```text id="c3wi40"
Puerto personalizado
Autenticación mediante clave pública
Contraseña deshabilitada
Acceso root deshabilitado
```

La comprobación de la configuración efectiva permitió detectar también un override generado por `cloud-init`.

---

## 5. Rechazo de contraseña y acceso root

![Rechazo SSH](05_ssh_rechazo_password_root.png)

Pruebas realizadas desde el cliente Windows para confirmar que:

- la autenticación mediante contraseña está deshabilitada;
- el acceso remoto directo mediante `root` está bloqueado.

---

## 6. Acceso correcto mediante Ed25519

![Acceso Ed25519](06_ssh_ed25519_acceso_correcto.png)

Conexión correcta mediante clave Ed25519 utilizando el puerto SSH personalizado.

También se verifican los banners de seguridad configurados antes y después del inicio de sesión.

---

## 7. Sincronización horaria con Chrony

![Chrony](07_chrony_sincronizacion.png)

Validación del servicio Chrony y de la sincronización NTP del servidor.

La comprobación incluye:

```text id="1hz2bm"
Stratum
Estado de sincronización
Fuente NTP seleccionada
Reachability
```

---

## 8. Servicios y puertos expuestos

![Servicios expuestos](08_servicios_expuestos_ss_tulpn.png)

Inspección final de sockets mediante:

```bash id="ex3ns6"
sudo ss -tulpn
```

La validación confirma que OpenSSH es el único servicio expuesto a la red, mientras que otros servicios locales permanecen vinculados únicamente a loopback.

---

## Resumen

Las evidencias permiten comprobar visualmente:

```text id="ks7fh8"
Virtualización anidada          OK
VMware Workstation              OK
pam_faillock                    OK
Hardening SSH                   OK
Contraseñas SSH deshabilitadas  OK
Root SSH deshabilitado          OK
Ed25519                         OK
Chrony                          OK
Servicios expuestos revisados   OK
```

Las imágenes utilizadas en este repositorio forman parte de un entorno de laboratorio personal y han sido seleccionadas evitando publicar credenciales, claves privadas u otra información sensible.
