UNITAT DIDÀCTICA: INTRODUCCIÓ AL NETDEVOPS AMB PYTHON
=====================================================

1\. Introducció Detallada a NetDevOps
-------------------------------------

NetDevOps no és una eina, sinó una **cultura operativa** que fusiona l'administració de xarxes (Networking) amb el desenvolupament de programari (DevOps). En un entorn tradicional, els canvis es realitzen manualment node per node, el que genera inconsistències i errors humans.

El cor de NetDevOps resideix en la **Infrastructure as Code (IaC)**. Això implica que l'estat desitjat de la xarxa es defineix en fitxers de text o codi Python, els quals s'executen per a configurar els dispositius de forma predictible i repetible.

**Per què Python?**  
Python s'ha consolidat com el llenguatge estàndard per a l'automatització de sistemes gràcies a la seua facilitat de lectura i a llibreries com `Netmiko`, `NAPALM` o `Nornir`, que gestionen les complexitats del protocol SSH i les diferències de sintaxi entre fabricants.

2\. Casos d'Ús Reals i Tècnics
------------------------------

### A. Gestió de Compliance i Auditoria

Les empreses han de complir amb normatives de seguretat. Un script de Python pot connectar-se a 50 switchos cada nit, descarregar la configuració i comparar-la amb un fitxer "mestre". Si hi ha canvis no autoritzats, l'script envia una alerta immediata.

### B. Orquestració de VLANs en Data Centers

En entorns virtualitzats, cal crear VLANs constantment. En lloc d'entrar a cada switch del rack, un script de NetDevOps rep una sol·licitud del departament de virtualització i crea la VLAN i l'etiquetatge (tagging) en tots els nodes afectats en menys de 10 segons.

### C. Validació de Salut Post-Canvi

Després d'una finestra de manteniment, l'enginyer ha de comprovar que tot funciona. L'automatització permet verificar en un segon si tots els veïnats BGP estan actius i si hi ha errors en les interfícies físiques.

3\. Disseny del Laboratori Virtual (Sense Privilegis d'Admin)
-------------------------------------------------------------

Aquest laboratori està dissenyat per a funcionar completament dins de **VirtualBox**, utilitzant adreçament IP que evita conflictes amb la xarxa del centre.

### Taula de Paràmetres de Xarxa

Element

Xarxa WAN (Adaptador Pont)

Xarxa Interna (Gestió OOB)

Rang Subxarxa

192.168.13.0/16

172.16.1.0/24

Assignació

DHCP (Router Classe)

Estàtica (Manual)

Finalitat

Accés a Internet / Descarregues

Comunicació Python <-> Router

4\. Activitats Detallades i Guiades (100% Complet)
--------------------------------------------------

### Pas 1: Descàrrega i Obtenció d'Imatges

Necessitareu descarregar les següents imatges ISO al vostre disc local:

*   **Control Node:** [Ubuntu Desktop 24.04.3 LTS](https://releases.ubuntu.com/24.04/) (Aprox. 4GB).
*   **Network Node:** [VyOS 1.5 Rolling](https://vyos.net/get/nightly-builds/) (Busqueu l'última ISO del 2026).

### Pas 2: Instal·lació de les Màquines Virtuals

Configuració de la VM Ubuntu (Control):

1.  Crear VM: Tipus Linux / Ubuntu 64-bit. RAM: 4GB. Disc: 25GB.
2.  **Xarxa Adaptador 1:** Mode "Adaptador Pont" (Bridged). Això us donarà IP 192.168.13.X.
3.  **Xarxa Adaptador 2:** Mode "Xarxa Interna". Nom: `netdevops-lab`.
4.  Instal·leu Ubuntu seguint els passos per defecte. Una vegada instal·lat, aneu a la configuració de xarxa i poseu a la segona targeta la IP `172.16.1.10/24`.

Configuració de la VM VyOS (Router):

1.  Crear VM: Tipus Linux / Debian 64-bit. RAM: 1GB. Disc: 4GB.
2.  **Xarxa Adaptador 1:** Mode "Xarxa Interna". Nom: `netdevops-lab`.
3.  Arrenqueu amb la ISO i entreu amb usuari `vyos` / clau `vyos`.
4.  Executeu `install image`, accepteu les opcions i reinicieu sense la ISO.
5.  Configureu el router amb les comandes següents:

configure
set interfaces ethernet eth0 address 172.16.1.1/24
set service ssh port 22
commit
save
exit
    

### Pas 3: Configuració de l'Entorn de Desenvolupament

A la VM Ubuntu, obriu la terminal i prepareu l'entorn de Python:

\# Actualitzar repositoris
sudo apt update && sudo apt install python3-pip python3-venv git -y

# Crear directori de treball i entorn virtual
mkdir ~/netdevops\_curs && cd ~/netdevops\_curs
python3 -m venv venv
source venv/bin/activate

# Instal·lar llibreria d'automatització
pip install netmiko
    

### Pas 4: Exercicis Pràctics d'Automatització

#### Exercici 4.1: Script de Lectura de Versió (get\_info.py)

Aquest script verifica que podem entrar al router i llegir dades.

from netmiko import ConnectHandler

dispositiu = {
    'device\_type': 'vyos',
    'host': '172.16.1.1',
    'username': 'vyos',
    'password': 'vyos',
}

print("Connectant al router...")
with ConnectHandler(\*\*dispositiu) as net\_connect:
    output = net\_connect.send\_command("show version")
    print("\\nResultat de la comanda:")
    print(output)
    

#### Exercici 4.2: Script de Configuració Masiva (setup\_network.py)

Crearem una interfície Loopback i canviarem el nom del router de forma programàtica.

from netmiko import ConnectHandler

dispositiu = {
    'device\_type': 'vyos',
    'host': '172.16.1.1',
    'username': 'vyos',
    'password': 'vyos',
}

config\_set = \[
    'set system host-name ROUTER-ACADEMIC-01',
    'set interfaces dummy dum0 address 10.255.255.1/32',
    'set system ntp server pool.ntp.org'
\]

with ConnectHandler(\*\*dispositiu) as net\_connect:
    print("Aplicant canvis...")
    # send\_config\_set entra en mode configure i fa commit automàtic
    resultat = net\_connect.send\_config\_set(config\_set)
    print(resultat)
    
    # Verificació final
    print(net\_connect.send\_command("show interfaces"))
    

#### Exercici 4.3: Script de Backup Automatitzat (backup\_manager.py)

Aquest script genera un fitxer amb la configuració actual i la marca temporal.

from netmiko import ConnectHandler
from datetime import datetime
import os

dispositiu = {
    'device\_type': 'vyos',
    'host': '172.16.1.1',
    'username': 'vyos',
    'password': 'vyos',
}

ara = datetime.now().strftime("%Y-%m-%d\_%H-%M")
nom\_fitxer = f"config\_backup\_{ara}.txt"

with ConnectHandler(\*\*dispositiu) as net\_connect:
    configuracio = net\_connect.send\_command("show configuration")
    
    with open(nom\_fitxer, "w") as f:
        f.write(configuracio)
    
    print(f"Backup realitzat amb èxit: {os.getcwd()}/{nom\_fitxer}")
    

© 2026 - Guia de NetDevOps per a Alumnes de Xarxes. Material acadèmic amb llicència Open-Access.
