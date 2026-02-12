NetDevOps representa l'aplicació de metodologies DevOps (Development and Operations) a l'administració d'infraestructures de xarxa. Aquest paradigma transiciona de la gestió manual (CLI, interacció humana directa) a l'automatització programàtica, tractant la configuració de xarxa com a codi (Infrastructure as Code - IaC).

## 1.1 El rol de Python en l'Automatització

Python s'estableix com l'estàndard en l'orquestració de xarxes degut a la seva extensibilitat i l'existència de biblioteques específiques per a la interacció amb dispositius de xarxa via SSH/API:

*   **Netmiko:** Llibreria multi-venedor que simplifica la gestió de connexions SSH, gestionant automàticament els prompts i l'execució de comandes privilegiades.
*   **Sintaxi:** Permet l'escriptura de scripts imperatius per a tasques repetitives i validació d'estats.

# 2. Anàlisi de Casos d'Ús Reals

L'automatització s'aplica principalment per garantir la consistència, reduir el temps d'operació (OpEx) i mitigar l'error humà.

## 2.1 Gestió de Compliment i Còpies de Seguretat (Backup & Compliance)

*   **Escenari Tècnic:** Necessitat de mantenir un històric de configuracions (*running-config*) de tot el parc de dispositius per a recuperació davant desastres.
*   **Implementació NetDevOps:** Execució programada (*cronjob*) d'un script Python que itera sobre un inventari d'IPs, estableix connexió SSH, extreu la configuració i l'emmagatzema en un repositori centralitzat amb control de versions (Git) i marca temporal.

## 2.2 Desplegament Massiu de Configuracions (Configuration Management)

*   **Escenari Tècnic:** Actualització de polítiques de seguretat (ex: canvi de servidors NTP, rotació de claus SNMP o creació d'usuaris locals) en múltiples nodes.
*   **Implementació NetDevOps:** L'script llegeix els paràmetres desitjats i aplica els canvis de forma concurrent o seqüencial, verificant posteriorment que el canvi s'ha aplicat correctament mitjançant anàlisi de la sortida (*parsing*).

## 2.3 Validació Operativa (State Validation)

*   **Escenari Tècnic:** Verificació prèvia i posterior a una finestra de manteniment.
*   **Implementació NetDevOps:** Automatització de la recollida de mètriques d'estat (estat de les interfícies, ús de CPU, taules d'enrutament). L'automatització compara l'estat actual amb un estat desitjat definit (*Golden State*) i alerta en cas de desviació.

# 3. Arquitectura del Laboratori Virtual

Ateses les restriccions de l'entorn (manca de permisos d'administrador al host i impossibilitat d'ús de contenidors Docker), es desplegarà una arquitectura basada en màquines virtuals (VM) sobre VirtualBox.

## 3.1 Topologia de Xarxa

Es defineixen dos segments de xarxa diferenciats:

1.  **Xarxa WAN (Accés):** `192.168.13.X/16` (DHCP). Proporciona accés a repositoris de programari.
2.  **Xarxa de Gestió (OOB - Out of Band):** `172.16.1.0/24`. Segment aïllat per a la comunicació entre el node de control i els dispositius de xarxa.

## 3.2 Components del Sistema

*   **Control Node (Client):** Ubuntu 24.04 LTS. Executarà l'entorn Python.
    *   IP Gestió: `172.16.1.10`
*   **Managed Node (Router):** VyOS (Rolling Release). Sistema operatiu de xarxa basat en Debian, amb CLI estil Juniper/Cisco.
    *   IP Gestió: `172.16.1.1`

# 4. Procediment d'Implementació

### Fase 1: Adquisició de Programari

Descarregar les imatges ISO necessàries (enllaços validats a 12/02/2026):

*   **Control Node:** Ubuntu 24.04.3 LTS Desktop.
    *   Font: Ubuntu Releases
*   **Router Node:** VyOS Rolling Release (Nightly Build).
    *   Font: VyOS Nightly Builds (Seleccionar l'última imatge .iso disponible).

### Fase 2: Configuració de l'Hipervisor (VirtualBox)

**2.1 Configuració VM "Control-Node" (Ubuntu)**

*   **Recursos:** 4096 MB RAM, 25 GB HDD.
*   **Interfícies de Xarxa:**
    *   Adaptador 1: "Adaptador Pont" (*Bridged Adapter*). Seleccionar la interfície física del host. (Assignació DHCP `192.168.13.X`).
    *   Adaptador 2: "Xarxa Interna" (*Internal Network*). Nom del segment: `netdevops-lab`.
    *   Mode Promiscu: Permetre tot (*Allow All*) per a l'Adaptador 2.

**2.2 Configuració VM "VyOS-Router"**

*   **Recursos:** 512 MB RAM, 4 GB HDD.
*   **Interfícies de Xarxa:**
    *   Adaptador 1: "Xarxa Interna". Nom del segment: `netdevops-lab`.

### Fase 3: Desplegament i Configuració Base

**3.1 Instal·lació del Control Node (Ubuntu)**

1.  Realitzar la instal·lació estàndard del sistema operatiu.
2.  Configurar l'adreçament IP estàtic per a la xarxa de gestió (Adaptador 2):
    *   IPv4: `172.16.1.10`
    *   Màscara: `255.255.255.0`
    *   Gateway: Buit.
3.  Instal·lar les dependències de desenvolupament:

```bash
sudo apt update
sudo apt install python3-pip python3-venv git -y
```

**3.2 Instal·lació del Managed Node (VyOS)**

1.  Iniciar la VM i accedir amb credencials per defecte (`vyos`/`vyos`).
2.  Executar la instal·lació persistent: `install image`. Seguir els passos per defecte i reiniciar.
3.  Configurar la interfície i el servei SSH:

```bash
# Entrar en mode configuració
configure

# 1. CONFIGURACIÓ LAN (Gestió NetDevOps - eth0)
set interfaces ethernet eth0 address '172.16.1.1/24'
set interfaces ethernet eth0 description 'LAN_NETDEVOPS'

# 2. CONFIGURACIÓ WAN (Xarxa de classe - eth1)
# Suposem que l'adaptador 2 de la VM està en mode Bridged/NAT cap a la 192.168.13.X
set interfaces ethernet eth1 address 'dhcp'
set interfaces ethernet eth1 description 'WAN_CLASSE'

# 3. SERVEIS I ACCÉS
set service ssh port '22'
set system ntp server pool.ntp.org

# 4. ENRUTAMENT I NAT (Opcional: per permetre que la xarxa interna isca a internet)
set nat source rule 100 outbound-interface 'eth1'
set nat source rule 100 source address '172.16.1.0/24'
set nat source rule 100 translation address 'masquerade'

# Aplicar i guardar
commit
save
exit
```

4.  **Validació:** Des de la terminal d'Ubuntu, executar `ping 172.16.1.1`. La connectivitat és requisit indispensable per continuar.

# 5. Desenvolupament d'Automatització (Python + Netmiko)

Totes les activitats es realitzen des del Control Node.

**Preparació de l'Entorn Virtual:**

```bash
mkdir ~/netdevops_project
cd ~/netdevops_project
python3 -m venv venv
source venv/bin/activate
pip install netmiko
```

### Activitat A: Extracció d'Informació (Read Operations)

**Objectiu:** Establir una sessió SSH i recuperar informació del sistema sense realitzar canvis.

**Codi:** `get_version.py`

```python
from netmiko import ConnectHandler

# Definició del diccionari de connexió
device = {
    'device_type': 'vyos',  # Driver específic per a VyOS
    'host':   '172.16.1.1',
    'username': 'vyos',
    'password': 'vyos',
    'port': 22,
}

# Instanciació de la connexió
connection = ConnectHandler(**device)

# Execució de comanda operativa
output = connection.send_command("show version")

print("Resultat de la consulta:")
print(output)

connection.disconnect()
```

### Activitat B: Automatització de Configuració (Write Operations)

**Objectiu:** Modificar l'estat del dispositiu de forma programàtica. Es canviarà el hostname i es crearà una interfície lògica (Loopback).

**Codi:** `configure_device.py`

```python
from netmiko import ConnectHandler

device = {
    'device_type': 'vyos',
    'host':   '172.16.1.1',
    'username': 'vyos',
    'password': 'vyos',
}

# Llista de comandes de configuració (Sintaxi VyOS)
config_set = [
    'set system host-name ROUTER-LAB-01',
    'set interfaces dummy dum0 address 10.0.0.1/32',
    'set interfaces dummy dum0 description "Managed by Python"'
]

connection = ConnectHandler(**device)

# send_config_set gestiona automàticament els modes (configure -> set -> commit -> exit)
output = connection.send_config_set(config_set)

print("Registre de canvis aplicats:")
print(output)

# Verificació post-canvi
verify = connection.send_command("show interfaces")
print("\nEstat actual de les interfícies:")
print(verify)

connection.disconnect()
```

### Activitat C: Procediment de Backup (File Operations)

**Objectiu:** Extreure la configuració completa i persistir-la en un fitxer local amb marcatge temporal per a auditoria.

**Codi:** `backup_routine.py`

```python
from netmiko import ConnectHandler
import datetime
import os

device = {
    'device_type': 'vyos',
    'host':   '172.16.1.1',
    'username': 'vyos',
    'password': 'vyos',
}

# Generació de nom de fitxer dinàmic
timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
filename = f"config_backup_{timestamp}.txt"

connection = ConnectHandler(**device)

# Extracció de la configuració
# Nota: VyOS utilitza 'show configuration' per veure l'estructura completa
config_data = connection.send_command("show configuration")

# Escriptura en disc
with open(filename, 'w') as file:
    file.write(config_data)

print(f"Operació completada. Arxiu generat: {os.path.abspath(filename)}")

connection.disconnect()
```

# 6. Conclusions Tècniques

La implementació d'aquest laboratori demostra la viabilitat de l'automatització de xarxes en entorns restringits mitjançant l'ús d'eines de codi obert. L'arquitectura desplegada permet l'escalabilitat horitzontal (afegir més routers VyOS a la xarxa interna) i l'evolució cap a scripts complexos que incloguin bucles de control, gestió d'errors (Try/Except) i lectura d'inventaris externs (CSV/YAML).
