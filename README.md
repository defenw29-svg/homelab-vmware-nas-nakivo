# Lab 03 - NAS + NAKIVO - Virtualizacion Anidada con VMware + vSphere + TrueNAS

<img width="1920" height="1280" alt="658502182-6ef3bdc5-6667-44be-841f-17bf1d1b126c-el-nombre-arriba-diseo-ciberpunk-moderno-solo-cambia" src="https://github.com/user-attachments/assets/56fee931-240e-4be5-9a07-1c420a45d4b9" />

> **Autor:** Ivan Ajenjo Morales (defenw29-svg) | Helpdesk L1/L2 | ITIL | Junior SecOps
> **Licencia:** MIT - Ver LICENSE - Mantener autoria obligatoria
> **Topologia:** Nested Virtualization - VMware Workstation -> ESXi01 + ESXi02 + TrueNAS + vCenter + NAKIVO

## Objetivo
Documentar los pasos logicos para configurar la topologia de virtualizacion anidada sobre la maquina fisica host (Physical Computer) para un laboratorio de alta disponibilidad.

---

## 🗺 Fase 1: Configuracion de Redes Virtuales (VMware Workstation)
Antes de crear cualquier maquina virtual, es obligatorio configurar los interruptores de red (*Virtual Network Editor* de VMware) para replicar la segmentacion del diagrama.

*   **Red WAN / NAT (VMnet8):**
    *   **Funcion:** Salida a Internet y acceso a produccion externa.
    *   **Rango IP:** `192.168.101.0/24`
    *   **Gateway / NAT Device:** `192.168.101.2`
    *   **DHCP Server:** Activado en el rango (IP del servidor interna: `192.168.101.254`).
*   **Red de Almacenamiento y Gestion (VMnet1):**
    *   **Funcion:** Trafico privado de iSCSI/NFS y gestion de cluster.
    *   **Tipo:** Host-Only (Aislada de Internet).
    *   **Rango IP:** `192.168.105.0/24`
    *   **DHCP Server:** Activado para asignaciones rapidas (IP interna: `192.168.105.254`).

---

## 💿 Fase 2: Configuracion del Almacenamiento Compartido (NAS)
El cluster vSphere requiere que los datastores sean accesibles por ambos hosts ESXi de forma simultanea.

1.  **Crear VM en VMware Workstation:** Instalar **TrueNAS / FreeNAS**.
2.  **Asignacion de Red:** 
    *   Conectar la tarjeta de red virtual a **VMnet1** (Red Host-Only).
3.  **Configuracion de IP Estatica:** Asignar la direccion fija `192.168.105.105`.
4.  **Almacenamiento:** Crear pools de discos y habilitar el servicio **iSCSI** o **NFS** compartidos para exportar espacio hacia los hipervisores.

---

## ⚡ Fase 3: Despliegue de los Hipervisores (ESXi01 y ESXi02)
Se replicaran dos instancias identicas para habilitar la alta disponibilidad.

### Configuracion del Hardware Virtual en cada VM:
*   **Procesador:** Habilitar obligatoriamente la casilla `Virtualize Intel VT-x/EPT or AMD-V/RVI` (para permitir la virtualizacion anidada).
*   **Adaptadores de Red (Dos por cada ESXi):**
    *   **NIC 1:** Conectado a **VMnet8** (Produccion/Acceso).
    *   **NIC 2:** Conectado a **VMnet1** (Gestion/Almacenamiento).

### Asignación de Direccionamiento IP:

| Servidor | Interfaz | Red Virtual | Dirección IP Asignada |
|:--------:|:--------:|---|:---------------------:|
| **ESXi01** | `vmk0` | `vSwitch0` → **VMnet8** (Producción / Acceso) | `192.168.101.101` |
| **ESXi01** | `vmk1` | `vSwitch1` → **VMnet1** (Gestión / Almacenamiento) | `192.168.105.101` |
| **ESXi02** | `vmk0` | `vSwitch0` → **VMnet8** (Producción / Acceso) | `192.168.101.102` |
| **ESXi02** | `vmk1` | `vSwitch1` → **VMnet1** (Gestión / Almacenamiento) | `192.168.105.102` |
| **TrueNAS** | `eth0` | **VMnet1** (iSCSI / NFS) | `192.168.105.105` |
| **vCenter** | `eth0` | **VMnet1** / **VMnet8** | `192.168.105.103` |

## 👑 Fase 4: Control Centralizado (vCenter Server - VCSA)
La pieza maestra que unifica la infraestructura corporativa.

1.  **Despliegue:** Montar la ISO del instalador del VCSA desde tu ordenador fisico.
2.  **Destino:** Indicar como destino de la instalacion el hipervisor **ESXi01**.
3.  **Configuracion de Red:**
    *   Conectar a la red de gestion en el rango interno.
    *   Asignar la IP estatica: `192.168.105.103` (o en su defecto en la red de produccion `192.168.101.103` segun preferencias de enrutamiento).
4.  **Cluster:** Crear el Datacenter en la interfaz web de vCenter y anadir ambos servidores (`101` y `102`).

---

## 🛡 Fase 5: Estrategia de Backup con NAKIVO
Una vez que el entorno vSphere esta operativo (con la maquina virtual `Lubuntu16` corriendo en produccion), se despliega el sistema de respaldos.

*   **Despliegue del Appliance:** Importar el archivo OVA de **NAKIVO Backup & Replication** directamente en el vCenter.
*   **Inventario:** Vincular NAKIVO con las credenciales de nuestro vCenter para que descubra automaticamente a los hosts **ESXi01** y **ESXi02**.
*   **Configuracion del Repositorio:** Conectar un almacenamiento secundario (puede ser otra ruta de red dedicada o un disco virtual independiente) configurando **repositorios inmutables** contra ataques de ransomware.
*   **Tareas de Backup:** Programar politicas de copias de seguridad incrementales con verificacion instantanea de recuperacion de datos (Flash VM Boot).

### Validacion
- Ping entre `192.168.101.101` y `192.168.105.105` (ESXi01 -> TrueNAS)
- vMotion entre ESXi01 y ESXi02 usando datastore compartido TrueNAS
- Backup NAKIVO de Lubuntu16 y restore de prueba con Flash VM Boot
## 🛠️ Anexo: Automatización de Red en ESXi mediante CLI (ESXCLI)

Para evitar configurar las interfaces de red de forma manual en el entorno visual (DCUI) de cada host, puedes habilitar el servicio **SSH** en tus servidores ESXi, conectarte a ellos y ejecutar los siguientes bloques de comandos para desplegar los switches virtuales, asociar las tarjetas físicas y levantar el direccionamiento estático de forma inmediata.

### 💻 Bloque de comandos para copiar y pegar en ESXi01:
## ⚙️ Automatización Avanzada y Optimización de Red mediante CLI (ESXCLI)

Para maximizar el rendimiento del tráfico de almacenamiento **iSCSI/NFS** y evitar configuraciones manuales propensas a errores, se implementa el uso de **Jumbo Frames (MTU 9000)** de extremo a extremo. Los siguientes bloques unifican el despliegue de la topología de red junto con la optimización de rendimiento corporativo.

---

### 💻 Bloque de comandos para copiar y pegar en ESXi01:

```bash
# ==============================================================================
# LAB_VSCHERE: CONFIGURACIÓN DE RED Y OPTIMIZACIÓN JUMBO FRAMES (ESXi01)
# ==============================================================================

# 1. Crear el switch virtual dedicado para Almacenamiento y Gestión Privada
esxcli network vswitch standard add --vswitch-name=vSwitch1

# 2. PRO-OPTIMIZACIÓN: Establecer MTU a 9000 a nivel de Switch Virtual (vSwitch1)
esxcli network vswitch standard set --mtu=9000 --vswitch-name=vSwitch1

# 3. Asociar la segunda tarjeta de red física (NIC 2 anidada) al nuevo vSwitch
esxcli network vswitch standard uplink add --uplink-name=vmnic1 --vswitch-name=vSwitch1

# 4. Crear el grupo de puertos (Port Group) para el tráfico de backend
esxcli network vswitch standard portgroup add --portgroup-name="Red_Privada" --vswitch-name=vSwitch1

# 5. Configurar la IP fija en la red pública de producción (VMnet8 / vSwitch0 predeterminado)
esxcli network ip interface ipv4 set --interface-name=vmk0 --ipv4=192.168.101.101 --netmask=255.255.255.0 --type=static

# 6. Crear la nueva interfaz VMkernel (vmk1) asignada al grupo de puertos privado
esxcli network ip interface add --interface-name=vmk1 --portgroup-name="Red_Privada"

# 7. Asignar la IP fija en el segmento privado para almacenamiento (VMnet1)
esxcli network ip interface ipv4 set --interface-name=vmk1 --ipv4=192.168.105.101 --netmask=255.255.255.0 --type=static

# 8. PRO-OPTIMIZACIÓN: Elevar MTU a 9000 en la interfaz VMkernel de almacenamiento (vmk1)
esxcli network ip interface set --mtu=9000 --interface-name=vmk1

# 9. Configurar de forma normativa la Puerta de Enlace Predeterminada del sistema (Gateway global)
esxcli network ip route ipv4 gateway set --gateway=192.168.101.2

# ==============================================================================
# [VERIFICACIÓN PRO] Listar interfaces para confirmar direccionamiento y MTU 9000
# ==============================================================================
esxcli network ip interface list
```

---

### 💻 Bloque de comandos para copiar y pegar en ESXi02:

```bash
# ==============================================================================
# LAB_VSCHERE: CONFIGURACIÓN DE RED Y OPTIMIZACIÓN JUMBO FRAMES (ESXi02)
# ==============================================================================

# 1. Crear el switch virtual dedicado para Almacenamiento y Gestión Privada
esxcli network vswitch standard add --vswitch-name=vSwitch1

# 2. PRO-OPTIMIZACIÓN: Establecer MTU a 9000 a nivel de Switch Virtual (vSwitch1)
esxcli network vswitch standard set --mtu=9000 --vswitch-name=vSwitch1

# 3. Asociar la segunda tarjeta de red física (NIC 2 anidada) al nuevo vSwitch
esxcli network vswitch standard uplink add --uplink-name=vmnic1 --vswitch-name=vSwitch1

# 4. Crear el grupo de puertos (Port Group) para el tráfico de backend
esxcli network vswitch standard portgroup add --portgroup-name="Red_Privada" --vswitch-name=vSwitch1

# 5. Configurar la IP fija en la red pública de producción (VMnet8 / vSwitch0 predeterminado)
esxcli network ip interface ipv4 set --interface-name=vmk0 --ipv4=192.168.101.102 --netmask=255.255.255.0 --type=static

# 6. Crear la nueva interfaz VMkernel (vmk1) asignada al grupo de puertos privado
esxcli network ip interface add --interface-name=vmk1 --portgroup-name="Red_Privada"

# 7. Asignar la IP fija en el segmento privado para almacenamiento (VMnet1)
esxcli network ip interface ipv4 set --interface-name=vmk1 --ipv4=192.168.105.102 --netmask=255.255.255.0 --type=static

# 8. PRO-OPTIMIZACIÓN: Elevar MTU a 9000 en la interfaz VMkernel de almacenamiento (vmk1)
esxcli network ip interface set --mtu=9000 --interface-name=vmk1

# 9. Configurar de forma normativa la Puerta de Enlace Predeterminada del sistema (Gateway global)
esxcli network ip route ipv4 gateway set --gateway=192.168.101.2

# ==============================================================================
# [VERIFICACIÓN PRO] Listar interfaces para confirmar direccionamiento y MTU 9000
# ==============================================================================
esxcli network ip interface list
```
fix(network): corregir sintaxis esxcli y optimizar almacenamiento con mtu 9000

- Corrige la sintaxis obsoleta de 'ip route ipv4 stat add' reemplazándola por 'gateway set' en los bloques de comandos del anexo CLI.
- Soluciona el error en la declaración de argumentos para la creación de la interfaz VMkernel vmk1 en el paso 5.
- Remueve el espacio en blanco invisible del port group "Red_Privada" que provocaba fallos de asociación de red.
- Agrega los comandos avanzados de automatización para implementar Jumbo Frames (MTU 9000) de extremo a extremo en vSwitch1 y vmk1.
- Introduce la sección de verificación profesional utilizando el comando 'vmkping' sin fragmentación (carga útil de 8972 bytes) para entornos de producción.

### ⚠️ Regla de Oro en Producción (Validación de Jumbo Frames)

Para garantizar que la infraestructura no descarte paquetes ni cause degradación severa (fragmentación), ejecuta el siguiente comando desde la consola SSH de cualquiera de tus hosts ESXi para **comprobar la conectividad limpia con TrueNAS sin fragmentar paquetes**:

```bash
# Validar conectividad de 9000 bytes hacia el almacenamiento de TrueNAS desde vmk1
vmkping -I vmk1 -s 8972 -d 192.168.105.105
```
*(Nota: El tamaño de payload 8972 es el máximo permitido para probar MTU 9000 restando las cabeceras ICMP/IP sin fragmentación).*

## 👤 Autoria y Proteccion
**Ivan Ajenjo Morales - defenw29-svg**
Este laboratorio es parte de mi portfolio De Helpdesk L1/L2 a Junior SecOps / SysAdmin.
Licencia MIT: Si haces fork, mantén mi nombre y el link al repo original.
