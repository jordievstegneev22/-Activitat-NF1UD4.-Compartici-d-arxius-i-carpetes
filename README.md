# Activitat NF1 UD4. Compartició d'arxius i carpetes (NFS)

La documentació de tot el procés és a [AA1-NFS.md](AA1-NFS.md).

---

## NF1.UD4. AA1-NFS

### Presentació de l'activitat

El problema que planteja DevOptimize Solutions és la necessitat d'evitar que el codi font, documents de disseny, documentació tècnica, etc. estigui fora de control, en els equips dels desenvolupadors i tampoc volen la dependència d'un servei de cloud per compartir fitxers.

Per això, teniu l'encàrrec de mostrar al CEO de l'startup el funcionament d'un servidor NFS per a la compartició de fitxers en una xarxa local. El client ha insistit que actualment treballa sense un entorn d'autenticació centralitzada i que, de moment, no té previst implementar-ne cap.

Per mostrar al client com quedarà la solució proposada a partir de les seves demandes i poder mostrar també les seves limitacions, se t'encarrega fer una demostració del sistema.

Crearàs un servidor NFS (NFSv3) i un client Linux que consumeixi els recursos compartits. Hauràs de crear usuaris i grups per simular l'entorn del client i demostrar el control d'accés utilitzant les opcions d'exportació (`/etc/exports`) i els permisos del sistema de fitxers (`chmod`, `chown`).

### Durada de l'activitat

La durada prevista de l'activitat és 8 hores a classe.

### Objectius específics de l'activitat

La finalitat de la tasca és que l'alumnat configuri un servidor de fitxers NFS amb diversos nivells d'accés i prepari un client Linux per accedir automàticament als recursos compartits.

### Competències treballades

- **f)** Instal·lar, configurar i mantenir serveis multiusuari, aplicacions i dispositius compartits en un entorn de xarxa local, atenent les necessitats i els requeriments especificats.
- **g)** Realitzar proves funcionals en sistemes microinformàtics i xarxes locals, localitzant i diagnosticant disfuncions per comprovar-ne i ajustar-ne el funcionament.
- **o)** Utilitzar els mitjans de consulta disponibles, seleccionant el més adequat en cada cas, per resoldre en un temps raonable supòsits no coneguts i dubtes professionals.

### Resultats d'aprenentatge i criteris d'avaluació

**RA4.** Gestiona els recursos compartits del sistema, interpretant especificacions i determinant nivells de seguretat.

- 4.1 Reconeix la diferència entre permís i dret.
- 4.2 Identifica els recursos del sistema que es compartiran i en quines condicions.
- 4.3 Assigna permisos als recursos del sistema que es compartiran.
- 4.5 Utilitza l'entorn gràfic per compartir recursos.
- 4.6 Estableix nivells de seguretat per controlar l'accés del client als recursos compartits en xarxa.

### Continguts

- 4.1 Permisos i drets.
- 4.2 Compartir arxius i directoris a través de la xarxa.
- 4.3 Configuració dels permisos dels recursos compartits.

### Capacitats clau treballades

- Autonomia
- Innovació
- Relació interpersonal
- Organització del treball
- Responsabilitat
- Treball en equip
- Resolució de problemes

---

## Enunciat de l'activitat

### Requeriments previs

Cal que tingueu un servidor Linux (Ubuntu Server) instal·lat i configurat de l'activitat anterior. Cas que sigui necessari, torneu a desplegar el servidor a partir del fitxer `.OVA` que vau generar.

### Fase 1. Preparació de l'entorn

Per al client, utilitzarem Zorin OS 18 amb configuració de "xarxa NAT". El server tindrem la interfície per treballar amb "xarxa només-amb-amfitrió" i l'altra amb "xarxa NAT". Això ens permetrà que el client tingui accés a Internet i que el servidor pugui comunicar-se amb el client.

Ens assegurarem que ambdues màquines tinguin accés a Internet i que es puguin comunicar entre elles a través de la xarxa només-amb-amfitrió i actualitzarem els sistemes amb les últimes actualitzacions disponibles.

### Fase 2: Preparació del servidor

Abans de compartir res, hem de preparar els usuaris i els directoris al Servidor.

- **Creació de Grups:** Crear dos grups per al client: `devs` i `admins`.
- **Creació d'Usuaris:**
  - Crear un usuari `dev01` (membre del grup `devs`).
  - Crear un usuari `admin01` (membre del grup `admins`).
- **Creació de Directoris (al Servidor):**
  - Crear el directori per als projectes de desenvolupament: `/srv/nfs/dev_projects`
  - Crear el directori per a les eines d'administració: `/srv/nfs/admin_tools`
- **Permisos del Servidor (El punt clau):**
  - Es vol que els developers tinguin control total sobre els seus projectes.
  - Es vol que els administradors tinguin control sobre les seves eines.
  - En tots dos casos, l'usuari propietari serà `root`.

Com a pas final, s'instal·laran els paquets necessaris per al servei NFS al servidor i es configurarà l'exportació dels directoris amb les opcions adequades.

> **Nota:** Perquè aquesta pràctica funcioni correctament, heu de replicar aquests usuaris i grups al client, o, idealment, assegurar-vos que els UID i GID (els números d'identificació) coincideixin a les dues màquines.

### Fase 3: L'Exportació d'Administració (El Dilema del root_squash)

El client necessita que el directori `/srv/nfs/admin_tools` sigui accessible per l'equip d'administradors. A vegades, l'usuari root del client (que sou vosaltres, els consultors) necessitarà escriure en aquest directori per instal·lar eines. Aquí mostrarem un error típic i la seva solució.

#### Prova 1 (L'error comú)

1. Exportar el directori `/srv/nfs/admin_tools` amb les opcions `rw,sync`.
2. Des del client, muntar aquest recurs compartit a `/mnt/admin_tools`. Com a root del client, intentar crear un fitxer dins d'aquest directori muntat.
3. Verificar quin és el propietari del fitxer creat. Per què? Justificar la resposta amb l'explicació tècnica de `root_squash`.

#### Prova 2 (La Solució)

1. Modificar l'exportació del directori `/srv/nfs/admin_tools` per incloure l'opció `no_root_squash`.
2. Al client, desmuntar i tornar a muntar el recurs compartit.
3. Com a root del client, intentar crear un fitxer dins d'aquest directori muntat. Observeu quin és el propietari del fitxer creat aquesta vegada. Ha canviat alguna cosa? Justificar la resposta amb l'explicació tècnica de `no_root_squash`.

### Fase 4: L'Exportació de Desenvolupament (Permisos rw vs ro)

1. Editar `/etc/exports` per afegir dues exportacions per al mateix directori. El client vol que la xarxa d'administració (p.ex., `192.168.56.0/24`) hi pugui escriure, però que la xarxa de consultors (simularem que és una altra IP, p.ex., `192.168.56.150`) només pugui llegir.

   Si les dues màquines es veuen per xarxa NAT, useu una IP consistent amb el rang de la xarxa NAT. Per exemple, per exemple, `10.0.2.100`.

2. Des del client, muntar el recurs compartit a `/mnt/dev_projects` i provar d'escriure-hi com a usuari `dev01`. Hauria de funcionar.
3. Desmuntar el recurs i canviar manualment la IP del client a `192.168.56.100`. Tornar a provar d'escriure al directori muntat com a usuari `dev01` i comprovar que només funciona la lectura.
4. Canvieu d'usuari al client a `admin01` i torneu a provar d'escriure al directori muntat. Comproveu que no es pot escriure, ja que `admin01` no és membre del grup `devs` (permisos locals del sistema de fitxers).

### Fase 5: Muntatge Automàtic amb /etc/fstab

És evident que els usuaris no poden estar muntant manualment els recursos compartits cada vegada que reinicien el sistema. Per això, es configurarà el muntatge automàtic mitjançant el fitxer `/etc/fstab` al client.

1. Editar el fitxer `/etc/fstab` al client per afegir les entrades necessàries per muntar automàticament els recursos compartits NFS al directori `/mnt/admin_tools` i `/mnt/dev_projects` durant l'inici del sistema.
2. Executar la comanda `mount -a` per provar les entrades sense reiniciar.
3. Reiniciar el client i verificar que els recursos compartits s'han muntat correctament.

---

## Lliurament de l'activitat

L'activitat es pot lliurar de dues maneres:

1. **Documentació en format PDF** amb captures de pantalla i explicacions detallades del procés seguit, incloent les respostes a les preguntes plantejades i la conclusió raonada.
2. **Repositori de GitHub.** Amb la següent estructura de fitxers:

```text
.
├── README.md
├── AA1-NFS.md
└── images/
    ├── screenshot1.png
    └── screenshot2.png
```

On `README.md` contindrà l'enunciat de l'activitat, `AA1-NFS.md` serà el fitxer principal amb la descripció de l'activitat i les captures de pantalla es guardaran a la carpeta `images/`.

- Documenta tot el procés seguint les fases descrites anteriorment. Per les comandes, escriu la comanda com un codi, això et permetrà copiar posteriorment amb facilitat. Per exemple:

  ```bash
  sudo apt update
  ```

- Inclou captures de pantalla per demostrar el correcte funcionament de la prova de concepte. Per inserir una imatge en Markdown, utilitza la següent sintaxi:

  ```markdown
  ![Descripció de la imatge](images/screenshot1.png)
  ```

- Respon les preguntes plantejades en les diferents fases.
- Redacta una conclusió raonada amb les teves recomanacions per al client. Cal plantejar els avantatges de la solució proposada, així com les limitacions i possibles millores futures.

> **Nota:** Els termes 'redacta' i 'raonada' indiquen clarament que s'espera que escriguis amb les teves pròpies paraules i que aportis un pensament crític al teu treball. Pensa que si les conclusions les redacta directament una IA, ens plantejarem seriosament substiuir-te per ella.

---

## Material de suport

- Material propi del mòdul. NF1.UD4.A1 Compartició d'arxius i carpetes
- Ruiz, P. Capítulo 11: Instalar y configurar NFS en Ubuntu. *Sistemas Operativos en Red (Actualizado)*. SomeBooks.es, Juny 2022. enllaç
