# ✨ Laboratorio con pfSense + Red Interna Aislada en Proxmox

## 🔧 Objetivo

Implementar una red virtual aislada tipo `172.20.0.0/16` dentro de un entorno de laboratorio en **Proxmox**, utilizando **pfSense como router/firewall**, para que las VMs puedan acceder a Internet y ser alcanzables desde la red física `192.168.0.0/24`.

---

## 📂 Infraestructura del entorno

### Red física:

- **Rango:** `192.168.0.0/24`
- **Router principal:** `192.168.0.1`
- **Proxmox host:** `192.168.0.10`
- **PC de trabajo:** `192.168.0.99`

### Red virtual del laboratorio:

- **Rango:** `172.20.0.0/16`
- **pfSense LAN:** `172.20.0.1`

### Elementos principales:

- **Bridge en Proxmox:**
  - `vmbr0` → Red física (WAN pfSense)
  - `vmbr1` → Red interna aislada (LAN pfSense)
- **VM pfSense:**
  - WAN: `192.168.0.100` (manual o DHCP con reserva)
  - LAN: `172.20.0.1`
- **VMs del lab:**
  - Conectadas sólo a `vmbr1`
  - IPs asignadas por DHCP o manuales en `172.20.0.0/16`

---

## 🔧 Configuración en Proxmox

### Agregar nuevo bridge `vmbr1`:

Editar `/etc/network/interfaces`:

```bash
auto vmbr1
iface vmbr1 inet manual
    bridge_ports none
    bridge_stp off
    bridge_fd 0
```

Reiniciar redes:

```bash
ifreload -a
```

---

## 🚪 Instalación y configuración de pfSense

1. Crear VM con 2 NICs:
   - NIC 1: `vmbr0` (WAN)
   - NIC 2: `vmbr1` (LAN)
2. Instalar pfSense desde ISO oficial
3. Asignar:
   - WAN: `em0` → `192.168.0.100`
   - LAN: `em1` → `172.20.0.1`
4. Habilitar DHCP en LAN: `172.20.10.100 - 172.20.10.200`
5. Confirmar NAT automático:
   - `Firewall > NAT > Outbound`
   - Modo: `Automatic Outbound NAT rule generation`
6. Verificar reglas en LAN:
   - `Firewall > Rules > LAN`
   - Permitir `any` desde `LAN net` hacia `any`

---

## 🐞 DNS Resolver (Unbound)

1. `Services > DNS Resolver`
2. Configurar:
   - Listen Interfaces: `LAN` + `Localhost`
   - Outgoing Interfaces: `WAN`
   - **Desactivar:** `Strict Binding`
3. Asegurar ACL:
   - `172.20.0.0/16` con acceso `Allow`
4. Validar en `Diagnostics > DNS Lookup`

---

## 👷 VM de prueba (Ubuntu 25.04)

- Conectada sólo a `vmbr1`
- Recibe IP `172.20.0.100` por DHCP
- Gateway: `172.20.0.1`

### DNS en Ubuntu:

Desactivar `systemd-resolved`:

```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
echo -e "nameserver 172.20.0.1\nnameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

Validar conectividad:

```bash
ping google.com
curl -I https://soporteregio.com
```

---

## 🛠️ Habilitar acceso desde red física (192.168.0.0/24)

### Paso 1: Desactivar bloqueos de pfSense

Ir a `Interfaces > WAN`:

-

### Paso 2: Agregar regla en WAN

`Firewall > Rules > WAN > +Add`

| Campo       | Valor                   |
| ----------- | ----------------------- |
| Action      | Pass                    |
| Protocol    | Any                     |
| Source      | 192.168.0.0/24          |
| Destination | 172.20.0.0/16           |
| Description | Permitir acceso LAN-Lab |

Guardar y Apply Changes.

### Paso 3: Agregar ruta estática en el router

**En router 192.168.0.1:**

| Red destino | Máscara     | Gateway       |
| ----------- | ----------- | ------------- |
| 172.20.0.0  | 255.255.0.0 | 192.168.0.100 |

Esto permite que **cualquier equipo en la red 192.168.0.0/24** acceda a las VMs del lab.

---

## 🚀 Estado final

| Elemento                   | Estado |
| -------------------------- | ------ |
| pfSense instalado          | ✅      |
| Red interna aislada        | ✅      |
| NAT y salida a Internet    | ✅      |
| DNS funcionando            | ✅      |
| VMs conectadas y navegando | ✅      |
| Acceso desde red física    | ✅      |

---

## 🔗 Siguientes pasos sugeridos

- 🌐 Integrar Tailscale en pfSense o una VM
- 🪜 Crear snapshot Golden de pfSense
- 🤧 Automatizar despliegue de VMs con Vagrant/Ansible
- 🚧 Aplicar microsegmentación por VLAN en futuro

---

**Documentado por:** Riky (Ricardo López)

**Entorno:** `pve.homelab01.local` - Proxmox con red aislada para labs

