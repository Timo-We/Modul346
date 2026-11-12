# KN01
## Theorie
### Virtualisierung & Hypervisor
#### Virtualisierung

Virtualisierung macht, dass das Betriebssystem und Applikationen abgespalten von der Hardware sind. Ein Hypervisor ermöchtlicht das ganze, er teilt verfügbare Ressourcen, z.B CPU, RAM, Disk, Netzwerk, passend zu verschiedensten kleinen VMs zu. So wird Hardware optimal genutzt und die VMs können innerhalb von sekunden erstellt, geklont und gelöscht werden. Durch diese Vorgehensweise wird Cloud Computing erst möglich gemacht

##### Typ1 vs Typ2
Ein Typ1 Hypervisor läuft direkt auf der Hardware, die Performance ist beim Typ1 deutlich höher da sich kein OS zwischen Hypervisor und Hardware befindet, praktisch für Cloud-Anbieter oder Rechenzentren.Ein Typ2 Hypervisor ist eine Applikation auf einem Betriebssystem, die Performance ist dafür schlechter. Wird z.B verwendet auf Entwickler-Laptops

#### Multipass
Bei Multipass handelt sich es um eine Verwaltungsschicht und nicht um einen Hypervisor. Es ist Betriebssystem-Abhängig welche Technik die VM nutzt. Multipass kann VMs nur mit einem Befehl starten und verwalten, die wirkliche visualisierung wird dem Backend überlassen.
Multipass läuft in diesem Projekt (KN01) als Typ2.

#### Hyperscaler
Hyperscaler nehmen das Prinzip von einem Hypervisor und vergrössert es stark. Es nimmt viele Physische Server mit einem Typ1 Hypervisor zusammen und macht diese zu einer Einheit.

### Cloud-init
#### Was ist Cloud-init?
Cloud-init richtet eine Instanz beim ersten Start automatisch ein. Man gibt bei der Erstellung eine YAML Datei mit (bei AWS im Feld user data) und cloud-init arbeitet die dann beim ersten Boot ab, also z.B Benutzer anlegen, Pakete installieren, Dateien schreiben oder Befehle ausführen. Es ist auf fast allen Cloud-Images schon vorinstalliert und funktioniert bei allen grossen Anbietern gleich, die selbe Datei läuft in AWS, Azure und auch lokal mit Multipass. Wenn man z.B 10 Server mit der gleichen Grundeinrichtung braucht, muss man die nicht von Hand oder mit einem Bash-Skript einrichten sondern gibt einfach jedem die gleiche Datei mit.
Cloud-init läuft aber nur beim ersten Boot. Will man an einer bestehenden Instanz was ändern macht man einfach eine neue, das ist so gewollt in der Cloud (ersetzen statt reparieren). Für Änderungen an laufenden Systemen nimmt man Ansible.

#### Aufbau
Die Datei muss immer mit #cloud-config anfangen, wenn die Zeile fehlt wird alles einfach ignoriert ohne Fehlermeldung. Danach kommen die Blöcke, z.B so:

```yaml
#cloud-config
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAA... aws-key
ssh_pwauth: false
disable_root: true
package_update: true
packages:
  - apache2
write_files:
  - path: /var/www/html/info.php
    content: |
      <?php phpinfo(); ?>
runcmd:
  - systemctl restart apache2
```

Mit users legt man Benutzer an mit sudo Rechten und SSH-Keys, der Key muss dabei ohne Zeilenumbrüche drin sein. ssh_pwauth und disable_root sind für die Härtung, so kann man sich nur noch per Schlüssel einloggen und nicht direkt als root, das sollte in jedes Template rein. package_update und packages ersetzen apt update und apt install. Mit write_files kann man Dateien erstellen, der Inhalt kommt nach content: |. runcmd führt am Schluss noch Befehle aus, aber nur einmal.
Man sollte immer zuerst schauen ob es einen passenden Block gibt und erst dann runcmd brauchen. Wenn man alles in runcmd reinpackt ist es eigentlich nur noch ein Bash-Skript in YAML und man verliert die Übersicht.

#### Reihenfolge
Die Reihenfolge gibt cloud-init selber vor, es ist egal wie es in der Datei steht. Zuerst kommt write_files, dann users, danach runcmd und erst ganz am Schluss package_update und packages. Das heisst man kann bei write_files nicht mit owner: einen Benutzer angeben der erst bei users erstellt wird, weil der zu dem Zeitpunkt noch gar nicht existiert und die Datei dann nicht geschrieben wird. Ausserdem sind die Pakete beim runcmd noch nicht installiert, wenn man eins dort braucht muss man es mit apt-get install -y selber installieren.

#### Fehlersuche
Wenn cloud-init nicht funktioniert merkt man das oft nicht direkt, die Instanz startet ganz normal aber die Einrichtung fehlt. Darum sollte man vorher die YAML Syntax auf https://www.yamllint.com/ prüfen und schauen ob #cloud-config in der ersten Zeile steht. Auf der Instanz selber findet man im Log was schief gelaufen ist:

```bash
cat /var/log/cloud-init-output.log
cloud-init status
```

Meistens liegt es an einem fehlenden #cloud-config, falscher Einrückung, einem SSH-Key mit Zeilenumbrüchen oder einem Befehl im runcmd der fehlschlägt.

#### Anwendung im Modul
Cloud-init brauchen wir von KN01 (lokal mit multipass launch --cloud-init datei.yaml) über KN02 (AWS user data) bis KN07, dort richtet der Autoscaler jede neue Instanz damit ein. Ab KN03 gibt OpenTofu die Datei als user_data mit. In KN02 macht man sich ein eigenes Template mit den eigenen Keys, dem Key der Lehrperson und der Härtung, das man danach in allen KNs wieder brauchen kann.

### YAML
#### Warum YAML?
YAML (YAML Ain't Markup Language) ist ein Textformat für strukturierte Daten, so ähnlich wie JSON aber viel besser lesbar. Fast alles in der Automatisierung wird mit YAML konfiguriert, z.B cloud-init, Ansible, Docker Compose, Kubernetes oder GitLab CI. Wenn man YAML lesen kann findet man sich also in all diesen Tools zurecht.

#### Grundelemente
Eigentlich gibt es nur drei Sachen in YAML. Key-Value Paare wie name: webserver-01, Listen wo jedes Element mit einem - anfängt und Verschachtelung durch Einrückung, normalerweise mit 2 Leerzeichen. Die Einrückung ist dabei nicht nur fürs Aussehen sondern bestimmt was wozu gehört.

```yaml
users:
  - name: ubuntu
    shell: /bin/bash
  - name: admin
    shell: /bin/zsh
```

Hier ist users eine Liste mit zwei Objekten, shell gehört jeweils zum gleichen Objekt wie name weil es gleich eingerückt ist. In JSON wäre das gleiche {"users": [{"name": "ubuntu", "shell": "/bin/bash"}, {"name": "admin", "shell": "/bin/zsh"}]}, YAML spart sich einfach die ganzen Klammern.
Kommentare macht man mit #, Text mit Sonderzeichen setzt man in Anführungszeichen und mit | kann man mehrzeiligen Text schreiben bei dem die Zeilenumbrüche bleiben, das braucht man z.B für Dateiinhalte oder Skripte.

#### Typische Fehler
Der häufigste Fehler ist die falsche Einrückung, schon ein Leerzeichen zu viel oder zu wenig kann die Bedeutung ändern oder die Datei kaputt machen. Tabs sind in YAML gar nicht erlaubt, nur Leerzeichen (VS Code macht das bei .yaml Dateien automatisch). Nach dem Doppelpunkt muss immer ein Leerzeichen kommen, port:80 geht nicht aber port: 80 schon. Werte mit Sonderzeichen wie : oder # sollte man in Anführungszeichen setzen. Ein spezieller Fall sind yes, no, on und off, die werden ohne Anführungszeichen je nach Parser als true oder false gelesen. country: NO für Norwegen wird so zu false, darum besser country: "NO" schreiben.
Am besten validiert man die Datei immer zuerst z.B auf https://www.yamllint.com/ bevor man sie braucht, so spart man sich bei cloud-init viel Zeit beim Fehler suchen.




## Vorgehen KNO1

### A)

Ich habe Multipass installiert und die version sowie den driver im Terminal angezeigt
![image](img\Screenshot_Multipass.png)


Danach habe ich nachgeschaut welche Edition von Windows ich habe.
![image](img\Screenshot_WinVersion.png)


Anschliessend habe ich eine kleine VM namens Smoke erstellt um Multipass zu testen.
![image](img\Screenshot_Launch_Smoke.png)


Mit diesem Befehl konnte ich die laufenden VMs anzeigen
![image](img\Screenshot_SmokeRunning.png)


Dann habe ich mir genauere Details über unsere Test-VM anzeigen lassen
![image](img\Screenshot_Exec.png)


Dann habe ich die Test VM wieder gelöscht
![image](img\Screenshot_delete.png)
