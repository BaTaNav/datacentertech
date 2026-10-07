# Documentatie – Datacenter Technologies

## Inhoud

- [Server- en loginoverzicht](#server--en-loginoverzicht)
- [Les 1 – ESXi 8.0.2 installatie](#les-1--esxi-802-installatie)
- [Les 2 – Router instellen](#les-2--router-instellen)
- [Les 3 – vCenter installatie](#les-3--vcenter-installatie)

---

## Server- en loginoverzicht

### Servers

| Server | Username | Password |
|--------|----------|----------|
| Server 1 | `root` | `VHMMEPVX9WAR` |
| Server 2 | `root` | `4W44FARVX3RV` |

### iDRAC webview

| Server | Username | Password |
|--------|----------|----------|
| Server 1 | `root` | `dfghdfgh1/` |
| Server 2 | `root` | `STUDENT1/` |

### vCenter (Server 2)

| Veld | Waarde |
|------|--------|
| Name | `ESXVC01` |
| Password | `Student1.` |
| IP-adres | `192.168.1.100` |
| Login | `293.268.1.100:443` |

### vSphere

| Veld | Waarde |
|------|--------|
| Login | `administrator@vsphere.local` |
| Password | `Student1.` |

### Ubuntu

| Username | Password |
|----------|----------|
| `arnaud` | `Student` |

---

## Les 1 – ESXi 8.0.2 installatie

### Installatie

1. Installeer ESXi 8.0.2 via de **virtuele console**.
2. Na de installatie: **Remove media / download device** → server **rebooten**.

### VMware host beheren

- **Username:** `root`
- **Password:** (fout gelopen → ESXi opnieuw gedownload en geïnstalleerd op de server)

### Probleem 1: inloggen via virtuele console

- **Probleem:** via de terminal kunnen we inloggen met het password (en bv. het password aanpassen), maar via de virtuele console view kunnen we niet inloggen.
- **Oplossing:** het password is `/` geworden.

### Hostname wijzigen

In de ESXi Host Client:

1. Ga naar **Networking**
2. Kies **TCP/IP configuration**
3. Open de **Default TCP/IP stack**
4. Klik op **Edit settings**

---

## Les 2 – Router instellen

### Doel

1. Controleren of alles van opdracht 1 gedaan is.
2. De servers verbinden met een router die verbonden is met het EhB-netwerk.

### Router configuratie

```
enable
configure terminal
hostname R1

! --- WAN: poort naar het EhB netwerk, IP via DHCP ---
interface FastEthernet0/0
 description WAN - EhB network
 ip address dhcp
 ip nat outside
 no shutdown

! --- LAN: management netwerk 192.168.1.0/24 ---
interface FastEthernet0/1
 description LAN - Management network
 ip address 192.168.1.1 255.255.255.0
 ip nat inside
 no shutdown

! --- Dynamic NAT (PAT) ---
access-list 1 permit 192.168.1.0 0.0.0.255
ip nat inside source list 1 interface FastEthernet0/0 overload
end
write memory
```

Na `write memory` is de router ingesteld.

### Volgende stappen

1. De servers verbinden met het EhB-netwerk (via de router).
2. Controleren of de servers een IP-adres hebben gekregen.

---

## Les 3 – vCenter installatie

- Tijd om de **vCenter** te installeren.
- Hiervoor is **internet** nodig.
- Check of er nog **config op de router** staat als je geen internetverbinding hebt.

### Installatiebestanden

Netwerkpad:

```
\\dt-srv-file1.ehb.local\Studentenstorage\TI\Datacenter Technologies\VMware\
```
