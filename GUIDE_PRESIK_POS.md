# Presik POS — le guide du poste de caisse

**Version du paquet : 6.0.75+1 — build local**
**Écrit par Guivens Chery**

---

Ce guide couvre le **poste de caisse** : l'installer, le brancher sur ton
serveur, le configurer, et s'en sortir quand ça coince.

Il ne dit pas comment monter un serveur Tryton. Chaque installation a le sien,
avec ses propres choix, et ce n'est pas le rôle de ce paquet. Ce qu'il te faut
côté serveur tient au chapitre 3, en une page.

---

## Sommaire

1. [Ce que c'est, et ce que ce n'est pas](#1-ce-que-cest-et-ce-que-ce-nest-pas)
2. [Ce dont tu as besoin](#2-ce-dont-tu-as-besoin)
3. [Ce que le serveur doit fournir](#3-ce-que-le-serveur-doit-fournir)
4. [Installer le poste](#4-installer-le-poste)
5. [Configurer la caisse](#5-configurer-la-caisse)
6. [Le premier démarrage](#6-le-premier-démarrage)
7. [Quand ça casse](#7-quand-ça-casse)
8. [Mettre à jour un poste](#8-mettre-à-jour-un-poste)
9. [Reconstruire le paquet](#9-reconstruire-le-paquet)
10. [Ce que ce build corrige](#10-ce-que-ce-build-corrige)
11. [Les limites que je connais](#11-les-limites-que-je-connais)
12. [Désinstaller](#12-désinstaller)

---

## 1. Ce que c'est, et ce que ce n'est pas

Presik POS est un logiciel de caisse qui parle à un serveur Tryton. Le paquet
que tu as entre les mains est un **build local**, reconstruit à partir des
sources.

Autant le dire dès la première page :

> Ce paquet **n'est pas distribué par Presik SAS**. Je l'ai reconstruit avec des
> correctifs qui n'existent pas en amont, pour qu'il fonctionne contre un
> serveur Tryton 8.0. Si tu appelles Presik pour du support, ils ne
> reconnaîtront pas cette version.

En trois lignes : le client ne savait pas lire le jeton de session de Tryton 8,
vingt-trois fichiers plantaient à cause d'un import Python oublié, et
l'interface était uniquement en espagnol. Le détail est au chapitre 10.

Le paquet contient **le client, et rien d'autre** : aucune adresse de serveur,
aucun nom de base, aucun compte. Tu renseignes tout ça au premier démarrage,
pour ton installation à toi.

---

## 2. Ce dont tu as besoin

- Debian, Ubuntu, Parrot ou n'importe quel dérivé avec `apt`
- Architecture **amd64** — `dpkg --print-architecture` te le confirme
- **glibc 2.31 minimum**, donc Debian 11+ ou Ubuntu 20.04+ — `ldd --version`
  te donne le numéro. En dessous, le paquet s'installe mais l'application ne
  démarre pas.
- Un bureau graphique, X11 ou Wayland
- Environ 700 Mo de libre sur `/opt`

Pas besoin d'installer Python ni PySide6 : ils sont dans le paquet. C'est pour
ça qu'il pèse 162 Mo.

Et bien sûr, un serveur joignable depuis ce poste — voir le chapitre suivant.

---

## 3. Ce que le serveur doit fournir

C'est le point que personne ne devine, donc je m'y arrête un instant.

Presik POS n'interroge pas un Tryton ordinaire. Il attend des modèles qui
n'existent pas dans une installation standard :

```
sale.shop            sale.device           sale.discount
sale.source          sale.delivery_party   sale.shop.table
shop.table.zone      party.consumer        sale.product.mix
sale.payment         sale_pos.expenses_daily
```

plus une trentaine de champs ajoutés à `sale.configuration`, une vingtaine à
`sale.sale`, et d'autres sur `res.user`, `account.statement.journal`,
`party.party` et `company.employee`.

Ces modèles viennent des **modules de vente de Presik**, installés sur le
serveur. Trois conditions, donc :

1. Le serveur tourne en **Tryton 8.0** — pas 7.x, pas 6.x. Les versions
   antérieures n'ont pas la même méthode de connexion.
2. Les modules de vente Presik sont installés et activés sur la base.
3. Un magasin, un terminal et un utilisateur de caisse existent, et
   l'utilisateur est bien rattaché aux deux premiers.

Si tu n'administres pas le serveur, c'est la liste à transmettre à celui qui
s'en occupe.

**Comment tu sauras que quelque chose manque :** la connexion réussit — tu vois
même l'écran de login passer, ce qui est trompeur — puis l'application s'arrête
au chargement de la configuration du terminal. Le chapitre 7 traduit les
messages d'erreur.

---

## 4. Installer le poste

Une commande.

```bash
sudo apt install ./presik-pos_6.0.75+1_amd64.deb
```

`apt` récupère tout seul les bibliothèques système manquantes
(`libxcb-cursor0`, `libegl1`, et les autres). Le `./` n'est pas décoratif :
sans lui, `apt` cherche un paquet nommé « presik-pos » dans les dépôts et ne le
trouve pas.

Compte une à deux minutes, il y a 162 Mo à décompresser.

**Vérifie que c'est propre :**

```bash
dpkg -l presik-pos
```

Tu dois lire `ii` au début de la ligne : installé et configuré. Si tu vois
autre chose — `iU`, `iHR`, `iF` — va directement au chapitre 7.

### Retrouver le serveur par son nom

Si l'adresse IP du serveur bouge (box qui redémarre, DHCP), tu peux l'appeler
par son nom plutôt que par son numéro :

```bash
sudo apt install -y avahi-daemon libnss-mdns
sudo systemctl enable --now avahi-daemon
```

À faire sur le poste **et** sur le serveur. Celui-ci répond alors à
`nom-de-la-machine.local`, quelle que soit son adresse.

> Ça repose sur le multicast, que beaucoup de réseaux Wi-Fi invités bloquent.
> Si `ping monserveur.local` reste muet, ne cherche pas : mets l'IP en dur.

---

## 5. Configurer la caisse

Au premier lancement, un fichier de configuration est créé dans ton dossier
personnel, à partir du modèle livré avec le paquet :

```
~/.config/presik_pos/config_pos.ini
```

**La copie ne se fait que s'il n'existe pas déjà.** Mettre à jour le paquet
n'écrasera donc jamais ta configuration.

Le modèle est livré **vide de toute adresse** : c'est à toi de le remplir.

```ini
[General]
server=                      # IP ou nom du serveur, ex. 192.168.1.20 ou serveur.local
port=8000
mode=http                    # http ou https
database=                    # le nom de la base Tryton
user=                        # utilisateur pré-rempli à l'écran de connexion (facultatif)

printer_sale_name=usb,/dev/usb/lp0
profile_printer=TM-P80
row_characters=48            # largeur du ticket en caractères
print_receipt=automatic      # automatic ou manually

enviroment=restaurant        # restaurant (tables) ou retail (comptoir)
theme=base                   # base = clair, dark = sombre
mode_window=maximized        # maximized ou fullscreen
```

Tu n'es pas obligé d'ouvrir le fichier : à l'écran de connexion, le menu
**Opciones → Configuracion** l'édite, et **Opciones → Servidor / Serveur**
change l'adresse du serveur en deux clics.

### L'imprimante

Trois écritures, selon le branchement :

```ini
printer_sale_name=usb,/dev/usb/lp0           # USB direct
printer_sale_name=network,192.168.0.36:9100  # imprimante réseau
printer_sale_name=cups,NOM-DE-LIMPRIMANTE    # via CUPS
```

En USB, si rien ne sort, c'est presque toujours une histoire de droits. Ajoute
l'utilisateur au groupe `lp`, puis **reconnecte la session** :

```bash
sudo usermod -aG lp $USER
```

---

## 6. Le premier démarrage

```bash
presik-pos
```

Ou par l'icône **Presik POS** dans le menu des applications.

**La langue.** Au tout premier lancement, l'application demande Français ou
Español. Le choix est retenu dans `~/.config/presik_pos/language.conf`. Pour en
changer : menu **Opciones → Idioma / Langue**, avec redémarrage proposé.

**La connexion.**

| Champ | Ce qu'il attend |
|---|---|
| Entreprise | le **nom de la base** Tryton |
| Utilisateur | un utilisateur de caisse existant sur le serveur |
| Mot de passe | le sien |

C'est le piège classique : « Entreprise » attend le nom de la base de données,
pas le nom commercial de la société.

Si l'écran de vente s'ouvre avec les boutons de catégories : tout est en place.
Fais une vente d'essai avant de laisser la caisse à quelqu'un.

---

## 7. Quand ça casse

### L'installation s'est interrompue

Coupure de courant, PC éteint en pleine installation, `Ctrl+C` malheureux.
`dpkg -l presik-pos` affiche `iHR` ou `iU` au lieu de `ii`, et
`/opt/presik_pos` est plein de fichiers `.dpkg-new` et `.dpkg-tmp`.

Dans cet état, l'application est un mélange d'ancien et de neuf. **Ne la lance
pas**, répare :

```bash
sudo dpkg --configure -a
sudo apt install -y --reinstall ./presik-pos_6.0.75+1_amd64.deb
```

`dpkg --configure -a` seul ne suffit pas quand le dépaquetage lui-même a été
coupé — il faut bien la réinstallation derrière. Vérifie ensuite :

```bash
dpkg -l presik-pos                    # doit afficher ii
find /opt/presik_pos -name '*.dpkg-*' # ne doit rien afficher
```

### Les messages d'erreur, et ce qu'ils veulent vraiment dire

Lance la caisse depuis un terminal pour voir les erreurs :

```bash
presik-pos
```

| Ce que tu lis | Ce que c'est vraiment |
|---|---|
| `TypeError: string indices must be integers` | Un appel au serveur a échoué : le code a reçu la chaîne `'error_network'` là où il attendait une liste. Regarde le réseau et le service Tryton. |
| `JSONDecodeError: zero-length document` | Le serveur a répondu `401` avec un corps vide : le jeton de session n'est pas passé. Souvent un mot de passe faux, ou un serveur qui n'est pas en Tryton 8.0. |
| `JSONDecodeError: unexpected character` | Le serveur a renvoyé une page d'erreur HTML au lieu de données : il manque un champ ou un modèle côté serveur (chapitre 3). |
| `ValueError: too many values to unpack` | `sale.configuration` est incomplet sur le serveur. |
| L'application se ferme juste après la connexion | `_load_device` : le magasin, le terminal ou le rattachement de l'utilisateur manque côté serveur. |

### Les commandes qui aident

```bash
ping monserveur.local                # la machine répond-elle
nc -vz <serveur> 8000                # le port est-il joignable d'ici
avahi-resolve -n monserveur.local    # le nom se résout-il
```

Dans cet ordre, du plus proche au plus lointain. Ça évite de chercher un
problème de configuration alors que le câble est débranché.

---

## 8. Mettre à jour un poste

La méthode normale :

```bash
sudo apt install ./presik-pos_<nouvelle-version>_amd64.deb
```

Ta configuration est conservée.

**Le correctif d'urgence**, quand il faut changer une ligne de code sur une
caisse en production sans reconstruire les 162 Mo. L'application est gelée en
bytecode : il faut compiler le fichier et le poser à la main.

```bash
python3 -c "import py_compile; py_compile.compile('app/ui/payment_panel.py', \
  cfile='/tmp/payment_panel.pyc', dfile='app/ui/payment_panel.py', doraise=True)"

sudo install -m 644 /tmp/payment_panel.pyc \
  /opt/presik_pos/lib/app/ui/payment_panel.pyc
```

> Le Python qui compile doit être de la **même version** que celui embarqué
> (3.13). Un bytecode compilé par un autre Python a un numéro magique
> différent, et l'application refusera de démarrer.

C'est du dépannage, pas une méthode : au prochain `apt install`, ton correctif
disparaît. Pense à le reporter dans les sources.

---

## 9. Reconstruire le paquet

Depuis la racine des sources :

```bash
./build_linux.sh
```

Le script crée l'environnement virtuel, installe les dépendances, gèle
l'application avec cx_Freeze, assemble l'arborescence Debian et produit
`dist/presik-pos_<version>_amd64.deb`.

Compte une dizaine de minutes. L'étape lente est la compression xz des 614 Mo
de fichiers gelés : c'est normal, laisse tourner.

Le numéro de version vient du fichier `VERSION`, et de nulle part ailleurs.

> **N'exécute jamais `lupdate` sur ce projet sans avoir sauvegardé
> `locale/*.ts` avant.** Son analyseur Python ne reconnaît pas toutes les
> formes d'appel à `self.tr()` ; il juge les traductions correspondantes
> obsolètes et les supprime. J'ai reperdu des dizaines de chaînes comme ça.

---

## 10. Ce que ce build corrige

Pour ceux qui reprendront ce travail. Chaque ligne est un problème qui m'a
coûté du temps.

**L'authentification Tryton 8.** C'est le correctif central. Le client appelait
`/<base>/fast_login` et lisait le jeton de session dans le corps de la réponse.
Tryton 8 a changé : l'appel est `/<base>/session/login` avec la méthode
`common.db.login`, le corps ne contient que l'identifiant de l'utilisateur, et
le **jeton part dans un cookie** `tryton_session=login:user_id:token`. Sans ce
correctif, chaque appel suivant repart sans en-tête `Authorization` et reçoit un
`401` au corps vide — d'où le `JSONDecodeError` du chapitre 7.

**L'import Qt manquant.** 23 fichiers appelaient `QCoreApplication.translate()`
sans jamais importer `QCoreApplication`. Chacun plantait à l'ouverture de son
écran.

**La traduction française.** 389 chaînes : boutons, messages d'erreur, colonnes
des tableaux, lignes de ticket, dialogues.

**Le choix de la langue au démarrage**, mémorisé dans
`~/.config/presik_pos/language.conf`, avec une entrée de menu pour en changer.

**Le réglage du serveur depuis l'écran de connexion**, pour ne pas avoir à
éditer le fichier INI sur chaque poste.

**L'alias de thème** : `theme=base` désigne explicitement le thème clair. Avant,
une valeur inconnue donnait un écran à moitié noir.

Tout cela concerne le client. Ce que le serveur doit apporter est au chapitre 3.

---

## 11. Les limites que je connais

Je préfère les écrire que les laisser découvrir.

**Le client dépend entièrement des modules de vente de Presik.** Ce paquet ne
les contient pas et ne peut pas les remplacer. Si la base Tryton ne les a pas,
la caisse ne dépassera pas l'écran de connexion.

**`regime_tax` et `authorizations` sortent vides sur les tickets.** Ce sont des
champs d'obligation fiscale colombienne, sans équivalent ailleurs. Je les ai
laissés vides plutôt que d'inventer.

**`electronic.payment` et `hotel.folio` n'existent pas.** Ils appartiennent aux
modules `payment_openpay` et `hotel`, absents de cette installation. Le client
ne les interroge que si ces modules sont déclarés actifs — ne les active pas.

**La langue ne se change qu'à l'écran de connexion.** L'écran de vente n'a pas
de barre de menu, il n'y a nulle part où mettre l'option. Déconnecte-toi pour
changer.

**Le paquet est amd64 uniquement.** Pas d'ARM, donc pas de Raspberry Pi.

---

## 12. Désinstaller

Retirer l'application, garder la configuration :

```bash
sudo apt remove presik-pos
```

Tout retirer, configuration comprise :

```bash
sudo apt remove presik-pos
rm -rf ~/.config/presik_pos
```

---

*Guivens Chery — Presik POS 6.0.75+1*
