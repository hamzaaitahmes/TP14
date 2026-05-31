# TP14
LAB14-Bypass-Root-Frida/

 Étape 1 — Validation de l'injection Frida
Avant de développer des scripts complexes, il convient de s'assurer que l'infrastructure de base fonctionne correctement[cite: 2]. Un script minimal permet de valider la communication[cite: 2] :


frida -U -f owasp.mstg.uncrackable1 -l hello.js
```[cite: 2]

Si le serveur `frida-server` est actif, l'appareil ADB connecté et les versions alignées, le terminal affiche le message suivant[cite: 2] :

text
Connected to Android Emulator 5554
Spawned `owasp.mstg.uncrackable1`. Resuming main thread!
[+] Script injecté: Java.perform OK
```[cite: 2]

<img width="1232" height="406" alt="Test injection Frida avec hello.js" src="[https://github.com/user-attachments/assets/dbacaa53-12bb-4e63-9631-7b5a93fdbb51](https://github.com/user-attachments/assets/dbacaa53-12bb-4e63-9631-7b5a93fdbb51)" />[cite: 2]

Frida est alors correctement injecté dans le processus, permettant de passer aux étapes suivantes[cite: 2].

---

### Étape 2 — Comportement initial de l'application
Sans aucune modification, l'application identifie le root dès son ouverture et se ferme automatiquement après avoir affiché ce message d'erreur[cite: 2] :

> *"Root detected! This is unacceptable. The app is now going to exit."*[cite: 2]

<img width="398" height="821" alt="Root détecté - application bloquée" src="[https://github.com/user-attachments/assets/178acd47-8c49-4879-aac9-6ef72dbec20c](https://github.com/user-attachments/assets/178acd47-8c49-4879-aac9-6ef72dbec20c)" />[cite: 2]

L'objectif de ce laboratoire est de supprimer ce blocage et d'analyser les raisons de son apparition[cite: 2].

---

### Étape 3 — Premier niveau de contournement : les vérifications Java
La majorité des applications Android effectuent leurs contrôles de sécurité au niveau de la couche Java[cite: 2]. Elles vérifient généralement la valeur de `Build.TAGS`, recherchent la présence de fichiers spécifiques comme `/system/bin/su` ou `Superuser.apk`, ou exécutent des commandes via `Runtime.exec`[cite: 2]. Le script `bypass_root_basic.js` est conçu pour intercepter et neutraliser ces vérifications[cite: 2].

```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js
```[cite: 2]

Le terminal confirme la mise en place progressive de chaque hook[cite: 2] :

```text
[+] Build.TAGS -> release-keys
[*] RootBeer non présent
[+] Runtime.exec hooks installés
[+] Bypass Java installé
[+] File.exists bypass: /system/bin/su
[+] File.exists bypass: /system/xbin/su
[+] File.exists bypass: /system/app/Superuser.apk
```[cite: 2]

<img width="1368" height="466" alt="Bypass Java actif" src="[https://github.com/user-attachments/assets/27471f65-aad4-46ba-aff4-b6fe223a1ea7](https://github.com/user-attachments/assets/27471f65-aad4-46ba-aff4-b6fe223a1ea7)" />[cite: 2]

À ce stade, l'application s'exécute normalement, sans affichage de pop-up ni fermeture forcée[cite: 2].

<img width="403" height="828" alt="Application accessible après bypass Java" src="[https://github.com/user-attachments/assets/dedac173-bb0c-46d2-af2c-e5d6b3bb93c7](https://github.com/user-attachments/assets/dedac173-bb0c-46d2-af2c-e5d6b3bb93c7)" />[cite: 2]

---

### Étape 4 — Contournement avancé : implémentation des hooks natifs
Certaines applications complètent leurs vérifications en interrogeant directement le système via du code natif (C/C++) via le NDK, contournant ainsi les API Java[cite: 2]. Elles appellent des fonctions système telles que `open`, `stat`, `access` ou `readlink` pour inspecter le système de fichiers à bas niveau[cite: 2]. Les hooks Java sont inefficaces contre ces méthodes[cite: 2].

Pour y remédier, nous injectons simultanément un second script[cite: 2] :

```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js -l bypass_native.js
```[cite: 2]

Frida intercepte alors les fonctions directement au niveau du binaire[cite: 2] :

```text
[+] Hooked native open
[+] Hooked native openat
[+] Hooked native access
[+] Hooked native stat
[+] Hooked native lstat
[+] Hooked native fopen
[+] Hooked native readlink
[+] Native root bypass installed
```[cite: 2]

<img width="1476" height="686" alt="Bypass Java + natif combinés" src="[https://github.com/user-attachments/assets/fa906311-0596-4b7d-9069-a3127270fea8](https://github.com/user-attachments/assets/fa906311-0596-4b7d-9069-a3127270fea8)" />[cite: 2]

L'activation conjointe de ces deux scripts permet de couvrir à la fois la couche Java et la couche native, offrant une solution de contournement complète[cite: 2].

---

### Étape 5 — Alternative automatisée avec Objection
Afin de comparer notre méthode manuelle avec un outil automatisé, nous testons l'utilisation du framework Objection[cite: 2] :

```bash
objection -g owasp.mstg.uncrackable1 explore --startup-command "android root disable"
```[cite: 2]

Le terminal indique la prise en compte de la commande dès le démarrage[cite: 2] :

```text
Running a startup command... android root disable
(agent) Registering job. Name: root-detection-disable
owasp.mstg.uncrackable1 on Android: 11 [usb]
```[cite: 2]

<img width="1465" height="441" alt="Bypass via Objection" src="[https://github.com/user-attachments/assets/4e314f01-9b08-4a3a-ac19-57b9b535ea09](https://github.com/user-attachments/assets/4e314f01-9b08-4a3a-ac19-57b9b535ea09)" />[cite: 2]

Objection permet de traiter rapidement les cas standards sans rédaction de code JavaScript[cite: 2]. Cependant, l'utilisation de scripts personnalisés reste nécessaire face à des vérifications spécifiques ou modifiées[cite: 2].

---

### Étape 6 — Observation des appels système via frida-trace
Cette technique permet d'analyser les fonctions appelées par l'application en temps réel avant de concevoir un script de contournement[cite: 2] :

```bash
frida-trace -U -i open -i access -i stat -i openat -i fopen -i readlink Uncrackable1
```[cite: 2]

L'outil configure les points de contrôle pour les fonctions sélectionnées[cite: 2] :

```text
Instrumenting...
open: Auto-generated handler
access: Auto-generated handler
stat: Auto-generated handler
openat: Auto-generated handler
fopen: Auto-generated handler
readlink: Auto-generated handler
Started tracing 6 functions.
```[cite: 2]

<img width="1458" height="244" alt="Trace des appels natifs" src="[https://github.com/user-attachments/assets/25ce916a-1242-433d-acd8-1e36191ea2a5](https://github.com/user-attachments/assets/25ce916a-1242-433d-acd8-1e36191ea2a5)" />[cite: 2]

Ce suivi permet d'identifier précisément les fichiers, commandes et chemins ciblés par l'application, facilitant ainsi la création d'un script de bypass adapté[cite: 2].
