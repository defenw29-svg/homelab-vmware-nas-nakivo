# Lab 03 - NAS + NAKIVO - Virtualizacion Anidada con VMware + vSphere + TrueNAS

<img width="1920" height="1280" alt="image" src="https://github.com/user-attachments/assets/0ce0f38b-8ac3-473c-969d-0362fb6e485d" />

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

### Asignacion de Direccionamiento IP:

| Servidor | Interfaz de Red Virtual (VMkernel) | Direccion IP Asignada |
| :--- | :--- | :--- |
| **ESXi01** | `vmk0` (Red WAN/NAT en vSwitch0) | `192.168.101.101` |
| **ESXi01** | `vmk1` (Red Privada en vSwitch1) | `192.168.105.101` |
| **ESXi02** | `vmk0` (Red WAN/NAT en vSwitch0) | `192.168.101.102` |
| **ESXi02** | `vmk1` (Red Privada en vSwitch1) | `192.168.105.102` |

---

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

---

## 👤 Autoria y Proteccion
**Ivan Ajenjo Morales - defenw29-svg**
Este laboratorio es parte de mi portfolio De Helpdesk L1/L2 a Junior SecOps / SysAdmin.
Licencia MIT: Si haces fork, mantén mi nombre y el link al repo original.
