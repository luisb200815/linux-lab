# Notas de Laboratorio: Seguridad en SSH, Mosh y Port Forwarding

Este documento resume los pasos, comandos y conceptos clave aprendidos y aplicados durante el laboratorio de administración de servidores Linux.

---

## 1. Endurecimiento de Seguridad en el Servidor SSH (*Hardening*)

Para proteger un servidor Linux expuesto a internet contra ataques de fuerza bruta y accesos no autorizados, es fundamental configurar correctamente el servicio del servidor SSH.

* **Archivo de configuración del servidor:** `/etc/ssh/sshd_config`  
  *(Nota: No confundir con `/etc/ssh/ssh_config`, que pertenece al cliente SSH).*

### Directivas de seguridad aplicadas:
1. **Deshabilitar autenticación por contraseña:**
   ```text
   PasswordAuthentication no
   ```
   * **Propósito:** Obliga a utilizar llaves criptográficas (como Ed25519) para iniciar sesión, bloqueando ataques automatizados que intentan adivinar contraseñas.

2. **Prohibir el inicio de sesión directo como `root`:**
   ```text
   PermitRootLogin no
   ```
   * **Propósito:** Evita que un atacante intente acceder directamente con el usuario administrador supremo. Obliga a los administradores a entrar con un usuario personal y escalar privilegios usando `sudo`, mejorando la trazabilidad y auditoría.

### Comandos de gestión y verificación:
* **Editar el archivo con Vim:**
  ```bash
  sudo vim /etc/ssh/sshd_config
  ```
* **Buscar texto en Vim (en modo normal):**
  ```vim
  /PermitRootLogin
  ```
  *(La búsqueda distingue entre mayúsculas y minúsculas. Si la directiva no existe en el archivo, se puede agregar manualmente al final).*
* **Reiniciar el servicio SSH para aplicar los cambios:**
  ```bash
  sudo service sshd restart
  ```
  *(Alternativa en sistemas con systemd: `sudo systemctl restart ssh`)*
* **Comprobar el bloqueo de `root`:**
  ```bash
  ssh root@localhost
  ```
  *Resultado esperado:* `Permission denied`.

---

## 2. Conexiones Resilientes con Mosh (*Mobile Shell*)

**Mosh** es una alternativa moderna a SSH diseñada para conexiones móviles o inestables. A diferencia de SSH (que usa TCP y corta la sesión ante cualquier microcorte de red), Mosh utiliza **UDP** y un protocolo de sincronización de estado (*SSP*).

* **Ventaja principal:** Si se desconecta la red o cambia la dirección IP del cliente, la sesión no se pierde; entra en pausa y se recupera automáticamente en cuanto vuelve la conectividad.

### Pasos realizados:
1. **Instalación en el servidor/VM:**
   ```bash
   sudo apt update && sudo apt install mosh
   ```
2. **Conexión mediante Mosh:**
   ```bash
   mosh tu_usuario@localhost
   ```
3. **Prueba de recuperación:** Al desconectar y volver a conectar el adaptador de red en VirtualBox mientras corría un comando continuo (`ping 8.8.8.8`), Mosh mantuvo la sesión intacta y reanudó la terminal automáticamente.

---

## 3. Reenvío de Puertos en Segundo Plano (*Background Port Forwarding*)

Para crear un túnel SSH sin dejar la terminal bloqueada con una sesión interactiva abierta, se combinan las siguientes banderas:

* **`-f` (*Fork into background*):** Envía el proceso SSH a segundo plano inmediatamente después de la autenticación, liberando la terminal.
* **`-N` (*No remote command*):** Indica a SSH que no ejecute ningún comando ni abra una shell en el servidor remoto (ideal cuando solo se requiere reenviar puertos).
* **`-L` (*Local port forwarding*):** Define el mapeo del puerto local hacia el destino remoto.

### Comando utilizado:
```bash
ssh -f -N -L 9999:localhost:8888 tu_usuario@localhost
```

### Verificación y cierre del túnel:
* **Comprobar que el proceso sigue activo en segundo plano:**
  ```bash
  ps aux | grep ssh
  ```
* **Verificar que el puerto local está escuchando:**
  ```bash
  ss -tlnp | grep 9999
  ```
* **Cerrar el túnel:** Identificar el número de proceso (`PID`) con `ps aux | grep ssh` y detenerlo con `kill <PID>`.