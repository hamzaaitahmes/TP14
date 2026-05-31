LAB-14 — Bypass de détection Root sur Android avec Frida, Objection et Hooks Natifs

Objectif
Ce lab adopte une approche méthodique et multicouche du bypass de détection root sur Android. On ne se limite pas à une seule technique — on combine hooks Java, hooks natifs en C, l'outil Objection, et la trace d'appels système avec frida-trace pour comprendre ce que l'application fait réellement sous le capot.
Cible : OWASP UnCrackable Level 1

Environnement
ÉlémentDétailSystèmeWindows PowerShellÉmulateurAndroid Emulator 5554Frida17.8.0Objection1.12.4Android11Package cibleowasp.mstg.uncrackable1

Structure du projet
LAB14-Bypass-Root-Frida/
│
├── README.md
├── hello.js               # Vérifie que l'injection Frida fonctionne
├── bypass_root_basic.js   # Neutralise les vérifications Java
└── bypass_native.js       # Hooke les fonctions système en C

Étape 1 — Vérifier l'injection Frida
Avant toute chose, on s'assure que Frida peut s'injecter dans le processus cible avec un script minimal.
bashfrida -U -f owasp.mstg.uncrackable1 -l hello.js
Résultat attendu :
Connected to Android Emulator 5554
Spawned `owasp.mstg.uncrackable1`. Resuming main thread!
[+] Script injecté: Java.perform OK
Si frida-server tourne, qu'ADB est connecté et que les versions sont compatibles, l'injection se passe sans accroc.

Étape 2 — Comportement sans intervention
Sans bypass, l'application se ferme immédiatement au lancement avec le message suivant :
Root detected!
This is unacceptable. The app is now going to exit.
C'est le point de départ. L'objectif est de faire disparaître ce blocage — et de comprendre précisément d'où il vient.

Étape 3 — Bypass Java avec bypass_root_basic.js
La majorité des applications Android détectent le root en Java : elles vérifient Build.TAGS, cherchent des fichiers comme /system/bin/su ou Superuser.apk, ou exécutent des commandes via Runtime.exec. Ce script neutralise toutes ces vérifications.
bashfrida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js
Sortie du script :
[+] Build.TAGS -> release-keys
[*] RootBeer non présent
[+] Runtime.exec hooks installés
[+] Bypass Java installé
[+] File.exists bypass: /system/bin/su
[+] File.exists bypass: /system/xbin/su
[+] File.exists bypass: /system/app/Superuser.apk
L'application se lance normalement — plus de popup, plus de fermeture forcée.

Étape 4 — Bypass natif avec bypass_native.js
Certaines applications contournent les APIs Java et effectuent leurs vérifications directement via du code natif (C/C++ via le NDK), en appelant des fonctions système comme open, stat, access ou readlink. Les hooks Java n'ont aucun effet sur cette couche.
On ajoute le second script en parallèle :
bashfrida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js -l bypass_native.js
Sortie :
[+] Hooked native open
[+] Hooked native openat
[+] Hooked native access
[+] Hooked native stat
[+] Hooked native lstat
[+] Hooked native fopen
[+] Hooked native readlink
[+] Native root bypass installed
Avec les deux scripts actifs simultanément, on couvre l'intégralité des couches de détection — Java et native. C'est le bypass le plus complet réalisable avec cette approche.

Étape 5 — Alternative rapide avec Objection
Pour les cas standards, Objection permet de faire la même chose en une seule commande :
bashobjection -g owasp.mstg.uncrackable1 explore --startup-command "android root disable"
Sortie :
Running a startup command... android root disable
(agent) Registering job. Name: root-detection-disable
owasp.mstg.uncrackable1 on Android: 11 [usb]
Objection est idéal pour tester rapidement. En revanche, dès qu'une application implémente des vérifications personnalisées, les scripts Frida manuels redeviennent indispensables.

Étape 6 — Observer les appels système avec frida-trace
Avant d'écrire un bypass, il est souvent plus efficace d'observer ce que l'application appelle réellement en temps réel. frida-trace génère automatiquement des handlers pour chaque fonction tracée.
bashfrida-trace -U -i open -i access -i stat -i openat -i fopen -i readlink Uncrackable1
Sortie :
Instrumenting...
open: Auto-generated handler
access: Auto-generated handler
stat: Auto-generated handler
openat: Auto-generated handler
fopen: Auto-generated handler
readlink: Auto-generated handler
Started tracing 6 functions.
Cette technique permet de voir exactement quels fichiers l'application tente d'ouvrir, quels chemins elle inspecte et quelles commandes elle teste. À partir de là, écrire un bypass ciblé devient beaucoup plus direct.

Récapitulatif des techniques
TechniqueCommandeUsageBypass Javafrida -l bypass_root_basic.jsNeutralise les checks Java standardsBypass natiffrida -l bypass_native.jsIntercepte les appels système en CBypass combinéLes deux scripts en même tempsCouverture maximaleObjectionandroid root disableBypass rapide pour cas standardsTracefrida-trace -i open -i stat ...Observation des appels avant bypass

Prérequis

frida-server démarré sur l'émulateur Android
ADB connecté et fonctionnel
Versions Frida client/serveur alignées
OWASP UnCrackable Level 1 installé sur l'émulateur
