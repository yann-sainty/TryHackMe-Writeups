# Write-up : TryHackMe - RootMe

**Date :** 02/01/2026
**Auteur :** SAINTY Yann
**Catégorie :** Web / Linux
**Difficulté :** Facile
**Lien :** [TryHackMe | RootMe](https://tryhackme.com/room/rrootme)

---

## 1. Introduction & Scénario

L'objectif de ce CTF (Capture The Flag) est d'obtenir un accès root sur la machine "RootMe".
D'un point de vue technique, ce challenge consiste à énumérer un serveur web Apache, exploiter une vulnérabilité d'upload de fichier pour obtenir un accès initial (Reverse Shell), et effectuer une escalade de privilèges sur Linux via une mauvaise configuration de permission (SUID).

---

## 2. Reconnaissance & Énumération

### Scan Nmap

La première étape consiste à scanner l'adresse IP cible pour identifier les services actifs. J'utilise les options `-sC` (scripts par défaut) et `-sV` (versions des services).

```bash
nmap -sC -sV 10.80.152.135

```

Résultats du scan:

* **Port 22 (SSH) :** OpenSSH 8.2p1.
* **Port 80 (HTTP) :** Serveur Apache 2.4.41 (Ubuntu).

Le port 80 étant ouvert, la surface d'attaque principale semble être l'application web.

### Énumération Web

Je lance une énumération des répertoires pour cartographier l'arborescence du site. Pour cela, j'utilise l'outil **Gobuster** avec une wordlist standard nommée `common.txt` (fournie en annexe).

```bash
gobuster dir -u http://10.80.152.135 -w common.txt

```

**Découvertes intéressantes :**

* `/panel/` : Une page contenant un formulaire d'upload de fichiers.
* `/uploads/` : Un répertoire où les fichiers uploadés semblent être stockés.

---

## 3. Accès Initial

Je me rends sur la page `/panel/`. Le formulaire permet de téléverser des fichiers, ce qui est un vecteur d'attaque idéal pour un **Reverse Shell**.

### Exploitation (File Upload Bypass)

J'utilise le script PHP Reverse Shell de pentestmonkey. Cependant, en tentant d'uploader le fichier `.php`, le serveur rejette l'envoi. Il y a un filtrage sur l'extension.

Pour contourner cette restriction, je renomme mon script malveillant avec une extension alternative interprétée par Apache : **`.phtml`**.
Le fichier utilisé pour l'attaque est nommé `shell.phtml` (fourni en annexe).

Mise en place de l'attaque:

1. Je configure un écouteur (listener) sur ma machine attaquante :
```bash
ncat -lvnp 1234

```


2. J'uploade le fichier `shell.phtml` via le panel. Succès !
3. J'accède au fichier via l'URL : `http://10.80.152.135/uploads/shell.phtml`.

La connexion est établie et j'obtiens un shell en tant qu'utilisateur `www-data`.

---

## 4. Récupération du User Flag

Une fois sur le système, je recherche le premier drapeau. En explorant les dossiers standards d'un serveur web, je le localise rapidement.

**Accès :**

```bash
cat /var/www/user.txt

```

**Flag 1 :** *[User Flag Validé]*

---

## 5. Escalade de Privilèges (PrivEsc)

Pour obtenir le dernier flag, il faut des droits `root`. Je recherche les fichiers possédant la permission **SUID** (Set User ID), qui permettent d'exécuter un programme avec les droits de son propriétaire (ici root).

```bash
find / -user root -perm /4000 2>/dev/null

```

Résultat:
La commande retourne `/usr/bin/python`. C'est une faille critique : nous pouvons utiliser Python pour invoquer un shell système avec les droits maintenus.

### Exploitation SUID

J'exécute la commande suivante pour devenir root :

```bash
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

```

Je vérifie mon identité avec `whoami` : le système répond **root**.
Il ne reste plus qu'à lire le dernier fichier.

```bash
cat /root/root.txt

```

**Flag 2 :** *[Root Flag Validé]*

---

## Conclusion

Ce CTF permet de valider des concepts fondamentaux :

1. **Reconnaissance Web :** L'utilisation de wordlists (`common.txt`) pour trouver des panels cachés.
2. **Bypass d'Upload :** Savoir jongler avec les extensions (`.php` vs `.phtml`) pour contourner les filtres basiques.
3. **Privilege Escalation :** L'exploitation des binaires SUID mal configurés (Python) est un classique des environnements Linux.

### Annexes

Les fichiers suivants sont joints à ce rapport :

* `common.txt` : Liste utilisée pour le brute-force de répertoires.
* `shell.phtml` : Le payload utilisé pour obtenir l'accès initial.
