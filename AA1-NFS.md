# AA1. Compartició de carpetes amb NFS

## Índex

1. [Entorn de treball](#entorn-de-treball)
2. [Fase 1. Preparació de l'entorn](#fase-1-preparació-de-lentorn)
3. [Fase 2. Preparació del servidor](#fase-2-preparació-del-servidor)
4. [Fase 3. root_squash](#fase-3-root_squash)
5. [Fase 4. rw vs ro](#fase-4-rw-vs-ro)
6. [Incidències i decisions](#incidències-i-decisions)
7. [Fase 5. fstab](#fase-5-fstab)
8. [Conclusió](#conclusió)

## Entorn de treball

| Màquina  | Sistema            | Hostname       | Usuari   | enp0s3 (NAT) | enp0s8 (Només amfitrió) |
|----------|--------------------|----------------|----------|--------------|-------------------------|
| Servidor | Ubuntu Server 26.04 | `nom-servidor` | `usuari` | 10.0.2.5     | 192.168.56.101          |
| Client   | Zorin OS 18        | `nom-client`   | `usuari` | 10.0.2.3     | 192.168.56.104          |

| Grup     | GID  | Usuari    | UID  |
|----------|------|-----------|------|
| `devs`   | 2001 | `dev01`   | 3001 |
| `admins` | 2002 | `admin01` | 3002 |

| Carpeta al servidor        | Propietari    | Permisos | Punt de muntatge al client |
|----------------------------|---------------|----------|----------------------------|
| `/srv/nfs/dev_projects`    | `root:devs`   | 770      | `/mnt/dev_projects`        |
| `/srv/nfs/admin_tools`     | `root:admins` | 770      | `/mnt/admin_tools`         |

---

## Fase 1. Preparació de l'entorn

### 1.1 Xarxa de VirtualBox

Les dues màquines tenen dos adaptadors de xarxa:

- Adaptador 1: **NAT**, per tenir accés a Internet.
- Adaptador 2: **Només amfitrió** (*host-only*), perquè el servidor i el client es vegin entre ells a la xarxa 192.168.56.0/24.

<!-- FALTA CAPTURA: configuració de xarxa de VirtualBox (servidor i client) -->

### 1.2 Hostname del servidor

```bash
sudo hostnamectl set-hostname nom-servidor
hostnamectl
```

![Canvi de hostname al servidor](images/f1-01-hostname-servidor.png)

La comanda canvia el nom de la màquina. A la captura es veu `Static hostname: nom-servidor` i el sistema Ubuntu 26.04.1 LTS.

### 1.3 Hostname del client

```bash
sudo hostnamectl set-hostname nom-client
hostnamectl
```

![Canvi de hostname al client](images/f1-02-hostname-client.png)

Es fa el mateix al client. La captura mostra `Static hostname: nom-client` i el sistema Zorin OS 18.1.

### 1.4 Idioma del servidor

```bash
sudo apt install language-pack-ca -y
```

![Instal·lació del paquet d'idioma català](images/f1-03-language-pack-servidor.png)

S'instal·la el paquet d'idioma català. Es veu que es generen els locales `ca_ES.UTF-8` i altres variants.

```bash
sudo update-locale LANG=ca_ES.UTF-8
```

![Canvi de l'idioma per defecte](images/f1-04-update-locale-servidor.png)

Aquesta comanda posa el català com a idioma per defecte del sistema. No mostra cap missatge si va bé.

```bash
sudo dpkg-reconfigure keyboard-configuration
```

![Configuració del teclat](images/f1-05-teclat-dialeg-servidor.png)

![Resultat de la configuració del teclat](images/f1-06-teclat-reconfigure-servidor.png)

Es torna a configurar el teclat. A la primera captura es tria el model "Generic 105-key PC" i a la segona es veu que es regenera l'`initrd`.

```bash
locale
```

![Comprovació de l'idioma al servidor](images/f1-07-locale-servidor.png)

La comanda mostra l'idioma actual. A la captura encara surt `es_ES.UTF-8`, perquè el canvi s'aplica quan es torna a iniciar la sessió.

Més endavant, la sortida de les comandes del servidor ja surt en català (per exemple, a la captura de l'`apt upgrade`).

### 1.5 Idioma del client

```bash
locale
```

![Comprovació de l'idioma al client](images/f1-08-locale-client.png)

Al client l'idioma és `es_ES.UTF-8`. També surten avisos de `Cannot set LC_CTYPE`, perquè el locale per defecte no està generat.

### 1.6 Adreces IP

```bash
ip a
```

![Adreces IP del client](images/f1-09-ip-a-client.png)

La comanda mostra les interfícies de xarxa. El client té `10.0.2.3` a `enp0s3` (NAT) i `192.168.56.104` a `enp0s8` (només amfitrió).

<!-- FALTA CAPTURA: ip a al servidor (10.0.2.5 i 192.168.56.101) -->

### 1.7 Ping a Internet

```bash
ping -c 3 google.com
```

![Ping a Internet des del servidor](images/f1-10-ping-internet-servidor.png)

![Ping a Internet des del client](images/f1-11-ping-internet-client.png)

Les dues màquines envien 3 paquets i en reben 3 (0% de pèrdua). Això demostra que tenen accés a Internet per l'adaptador NAT.

### 1.8 Ping entre les màquines

Del servidor al client:

```bash
ping -c 3 192.168.56.104
```

![Ping del servidor al client](images/f1-12-ping-servidor-a-client.png)

Del client al servidor:

```bash
ping -c 3 192.168.56.101
```

![Ping del client al servidor](images/f1-13-ping-client-a-servidor.png)

Els dos pings funcionen sense pèrdua de paquets. Les dues màquines es veuen per la xarxa només amfitrió.

### 1.9 Actualització del sistema

```bash
sudo apt update && sudo apt upgrade -y
```

![Actualització del servidor](images/f1-14-apt-upgrade-servidor.png)

Al servidor tots els paquets ja estan actualitzats (0 paquets per actualitzar).

![Actualització del client](images/f1-15-apt-upgrade-client.png)

Al client es veu que hi ha 218 paquets per actualitzar i que comença l'actualització.

---

## Fase 2. Preparació del servidor

### 2.1 Crear els grups al servidor

```bash
sudo groupadd -g 2001 devs
sudo groupadd -g 2002 admins
```

![groupadd al servidor](images/f2-01-groupadd-servidor.png)

La comanda crea un grup amb un GID concret. Al servidor surt que els dos grups ja existeixen, perquè venien d'una activitat anterior (vegeu [Incidències i decisions](#incidències-i-decisions)).

### 2.2 Crear els usuaris al servidor

```bash
sudo useradd -m -u 3001 -g devs -s /bin/bash dev01
sudo useradd -m -u 3002 -g admins -s /bin/bash admin01
```

![useradd al servidor](images/f2-02-useradd-servidor.png)

`useradd` crea l'usuari amb el seu UID, el grup principal i la shell. Al servidor els usuaris també ja existien.

### 2.3 Contrasenyes al servidor

```bash
sudo passwd dev01
sudo passwd admin01
```

![passwd al servidor](images/f2-03-passwd-servidor.png)

Es posa una contrasenya a cada usuari. Les dues s'actualitzen correctament.

### 2.4 Comprovació d'usuaris i grups al servidor

```bash
id dev01
id admin01
getent group devs admins
```

![id i getent al servidor](images/f2-04-id-usuaris-servidor.png)

`id` mostra l'UID i els grups d'un usuari, i `getent` mostra el GID dels grups. Els UID són correctes (3001 i 3002) i `admins` té el GID 2002. Però en aquesta captura el grup `devs` encara té el **GID 1001**, i no 2001.

### 2.5 Crear els grups i usuaris al client

```bash
sudo groupadd -g 2001 devs
sudo groupadd -g 2002 admins
```

![groupadd al client](images/f2-05-groupadd-client.png)

```bash
sudo useradd -m -u 3001 -g devs -s /bin/bash dev01
sudo useradd -m -u 3002 -g admins -s /bin/bash admin01
```

![useradd al client](images/f2-06-useradd-client.png)

Al client els grups i els usuaris es creen sense errors. A la captura també hi ha una comanda repetida de `dev01` que dona "el usuario ya existe", però no canvia res.

```bash
sudo passwd dev01
sudo passwd admin01
```

![passwd al client](images/f2-07-passwd-client.png)

Les contrasenyes s'actualitzen correctament. El sistema avisa que tenen menys de 8 caràcters.

### 2.6 Comprovació d'usuaris i grups al client

```bash
id dev01
id admin01
getent group devs admins
```

![id i getent al client](images/f2-08-id-usuaris-client.png)

Al client: `dev01` té UID 3001 i GID 2001, `admin01` té UID 3002 i GID 2002.

**Comparació servidor i client**

| | Servidor (captura 2.4) | Client (captura 2.6) |
|---|---|---|
| UID `dev01` | 3001 | 3001 |
| UID `admin01` | 3002 | 3002 |
| GID `devs` | **1001** | 2001 |
| GID `admins` | 2002 | 2002 |

Els UID coincideixen, però el GID de `devs` no. NFS identifica els usuaris i grups pel número, no pel nom. Per això s'ha de corregir el GID del servidor (vegeu [Incidències i decisions](#incidències-i-decisions)).

### 2.7 Crear les carpetes compartides

```bash
sudo mkdir -p /srv/nfs/dev_projects /srv/nfs/admin_tools
```

![mkdir al servidor](images/f2-09-mkdir-servidor.png)

Es creen les dues carpetes que es compartiran. L'opció `-p` crea també les carpetes pare si no existeixen.

### 2.8 Propietari i grup

```bash
sudo chown root:devs /srv/nfs/dev_projects
sudo chown root:admins /srv/nfs/admin_tools
```

![chown al servidor](images/f2-10-chown-servidor.png)

Cada carpeta passa a tenir `root` com a propietari i el grup corresponent.

### 2.9 Permisos

```bash
sudo chmod 770 /srv/nfs/dev_projects /srv/nfs/admin_tools
```

![chmod al servidor](images/f2-11-chmod-servidor.png)

Amb `770`, el propietari i el grup tenen tots els permisos i la resta d'usuaris no en té cap.

### 2.10 Comprovació de permisos

```bash
ls -ld /srv/nfs/*
```

![ls -ld al servidor](images/f2-12-ls-ld-servidor.png)

Es veu `drwxrwx---` a les dues carpetes, amb `root admins` i `root devs`. Els permisos i propietaris són correctes.

### 2.11 Instal·lació de NFS

Al servidor:

```bash
sudo apt install nfs-kernel-server -y
```

![Instal·lació de nfs-kernel-server](images/f2-13-nfs-kernel-server.png)

`nfs-kernel-server` és el servidor NFS. A la captura surt que ja estava instal·lat a la versió més recent.

Al client:

```bash
sudo apt install nfs-common -y
```

![Instal·lació de nfs-common al client](images/f2-14-nfs-common-client.png)

`nfs-common` conté les eines per muntar carpetes NFS. Al client s'instal·la juntament amb `rpcbind` i altres dependències.

<!-- FALTA CAPTURA: instal·lació de nfs-common al servidor -->

### 2.12 Fitxer /etc/exports inicial

```bash
sudo tee /etc/exports > /dev/null << 'EOF'
/srv/nfs/admin_tools 192.168.56.0/24(rw,sync,no_subtree_check)
/srv/nfs/dev_projects 192.168.56.0/24(rw,sync,no_subtree_check)
/srv/nfs/dev_projects 192.168.56.150(ro,sync,no_subtree_check)
EOF
```

![Escriptura de /etc/exports](images/f2-15-exports-inicial.png)

```bash
cat /etc/exports
```

![Contingut de /etc/exports](images/f2-16-cat-exports.png)

El fitxer `/etc/exports` diu quines carpetes es comparteixen, amb quines IP i amb quines opcions. `admin_tools` i `dev_projects` es comparteixen amb lectura i escriptura (`rw`) per a tota la xarxa 192.168.56.0/24, i `dev_projects` només en lectura (`ro`) per a la IP 192.168.56.150.

### 2.13 Aplicar i comprovar les exportacions

```bash
sudo exportfs -ra
sudo exportfs -v
```

![exportfs -ra i exportfs -v](images/f2-17-exportfs.png)

`exportfs -ra` torna a llegir `/etc/exports` i aplica els canvis. `exportfs -v` mostra les carpetes compartides amb totes les opcions. Es veu que per defecte s'aplica `root_squash`.

### 2.14 Estat del servei

```bash
sudo systemctl restart nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

![Estat del servei NFS](images/f2-18-systemctl-status.png)

Es reinicia el servei i es comprova el seu estat. Surt `active (exited)` i `enabled`, que és normal en aquest servei: vol dir que s'ha iniciat bé i que arrencarà amb el sistema.

### 2.15 Punts de muntatge al client

```bash
sudo mkdir -p /mnt/admin_tools /mnt/dev_projects
```

![Creació dels punts de muntatge](images/f2-19-mkdir-punts-muntatge-client.png)

Al client es creen les carpetes on es muntaran les carpetes compartides.

---

## Fase 3. root_squash

### Prova 1. admin_tools amb rw,sync

`admin_tools` s'exporta amb `rw,sync,no_subtree_check` (sense `no_root_squash`).

```bash
sudo mount -t nfs 192.168.56.101:/srv/nfs/admin_tools /mnt/admin_tools
```

![Muntatge d'admin_tools](images/f3-01-mount-admin-tools.png)

El client munta la carpeta `admin_tools` del servidor. No surt cap error, per tant s'ha muntat bé.

```bash
sudo touch /mnt/admin_tools/prova1_root.txt
ls -l /mnt/admin_tools
```

![touch com a root amb root_squash](images/f3-02-root-squash-error.png)

Es prova de crear un fitxer com a root al client. El `touch` dona **"Permiso denegado"** i el `ls -l` també.

#### Resposta (Prova 1)
<!-- ESCRIU AQUÍ LA TEVA RESPOSTA -->

### Prova 2. Afegir no_root_squash

Al servidor s'edita `/etc/exports`:

```bash
sudo nano /etc/exports
```

Abans del canvi:

![/etc/exports abans del canvi](images/f3-03-exports-abans-prova2.png)

Després del canvi:

![/etc/exports amb no_root_squash](images/f3-04-exports-no-root-squash.png)

S'afegeix l'opció `no_root_squash` a la línia d'`admin_tools`.

```bash
cat /etc/exports
sudo exportfs -ra
sudo exportfs -v
```

![exportfs amb no_root_squash](images/f3-05-exportfs-no-root-squash.png)

Es comprova el fitxer i s'apliquen els canvis. A `exportfs -v` ara surt `no_root_squash` a `admin_tools`, i `dev_projects` continua amb `root_squash`.

Al client es desmunta i es torna a muntar la carpeta:

```bash
sudo umount /mnt/admin_tools
sudo mount -t nfs 192.168.56.101:/srv/nfs/admin_tools /mnt/admin_tools
```

![Remuntatge d'admin_tools](images/f3-06-remuntar-admin-tools.png)

Les dues comandes funcionen sense errors.

```bash
sudo touch /mnt/admin_tools/prova2_root.txt
ls -l /mnt/admin_tools
```

![touch com a root amb no_root_squash](images/f3-07-touch-no-root-squash.png)

Ara el `touch` com a root **no dona cap error**, per tant el fitxer es crea. El `ls -l` sense `sudo` dona "Permiso denegado", perquè l'usuari `usuari` no és del grup `admins`.

<!-- FALTA CAPTURA: sudo ls -l /mnt/admin_tools (o ls -l al servidor) que mostri que prova2_root.txt és de root -->

#### Resposta (Prova 2)
<!-- ESCRIU AQUÍ LA TEVA RESPOSTA -->

---

## Fase 4. rw vs ro

El fitxer `/etc/exports` per a `dev_projects` té dues línies:

```bash
/srv/nfs/dev_projects 192.168.56.0/24(rw,sync,no_subtree_check)
/srv/nfs/dev_projects 192.168.56.150(ro,sync,no_subtree_check)
```

Tota la xarxa té lectura i escriptura, però la IP 192.168.56.150 només té lectura.

### 4.1 Muntar dev_projects (IP 192.168.56.104)

```bash
sudo mount -t nfs 192.168.56.101:/srv/nfs/dev_projects /mnt/dev_projects
```

![Muntatge de dev_projects](images/f4-01-mount-dev-projects.png)

El client munta `dev_projects` sense errors.

### 4.2 dev01 escriu

```bash
su - dev01
touch /mnt/dev_projects/fitxer_dev01.txt
ls -l /mnt/dev_projects
exit
```

![dev01 crea un fitxer](images/f4-02-dev01-escriu.png)

`dev01` és del grup `devs` i la IP .104 té `rw`, per això pot crear el fitxer. El `ls -l` mostra `fitxer_dev01.txt` amb propietari `dev01` i grup `devs`.

### 4.3 admin01 intenta escriure

```bash
su - admin01
touch /mnt/dev_projects/fitxer_admin01.txt
```

![admin01 no pot escriure](images/f4-03-admin01-permis-denegat.png)

El `touch` dona **"Permiso denegado"**. NFS permet escriure (la IP .104 té `rw`), però `admin01` no és del grup `devs` i la carpeta té permisos `770`.

### 4.4 Canvi d'IP del client

```bash
sudo umount /mnt/dev_projects
```

![Desmuntatge de dev_projects](images/f4-04-umount-dev-projects.png)

Primer es desmunta la carpeta.

```bash
sudo ip addr del 192.168.56.104/24 dev enp0s8
sudo ip addr add 192.168.56.150/24 dev enp0s8
ip a
```

![Canvi d'IP del client](images/f4-05-canvi-ip-client.png)

Es treu la IP .104 de la interfície `enp0s8` i se'n posa una de nova. **Atenció:** a la captura la comanda que es veu és `sudo ip addr add 193.168.56.150/24` i `ip a` mostra `inet 193.168.56.150/24`, no `192.168.56.150`.

### 4.5 Tornar a muntar i provar amb dev01

```bash
sudo mount -t nfs 192.168.56.101:/srv/nfs/dev_projects /mnt/dev_projects
```

![Muntatge de dev_projects amb la nova IP](images/f4-06-mount-dev-projects-nova-ip.png)

El muntatge funciona sense errors.

```bash
su - dev01
ls -l /mnt/dev_projects
ls -l /mnt/dev_projects/prova_consultor.txt
```

![dev01 llegeix dev_projects](images/f4-07-dev01-llegeix.png)

`dev01` pot llegir la carpeta i veu `fitxer_dev01.txt`. El fitxer `prova_consultor.txt` no existeix.

<!-- FALTA CAPTURA: touch de dev01 amb l'error "Read-only file system" -->

### 4.6 Tornar a posar la IP .104

```bash
sudo umount /mnt/dev_projects /mnt/admin_tools
sudo ip addr del 192.168.56.150/24 dev enp0s8
sudo ip addr add 192.168.56.104/24 dev enp0s8
```

![Restaurar la IP del client](images/f4-08-restaurar-ip.png)

Es desmunten les carpetes i es torna a la IP original. A la captura `ip addr del` dona "Address not found" i `ip addr add` dona "Address already assigned": la IP .150 ja no hi era i la .104 ja estava posada.

```bash
ip a
```

![IP del client restaurada](images/f4-09-ip-a-restaurada.png)

`enp0s8` torna a tenir `192.168.56.104/24`.

### 4.7 Diferència entre els dos errors

- **"Permission denied" (admin01):** el bloqueig ve dels **permisos locals** de la carpeta al servidor. NFS deixa escriure, però l'usuari no és del grup `devs` i la carpeta és `770`.
- **"Read-only file system" (dev01 amb la IP .150):** el bloqueig ve de **NFS**. L'usuari sí que té permisos a la carpeta, però el servidor exporta la carpeta només en lectura (`ro`) per a aquesta IP.

En resum: per poder escriure, NFS (segons la IP) i els permisos de la carpeta (segons l'usuari) han de permetre-ho tots dos.

---

## Incidències i decisions

### 1. El grup devs ja existia amb un altre GID

Al servidor el grup `devs` ja existia d'una activitat anterior amb el GID 1001 (es veu a la [captura 2.4](#24-comprovació-dusuaris-i-grups-al-servidor)). Al client tenia el GID 2001. Com que NFS fa servir els números, els dos GID han de ser iguals. Ho vaig corregir així:

```bash
sudo groupmod -g 2001 devs
sudo chgrp devs /srv/nfs/dev_projects
```

Després de canviar el GID, vaig tornar a fer `chgrp` perquè la carpeta quedés amb el grup correcte.

<!-- FALTA CAPTURA: groupmod, chgrp i getent group devs amb el GID 2001 -->

### 2. Xarxa del client

L'enunciat diu que el client només fa servir NAT. Però a la Fase 4 el client necessita una IP de la xarxa 192.168.56.X. Per això vaig posar NAT i només amfitrió també al client.

### 3. IP per a la prova de només lectura

L'enunciat diu que cal canviar la IP del client a 192.168.56.100. Però aquesta IP és dins de 192.168.56.0/24, que té `rw`, i no es veuria el bloqueig. Vaig fer servir 192.168.56.150, que és la IP dels consultors amb `ro`.

### 4. Ordre de les proves de la Fase 4

Vaig fer la prova d'`admin01` **abans** de canviar la IP. Així l'error "Permission denied" només pot venir dels permisos locals, i no de la IP.

### 5. Problemes amb dpkg

Durant les instal·lacions vaig tenir dos problemes:

- `dpkg` estava bloquejat perquè les actualitzacions automàtiques (`unattended-upgrades` i `dracut`) s'estaven executant.
- Va sortir l'error de "dpkg interromput". Ho vaig arreglar amb:

```bash
sudo dpkg --configure -a
```

<!-- FALTA CAPTURA: errors de dpkg -->

---

## Fase 5. fstab

### 5.1 Afegir les línies a /etc/fstab

```bash
sudo tee -a /etc/fstab > /dev/null << 'EOF'
192.168.56.101:/srv/nfs/admin_tools /mnt/admin_tools nfs defaults,_netdev 0 0
192.168.56.101:/srv/nfs/dev_projects /mnt/dev_projects nfs defaults,_netdev 0 0
EOF
```

![Afegir línies a /etc/fstab](images/f5-01-fstab-afegir.png)

S'afegeixen dues línies al final de `/etc/fstab` perquè les carpetes es muntin soles en arrencar. L'opció `_netdev` indica que és un recurs de xarxa i que s'ha d'esperar que la xarxa estigui activa.

### 5.2 Comprovar el fitxer

```bash
cat /etc/fstab
```

![Contingut de /etc/fstab](images/f5-02-cat-fstab.png)

Les dues línies noves surten al final del fitxer, després de les línies del sistema.

### 5.3 Muntar i comprovar

```bash
sudo mount -a
df -h | grep nfs
```

![mount -a i df -h](images/f5-03-mount-a-df.png)

`mount -a` munta tot el que hi ha a `/etc/fstab`. Surt un avís que recomana `systemctl daemon-reload`, però no és un error. `df -h` mostra les dues carpetes NFS muntades.

### 5.4 Reinici

```bash
sudo reboot
```

![Reinici del client](images/f5-04-reboot.png)

Es reinicia el client per comprovar que el muntatge automàtic funciona.

### 5.5 Comprovació després del reinici

```bash
df -h | grep nfs
ls -l /mnt/admin_tools /mnt/dev_projects
```

![Comprovació després del reinici](images/f5-05-df-despres-reinici.png)

Després del reinici, `df -h` mostra les dues carpetes muntades sense fer res. Per tant, `/etc/fstab` funciona. El `ls -l` amb l'usuari `usuari` dona "Permiso denegado", perquè aquest usuari no és de `devs` ni d'`admins`.

---

## Conclusió

### Avantatges
<!-- ESCRIU AQUÍ -->

### Limitacions
<!-- ESCRIU AQUÍ -->

### Millores i recomanacions per al client
<!-- ESCRIU AQUÍ -->
