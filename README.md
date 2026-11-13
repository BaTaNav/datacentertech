# Documentatie – Datacenter Technologies
## Inhoud

- [Server- en loginoverzicht](#server--en-loginoverzicht)
- [Les 1 – ESXi 8.0.2 installatie](#les-1--esxi-802-installatie)
- [Les 2 – Router instellen](#les-2--router-instellen)
- [Les 3 – vCenter installatie](#les-3--vcenter-installatie)
- [Les 4 – ESX networking (vSwitch & VLANs)](#les-4--esx-networking-vswitch--vlans)
- [Les 5 – Shared storage (iSCSI)](#les-5--shared-storage-iscsi)
- [Les 6 – Cluster](#les-6--cluster)

---

## Server- en loginoverzicht

### Servers / iDRAC webview

| Server | Username | Password |
|--------|----------|----------|
| Server 1 | `root` | `VHMMEPVX9WAR` | 
| Server 2 | `root` | `4W44FARVX3RV` |

### ESXI Host client

| Server | Username | Password | IP address
|--------|----------|----------|-----------|
| Server 1 | `root` | `dfghdfgh1/` | 192.168.1.10|
| Server 2 | `root` | `STUDENT1/` | 192.168.1.11|

### vCenter (Server 2)

| Veld | Waarde |
|------|--------|
| Name | `ESXVC01` |
| Password | `Student1.` |
| IP-adres | `192.168.1.100` |
| Login | `192.168.1.100:443` |

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

---

## Les 4 – ESX networking (vSwitch & VLANs)

### Doel

Per ESXi-host dezelfde vSwitch aanmaken met VLANs voor **Production**, **DMZ**, **vMotion** en **Storage**.
- Link fysieke switch → hosts: **trunk**
- Link fysieke switch → storage device: **access** (volgende opdracht)

### VLAN-plan

| VLAN | Naam | Subnet | Type | Server 1 | Server 2 |
|------|------|--------|------|----------|----------|
| 10 | Production | 192.168.10.0/24 | VM Port Group | – | – |
| 20 | DMZ | 192.168.20.0/24 | VM Port Group | – | – |
| 30 | vMotion | 192.168.30.0/24 | VMkernel (vMotion) | vmk1 `192.168.30.2` | vmk1 `192.168.30.3` |
| 40 | Storage | 192.168.40.0/24 | VMkernel | vmk2 `192.168.40.2` | vmk2 `192.168.40.3` |

Management blijft op **vSwitch0** (vmk0, `192.168.1.0/24`, untagged).

### Bekabeling

Switch: **Cisco WS-C3750G-24TS** (poort gevonden via vCenter → Physical adapters → vmnic → tab **CDP**)

| Server | NIC | Switchpoort | Functie |
|--------|-----|-------------|---------|
| Server 1 | vmnic0 | Gi1/0/4 | Management (access) |
| Server 2 | vmnic0 | Gi1/0/3 | Management (access) |
| Server 1 | vmnic1 | Gi1/0/8 | vSwitch1 (trunk) |
| Server 2 | vmnic1 | Gi1/0/7 | vSwitch1 (trunk) |

> Eerst hadden we de VLANs op vSwitch0 (vmnic0) gezet. Omdat een vmnic maar aan één vSwitch kan hangen en het voorbeeld van de leerkracht een aparte vSwitch1 gebruikt, hebben we een tweede NIC aangesloten en de portgroups verhuisd naar vSwitch1. Zo blijft management gescheiden en loopt het geen risico.

### Configuratie vCenter (per host)

**Oude portgroups verwijderen van vSwitch0**

1. Host → **Configure → Networking → Virtual switches**
2. Bij vMotion/Storage: eerst de VMkernel-poort (`...` → **Remove**), dan de portgroup
3. DMZ en Production: `...` → **Remove**
4. **Management Network** en **VM Network** blijven staan

**vSwitch1 aanmaken + DMZ**

1. **ADD NETWORKING** → *Virtual Machine Port Group for a Standard Switch*
2. **New standard switch** (MTU 1500)
3. **+** → vmnic1 (staat onder *Unclaimed adapters*) → moet onder **Active adapters** staan
4. Label `DMZ`, VLAN `20` → Finish

**Overige portgroups** (ADD NETWORKING → *existing standard switch: vSwitch1*)

| Type | Label | VLAN | Services | IP server 2 |
|------|-------|------|----------|-------------|
| VM Port Group | Production | 10 | – | – |
| VMkernel | vMotion | 30 | vMotion ✔ | 192.168.30.3 /24 |
| VMkernel | Storage | 40 | geen | 192.168.40.3 /24 |

Op server 1 dezelfde labels en VLANs, met IPs `.2`.

### Configuratie fysieke switch

```
enable
configure terminal
vlan 10
 name Production
vlan 20
 name DMZ
vlan 30
 name vMotion
vlan 40
 name Storage
exit
interface range gi1/0/7 , gi1/0/8
 description Trunk naar ESXi vSwitch1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 spanning-tree portfast trunk
 no shutdown
end
write memory
```

Controle:

```
show vlan brief
show interfaces trunk
show interfaces status
```

Access-poort voor het storage device (volgende opdracht):

```
interface gi1/0/X
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
```

### Testen

1. SSH aanzetten: host → **Configure → System → Services → SSH → START**
2. Verbinden vanaf pc:

```
ssh root@192.168.1.10
```

3. VMkernel-interfaces controleren:

```
esxcli network ip interface ipv4 get
```

4. Pingen naar de andere host:

```
vmkping -I vmk1 192.168.30.2
vmkping -I vmk2 192.168.40.2
```

Resultaat: beide pings **geslaagd** ✅

5. SSH daarna weer op **STOP** zetten.

### Problemen

**Probleem 1: "There are no active physical network adapters for the switch"**
- **Oorzaak:** vmnic1 stond niet onder *Active adapters*.
- **Oplossing:** vmnic1 met de pijlknop naar **Active adapters** verplaatst.

**Probleem 2: `vmkping` naar Storage faalt (`Host is down`, 100% packet loss)**
- **Oorzaak:** bij de Storage-portgroup was VLAN ID **40** niet ingevuld.
- **Oplossing:** VLAN ID op `40` gezet → ping werkt.

### Volgende stap

Als de storage (volgende opdracht) werkt, kunnen we **NIC teaming** toepassen: meerdere fysieke links tussen de switch en de ESXi-hosts.


---

## Les 5 – Shared storage (iSCSI)
 
### Doel
 
Via het IP storage netwerk (VLAN 40) en **iSCSI** shared storage van de Synology NAS koppelen aan beide ESXi-hosts. Alle config gebeurt via **vCenter** (niet via de ESXi Host Client).
 
### IP-plan storage (VLAN 40 – 192.168.40.0/24)
 
| Toestel           | Interface | IP-adres         |
|-------------------|-----------|------------------|
| Server 1          | vmk2      | `192.168.40.2`   |
| Server 2          | vmk2      | `192.168.40.3`   |
| Synology NAS (SAN)| LAN       | `192.168.40.100` |
 
Geen gateway nodig: storage-verkeer blijft op laag 2.
 
### Bekabeling
 
| Toestel      | Switchpoort | Modus            |
|--------------|-------------|------------------|
| Synology NAS | Gi1/0/5     | access VLAN 40   |
| Server 1     | Gi1/0/8     | trunk (vSwitch1) |
| Server 2     | Gi1/0/7     | trunk (vSwitch1) |
 
### Configuratie fysieke switch
 
De trunks naar de hosts (Gi1/0/7, Gi1/0/8) laten VLAN 40 al toe (les 4). Enkel de poort van de NAS moest nog ingesteld worden:
 
```
enable
configure terminal
interface gi1/0/5
 description NAS-Synology
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
end
write memory
```
 
Controle:
 
```
show interfaces gi1/0/5 status
show vlan brief
show interfaces trunk
```
 
### Configuratie vCenter (per host)
 
**1. Software iSCSI adapter toevoegen**
 
1. Host → **Configure → Storage Adapters**
2. **Add Software Adapter** → **Add software iSCSI adapter** → OK
3. Er verschijnt een nieuwe adapter `vmhbaXX` (iSCSI Software Adapter)

**2. Network Port Binding**
 
1. Selecteer de iSCSI-adapter → tab **Network Port Binding** → **Add**
2. Kies **vmk2** (portgroup `Storage`, VLAN 40) → OK
3. Teaming-override is niet nodig: vSwitch1 heeft maar één uplink (vmnic1)

**3. Link naar de SAN (Dynamic Discovery)**
 
1. Tab **Dynamic Discovery** → **Add**
2. iSCSI server: `192.168.40.100`, poort `3260`
3. OK → **Rescan Adapter**

**4. Datastore aanmaken (maar één keer, op één host)**
 
1. Host → **Actions → Storage → New Datastore**
2. **VMFS** → naam `iSCSI-DS01` → Synology LUN selecteren → **VMFS 6** → volledige grootte → Finish
3. Op de andere host: **Rescan Storage** → de datastore verschijnt automatisch
> De datastore **niet** op beide hosts aanmaken/formatteren, anders wordt de LUN gewist.
 
### Testen
 
1. SSH aanzetten: host → **Configure → System → Services → SSH → START**
2. Verbinden vanaf pc:
```
ssh root@<management-IP host>
```
 
3. Netwerk en iSCSI-poort testen:
```
vmkping -I vmk2 192.168.40.100
nc -z 192.168.40.100 3260
```
 
4. iSCSI-sessie en devices controleren:
```
esxcli iscsi session list
esxcli storage core device list | grep -i synology
esxcli storage core path list | grep -i state
```
 
5. In vCenter per host controleren:

| Tab (iSCSI-adapter)   | Verwacht resultaat                     |
|-----------------------|----------------------------------------|
| Static Discovery      | IQN van de Synology verschijnt         |
| Devices               | Synology LUN, status **Attached**      |
| Paths                 | **Active**                             |
| Network Port Binding  | vmk2 → **Active**                      |
 
6. Beide hosts zien dezelfde datastore `iSCSI-DS01` → shared storage werkt ✅
7. SSH daarna weer op **STOP** zetten.

---

## Les 6 – Cluster

### Doel

De cluster verder opbouwen: hosts in een cluster met HA en DRS, twee resource pools met eigen gebruikers, Windows-templates (sysprep), vMotion en HA/FT testen en NIC teaming op de trunk.

### Cluster aanmaken

1. Rechtsklik op het **Datacenter** → **New Cluster**
2. Naam: `<clusternaam>`
3. **vSphere DRS** aan, **vSphere HA** aan, **vSAN** uit
4. Cluster image: `<gekozen optie>`

> **vSAN** bundelt de lokale schijven van de hosts tot één gedeelde datastore. Dat is hier niet nodig, want de shared storage komt van de Synology via iSCSI (les 5).

### EVC

EVC staat **uit**, op advies van de leerkracht.

> EVC zet alle hosts op dezelfde CPU-featureset, zodat vMotion werkt tussen hosts met verschillende CPU-generaties. Bij hosts met dezelfde CPU is dat niet nodig.

### Hosts toevoegen

1. Server 1 (`192.168.1.10`) en Server 2 (`192.168.1.11`) naar de cluster slepen
2. Bij de vraag over VM's/resource pools: **root resource pool**

### vSphere DRS en HA

**DRS** (cluster → Configure → Services → vSphere DRS → Edit)

| Instelling | Waarde |
|------------|--------|
| Automation level | Fully Automated |
| Migration threshold | 3 |

**HA** (cluster → Configure → Services → vSphere Availability → Edit)

| Instelling | Waarde |
|------------|--------|
| Host failure response | Restart VMs |
| Admission control | Host failures cluster tolerates: 1 |
| Heartbeat datastore | `iSCSI-DS01` |

### Resource pools en gebruikers

**Resource pools** (cluster → New Resource Pool)

| Pool | CPU limit | Memory limit |
|------|-----------|--------------|
| `<Pool-A>` | `<...>` | `<...>` |
| `<Pool-B>` | `<...>` | `<...>` |

**Gebruikers en groepen** (Administration → Single Sign On → Users and Groups, domein `vsphere.local`)

| Groep | Pool | Gebruikers | Wachtwoord |
|-------|------|------------|------------|
| `<GroepA>` | `<Pool-A>` | `<...>` | `Student1.` |
| `<GroepB>` | `<Pool-B>` | `<...>` | `Student1.` |

**Rol:** Administration → Roles → clone **Resource pool administrator (sample)** → `<rolnaam>`

**Rechten**

| Object | Groep | Rol | Propagate |
|--------|-------|-----|-----------|
| `<Pool-A>` | `<GroepA>` | `<rolnaam>` | ✔ |
| `<Pool-B>` | `<GroepB>` | `<rolnaam>` | ✔ |
| `iSCSI-DS01` | beide | Datastore consumer (sample) | – |
| Production / DMZ | beide | Network administrator (sample) | – |

### Windows-templates

1. ISO's van de fileserver uploaden naar de datastore
2. Nieuwe VM aanmaken → Windows Server installeren via de web console
3. **VMware Tools** installeren (VM → Guest OS → Install VMware Tools) → de VM herstarten
4. Controle: VM → Summary → *VMware Tools: Running*
5. VM afsluiten → **Template → Convert to Template**
6. Hetzelfde voor de Windows Client

> Geen manuele sysprep: vCenter voert sysprep zelf uit via de Customization Spec bij het uitrollen.

**Customization Spec** (Policies and Profiles → VM Customization Specifications → New)

| Instelling | Waarde |
|------------|--------|
| Name | `<specnaam>` |
| Target guest OS | Windows, nieuwe SID genereren ✔ |
| Computer name | Use the virtual machine name |
| Administrator password | `<...>` |
| Time zone | (GMT+01:00) Brussels |
| Network | Standard (DHCP) |
| Workgroup | `WORKGROUP` |

### NIC teaming (vSwitch1)

Een tweede fysieke link op de VLAN-trunk (vSwitch1), voor redundantie en load balancing. Als één kabel, NIC of switchpoort uitvalt, blijft het netwerk werken.

#### Bekabeling (Server 2)

Nieuwe NIC gevonden via vCenter → **Physical adapters**: vmnic3 stond op **1 Gbit/s** en hing nog aan geen vSwitch. De switchpoort staat in de tab **CDP**.

| NIC | Switchpoort | Functie |
|-----|-------------|---------|
| vmnic1 | Gi1/0/7 | vSwitch1 (trunk) |
| vmnic3 | Gi1/0/10 | vSwitch1 (trunk) – nieuw |

#### Configuratie fysieke switch

De nieuwe poort krijgt dezelfde trunk-config als Gi1/0/7. Er komt geen EtherChannel, want een standard vSwitch ondersteunt geen LACP.

```
enable
configure terminal
interface gi1/0/10
 description Trunk naar ESXi vSwitch1 (teaming)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 spanning-tree portfast trunk
 no shutdown
end
write memory
```

Controle met `show interfaces trunk`: Gi1/0/7, Gi1/0/8 en Gi1/0/10 staan op **trunking** met VLAN 10,20,30,40.

#### Configuratie vCenter

**1. vmnic3 toevoegen aan vSwitch1**

1. Host → **Configure → Networking → Virtual switches** → vSwitch1 → **Manage Physical Adapters**
2. **+** → vmnic3 → OK
3. vmnic1 en vmnic3 staan allebei onder **Active adapters**

Teaming policy van vSwitch1 (standaard gelaten):

| Instelling | Waarde |
|------------|--------|
| Load balancing | Route based on originating virtual port |
| Network failure detection | Link status only |
| Notify switches | Yes |
| Failback | Yes |

**2. Override op de Storage-portgroup (iSCSI)**

Een VMkernel-poort met iSCSI port binding mag maar **één actieve uplink** hebben, en **geen standby**.

1. vSwitch1 → `...` bij **Storage** → **Edit Settings** → **Teaming and failover**
2. **Override** aanvinken
3. Active: **vmnic1**. Standby: leeg. Unused: **vmnic3**

#### Testen

Via SSH op de host:

```
esxcli network vswitch standard portgroup policy failover get -p Storage
esxcli iscsi networkportal list
```

| Controle | Resultaat |
|----------|-----------|
| Failover Storage | Active `vmnic1`, Standby leeg, Unused `vmnic3` |
| iSCSI port binding vmk2 | **compliant** / **active** |
| vmhba64 Devices / Paths | Synology LUN zichtbaar |
| Datastore | `iSCSI-DS01` zichtbaar |

NIC teaming werkt ✅. SSH daarna weer op **STOP** zetten.

### Problemen

**Probleem 1: "Select a valid destination storage" bij het uitrollen van een VM uit de template**
- **Oorzaak:** de datastore `iSCSI-DS01` was verdwenen. Enkel de lokale datastores (`datastore1`, `datastore2`) waren nog zichtbaar.
- **Oplossing:** `<...>`

**Probleem 2: vmk2 "Not compliant" en vmk2 niet meer toe te voegen (NIC teaming)**
- **Oorzaak:** na het toevoegen van vmnic3 stond die bij de Storage-portgroup op **Standby**. Een iSCSI-binding met een standby uplink is niet compliant, en vCenter toont alleen compliant VMkernel-poorten bij **ADD**.
- **Oplossing:** in vCenter vmnic3 naar **Unused adapters** verplaatst. Daarna vmk2 opnieuw gebonden via SSH:
```
esxcli iscsi networkportal add -A vmhba64 -n vmk2
esxcli storage core adapter rescan -A vmhba64
```

**Probleem 3: `-u` bij `failover set` werkt niet als "unused" (NIC teaming)**
- **Oorzaak:** `-u` betekent `--use-vswitch` (de instellingen van de vSwitch overnemen), niet unused. Ook een NIC die je niet vermeldt, verdwijnt niet automatisch van standby.
- **Oplossing:** Unused via de vCenter GUI instellen.

### Nog te doen

- VM's uitrollen uit de templates en vMotion testen
- HA / Fault Tolerance testen
- DRS testen met CPU-load
