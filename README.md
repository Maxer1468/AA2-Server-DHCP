# AA2 Server DHCP amb Kea-Maxim Slobodyan


##  Desenvolupament de la Pràctica Pas a Pas


#### Preparament de la maquina virtual

![](/img/Captura%20de%20pantalla%202026-10-08%20174953.png)

### 1. Preparació de Wireshark al Client (Zorin OS)

Per poder capturar el trànsit de xarxa durant la negociació DHCP, primer hem preparat la màquina client instal·lant l'eina Wireshark.

```bash
sudo apt update
sudo apt install wireshark
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175014.png)
Durant la instal·lació del paquet `wireshark-common`, el sistema ens pregunta si volem permetre que els usuaris no superusuaris puguin capturar paquets. Seleccionem l'opció **<Sí>**.

![]()
Un cop instal·lat, iniciem l'aplicació des de la terminal per comprovar que la interfície gràfica s'obre correctament:

```bash
sudo wireshark
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175404.png)

---

### 2. Configuració de les interfícies de xarxa al Servidor (Netplan)

El servidor Ubuntu compta amb dues interfícies de xarxa:
* **`enp0s3`**: Mode **NAT** per mantenir la connexió a Internet.
* **`enp0s8`**: Mode **Xarxa Interna** per donar servei DHCP a la xarxa local.

Comprovem l'estat inicial de les interfícies i la taula de rutes:

```bash
whoami
sudo whoami
hostname
ip -br link
ip -4 -br addr
ip route
```


![](/img/Captura%20de%20pantalla%202026-10-08%20175004.png)

A continuació, editem el fitxer de configuració de Netplan (`/etc/netplan/50-cloud-init.yaml`):

```bash
ls -l /etc/netplan
sudo netplan get
sudo nano /etc/netplan/50-cloud-init.yaml
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175509.png)

**Configuració establerta a Netplan:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.169.22.1/24
```

Apliquem els canvis i verifiquem que la interfície `enp0s8` ha agafat la IP estàtica `192.169.16.1/24`:

```bash
sudo netplan generate
sudo netplan try
ip -4 -br addr
ip route
getent hosts ubuntu.com
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175605.png)
![](/img/Captura%20de%20pantalla%202026-10-08%20175516.png)

---

### 3. Instal·lació i configuració del servei Kea DHCP

Al servidor Ubuntu instal·lem el paquet del servidor Kea DHCP v4:

```bash
sudo apt install kea
```

Verifiquem la versió de Kea instal·lada i els serveis actius:

```bash
kea-dhcp4 -v
systemctl list-unit-files 'kea*'
```

![](<Captura de pantalla 2026-10-07 173154.png>)

Editem el fitxer de configuració principal del servei (`/etc/kea/kea-dhcp4.conf`):

```bash
sudo nano /etc/kea/kea-dhcp4.conf
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175556.png)

**Configuració aplicada a `kea-dhcp4.conf`:**

---

### 4. Validació, arrencada i comprovació de logs del servei

Abans d'iniciar el servidor, comprovem que la sintaxi JSON del fitxer de configuració sigui correcta mitjançant el paràmetre de prova (`-t`):

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175631.png)

Quan confirmem que el test retorna el resultat `INFO` correcte, procedim a reiniciar el servei, l'habilitem per a l'arrencada automàtica i verifiquem que està actiu (`active (running)`):

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl enable kea-dhcp4-server
sudo systemctl status kea-dhcp4-server --no-pager
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175656.png)

Per últim, comprovem els logs del servei amb `journalctl` per verificar que el dimoni està escoltant a la interfície `enp0s8` i ha carregat la subxarxa `192.169.16.0/24`:

```bash
sudo journalctl -u kea-dhcp4-server -b -e --no-pager -n 16
```

![](/img/Captura%20de%20pantalla%202026-10-08%20175642.png)
---
