# Presik POS, guide du build et de l'utilisation

**Version du paquet : 6.0.75+4 (build local)**
**Écrit par Guivens Chery**

Ce guide est fait de procédures. Chaque étape te donne la commande à taper, puis
ce que tu dois voir à l'écran. Si ce que tu vois ne correspond pas, arrête-toi
là. La suite ne marchera pas, et tu perdras moins de temps à comprendre
maintenant qu'à défaire plus tard.

| Procédure | Pour qui | Durée |
|---|---|---|
| [A. Construire le paquet](#a-construire-le-paquet) | celui qui fabrique | 30 min, dont 15 d'attente |
| [B. Installer un poste](#b-installer-un-poste) | celui qui déploie | 5 min |
| [C. Configurer et démarrer](#c-configurer-et-démarrer) | celui qui déploie | 5 min |
| [D. Brancher l'imprimante](#d-brancher-limprimante) | celui qui déploie | 5 min |
| [E. Mettre à jour un poste](#e-mettre-à-jour-un-poste) | maintenance | 3 min |
| [F. Réparer une installation coupée](#f-réparer-une-installation-coupée) | maintenance | 5 min |
| [G. Désinstaller](#g-désinstaller) | maintenance | 1 min |

Ensuite viennent quatre annexes : les [messages d'erreur](#annexe-1-les-messages-derreur),
[ce que corrige ce build](#annexe-2-ce-que-corrige-ce-build),
[ce qu'il y a dans le paquet](#annexe-3-ce-quil-y-a-dans-le-paquet),
et les [limites connues](#annexe-4-les-limites-connues).

> Ce paquet n'est pas distribué par Presik SAS. C'est un build local, avec des
> correctifs qui n'existent pas en amont, pour qu'il fonctionne contre un
> serveur Tryton 7.0 ou 8.0.

# A. Construire le paquet

Cette procédure part des sources et se termine par un fichier
`presik-pos_6.0.75+4_amd64.deb` prêt à être installé.

Il te faut une machine Debian ou Ubuntu en amd64, Python 3.12 ou 3.13, les
commandes `dpkg-deb` et `git`, et une connexion internet.

Deux chemins possibles. Le **chemin court** utilise le script `build_linux.sh`
qui accompagne les sources, et tient en une commande. Le **chemin long** fait la
même chose à la main, étape par étape. Prends le chemin long la première fois :
tu comprendras ce qu'il y a dans le paquet, et tu sauras réparer le jour où le
script tombe en panne.

## Étape 0. Installer les outils de construction

Sur une machine fraîchement installée, rien de tout cela n'est présent. Fais-le
en premier, sinon tu buteras à l'étape 4 sans comprendre pourquoi.

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-dev \
    build-essential dpkg-dev libcups2-dev libusb-1.0-0-dev
```

À quoi sert chaque chose : `python3-venv` crée l'environnement isolé,
`python3-dev` et `build-essential` fournissent le compilateur et les en-têtes
sans lesquels `pyusb` et `pycups` ne se construisent pas, `dpkg-dev` apporte la
commande `dpkg-deb` qui fabrique le paquet, et `libcups2-dev` permet
l'impression par CUPS.

Vérifie que les deux commandes essentielles répondent :

```bash
dpkg-deb --version | head -1
git --version
```

Tu dois voir un numéro de version pour chacune.

## Étape 1. Récupérer les sources et vérifier Python

```bash
cd ~
git clone <url-du-dépôt-des-sources> presik_pos
cd presik_pos
python3 --version
```

Tu dois voir `Python 3.12.x` ou `Python 3.13.x`. Avec une autre version, le gel
échouera plus loin.

## Étape 2. Fixer le numéro de version

```bash
echo "6.0.75+4" > VERSION
cat VERSION
```

Tu dois voir `6.0.75+4`.

Ce fichier est la seule source du numéro. Il se retrouve dans le nom du `.deb` et
dans les métadonnées du paquet.

Un conseil : évite de mettre des lettres après le `+`. Pour dpkg, les lettres se
classent avant les chiffres, donc `6.0.75+abc` passe pour plus récent que
`6.0.75+4`, et tes mises à jour partent à l'envers. Tu peux vérifier :

```bash
dpkg --compare-versions "6.0.75+5" gt "6.0.75+4" && echo "ordre correct"
```

## Étape 3. Vider la configuration d'exemple

```bash
grep -E "^server=|^database=|^user=" config_pos.ini
```

Tu dois voir trois lignes vides à droite du signe égal :

```
server=
database=
user=
```

Si une adresse ou un nom de base traîne là, efface-les maintenant. Ce fichier est
recopié tel quel chez chaque utilisateur au premier démarrage. Ce qui y reste
part avec le paquet, chez tout le monde.

## Étape 4. Créer l'environnement Python et installer les dépendances

```bash
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m pip install "cx-Freeze>=8.4"
```

Tu dois voir une ligne `Successfully installed ...` à la fin de chaque
installation.

Si `pycups` refuse de se construire, c'est qu'il manque les en-têtes de CUPS.
Deux solutions : les installer avec `sudo apt install libcups2-dev`, ou retirer
la ligne `pycups` du fichier `requirements.txt`. L'impression par CUPS ne sera
alors plus disponible, mais l'USB et le réseau fonctionnent toujours.

## Étape 5. Geler l'application

C'est ici que Python et Qt entrent dans le paquet.

```bash
.venv/bin/python setup_freeze.py build_exe --build-exe build/exe
```

Compte cinq à huit minutes. Tu vois défiler des lignes `copying ...` puis des
`patchelf --set-rpath ...`.

Vérifie le résultat :

```bash
ls build/exe/presik_pos
du -sh build/exe
```

Tu dois voir le binaire `build/exe/presik_pos`, et une taille autour de 614 Mo.

## Étape 6. Construire l'arborescence du paquet

Un paquet Debian n'est rien d'autre qu'un dossier qui reproduit l'arborescence
d'installation, plus un dossier `DEBIAN` contenant les métadonnées.

> Les étapes 6 à 11 utilisent deux variables, `VERSION` et `DEB`. Elles ne
> vivent que dans le terminal où tu les as tapées. Si tu fermes ce terminal, si
> tu changes d'onglet ou si tu reviens le lendemain, retape ces deux lignes
> avant de continuer, sinon les commandes travailleront sur des chemins vides
> sans rien dire.

```bash
cd ~/presik_pos
VERSION="6.0.75+4"
DEB="dist/presik-pos_${VERSION}_amd64"

mkdir -p "$DEB/opt/presik_pos" \
         "$DEB/usr/bin" \
         "$DEB/usr/share/applications" \
         "$DEB/usr/share/icons/hicolor/256x256/apps" \
         "$DEB/DEBIAN"

cp -a build/exe/. "$DEB/opt/presik_pos/"
cp app/resources/share/icon.png "$DEB/usr/share/icons/hicolor/256x256/apps/presik-pos.png"
```

Vérifie :

```bash
ls "$DEB/opt/presik_pos/presik_pos"
```

Tu dois voir le chemin s'afficher, sans erreur.

## Étape 7. Écrire le lanceur

C'est le petit script qui sera installé dans `/usr/bin/presik-pos`. Il prépare la
configuration de l'utilisateur, puis démarre l'application.

```bash
cat > "$DEB/usr/bin/presik-pos" <<'EOF'
#!/bin/sh
set -eu

CONF_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/presik_pos"
CONF="$CONF_DIR/config_pos.ini"
GABARIT="/opt/presik_pos/config_pos.ini"

mkdir -p "$CONF_DIR"
[ -f "$CONF" ] || cp "$GABARIT" "$CONF"
chmod 0644 "$CONF"

cd /opt/presik_pos
exec /opt/presik_pos/presik_pos -o "$CONF" "$@"
EOF

chmod 0755 "$DEB/usr/bin/presik-pos"
```

La ligne qui compte est `[ -f "$CONF" ] || cp "$GABARIT" "$CONF"`. Elle ne copie
le modèle que si la configuration de l'utilisateur n'existe pas encore. C'est ce
qui garantit qu'une mise à jour du paquet n'écrase jamais les réglages d'un
poste.

## Étape 8. Écrire l'entrée du menu des applications

```bash
cat > "$DEB/usr/share/applications/presik-pos.desktop" <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Presik POS
GenericName=Point de vente
Comment=Client point de vente pour Tryton
Exec=presik-pos
TryExec=/usr/bin/presik-pos
Icon=presik-pos
Terminal=false
Categories=Office;Finance;
Keywords=POS;caisse;vente;Tryton;
EOF

chmod 0644 "$DEB/usr/share/applications/presik-pos.desktop"
```

## Étape 9. Écrire le fichier de contrôle

C'est la carte d'identité du paquet : son nom, sa version, son mainteneur, et ce
dont il a besoin pour fonctionner.

```bash
cat > "$DEB/DEBIAN/control" <<EOF
Package: presik-pos
Version: $VERSION
Section: office
Priority: optional
Architecture: amd64
Depends: libc6 (>= 2.31), libxkbcommon0, libegl1, libglib2.0-0, libdbus-1-3, libfontconfig1, libfreetype6, libxcb-cursor0
Recommends: cups, libusb-1.0-0, avahi-daemon, libnss-mdns
Maintainer: Prénom Nom <adresse@exemple.org>
Homepage: https://www.presik.com
Description: Client point de vente Presik POS (build local)
 Build local de Presik POS 6.0.75, adapté à un serveur Tryton 7.0 ou 8.0.
 Il n'est pas distribué par Presik SAS et contient des correctifs locaux.
 .
 Le paquet embarque Python, PySide6 et toutes les dépendances applicatives.
EOF
```

Trois lignes méritent ton attention.

`Depends` liste les bibliothèques système que `apt` installera automatiquement.
`libc6 (>= 2.31)` est la plus importante : c'est elle qui empêche l'installation
sur une machine trop ancienne pour l'application.

`Maintainer` est le nom qui s'affichera chez tous ceux qui installeront le
paquet. Ne laisse jamais un gabarit, ni le nom d'une organisation qui n'a pas
donné son accord.

La `Description` commence par une ligne courte, puis des lignes indentées d'un
espace. Une ligne contenant seulement un point marque un paragraphe.

## Étape 10. Écrire les scripts d'installation

Ils rafraîchissent le menu des applications et le cache des icônes, après la
pose et après le retrait.

```bash
cat > "$DEB/DEBIAN/postinst" <<'EOF'
#!/bin/sh
set -e
command -v update-desktop-database >/dev/null 2>&1 &&
    update-desktop-database -q /usr/share/applications || true
command -v gtk-update-icon-cache >/dev/null 2>&1 &&
    gtk-update-icon-cache -q /usr/share/icons/hicolor || true
exit 0
EOF

cat > "$DEB/DEBIAN/postrm" <<'EOF'
#!/bin/sh
set -e
command -v update-desktop-database >/dev/null 2>&1 &&
    update-desktop-database -q /usr/share/applications || true
exit 0
EOF

chmod 0755 "$DEB/DEBIAN/postinst" "$DEB/DEBIAN/postrm"
```

Ces deux scripts tournent en root chez l'utilisateur. Le `|| true` à la fin de
chaque ligne est là pour qu'une commande absente ne fasse jamais échouer une
installation.

## Étape 11. Fabriquer le fichier .deb

```bash
dpkg-deb --build --root-owner-group "$DEB" "dist/presik-pos_${VERSION}_amd64.deb"
```

Tu dois voir :

```
dpkg-deb: construction du paquet « presik-pos » dans « dist/presik-pos_6.0.75+4_amd64.deb ».
```

**La commande rend la main au bout de trois à cinq minutes**, sans rien afficher
entre-temps. C'est la compression des 614 Mo qui prend ce temps. Elle n'est pas
bloquée. Pour t'en assurer, depuis un autre terminal :

```bash
watch -n 5 ls -lh dist/*.deb
```

Tant que le fichier grossit, tout va bien. Il finit autour de 162 Mo.

L'option `--root-owner-group` est importante : sans elle, les fichiers du paquet
appartiendraient à ton compte, et l'installation poserait des fichiers avec un
propriétaire qui n'existe pas sur la machine de destination.

## Le chemin court

Une fois que tu as compris ce que font les étapes 4 à 11, le script les enchaîne
pour toi :

```bash
./build_linux.sh
```

Il fait exactement la même chose : venv, dépendances, gel, arborescence,
fichiers de contrôle, compression. Le numéro de version vient toujours du fichier
`VERSION`, et le mainteneur de `packaging/control.in`.

Tu dois voir défiler, dans cet ordre :

```
Successfully installed pip-...           (les dépendances)
copying ... -> build/exe/...             (le gel)
dpkg-deb: construction du paquet « presik-pos » dans « dist/... »
Built artifacts are in .../dist
```

## Si une étape échoue

Les trois pannes les plus courantes, et leur cause réelle.

| Ce que tu lis | Ce qui se passe | Quoi faire |
|---|---|---|
| `No module named venv` à l'étape 4 | `python3-venv` n'est pas installé | reprendre l'étape 0 |
| `fatal error: cups/http.h` à l'étape 4 | les en-têtes de CUPS manquent | `sudo apt install libcups2-dev`, ou retirer la ligne `pycups` de `requirements.txt` |
| `dpkg-deb: command not found` à l'étape 11 | `dpkg-dev` n'est pas installé | reprendre l'étape 0 |
| le `.deb` fait quelques kilo-octets | la compression a été interrompue | supprimer le fichier et refaire l'étape 11 |
| `cannot find ./opt/presik_pos/...` à l'étape 14 | l'arborescence de l'étape 6 est incomplète | vérifier que `$DEB` pointe bien quelque part, avec `echo "$DEB"` |

En cas de doute, tu peux tout reprendre à zéro sans rien casser :

```bash
rm -rf build dist .venv
```

Les sources ne sont pas touchées, seuls les produits du build disparaissent.

## Étape 12. Vérifier les métadonnées

```bash
dpkg-deb -f dist/presik-pos_6.0.75+4_amd64.deb Package Version Maintainer
```

Tu dois voir :

```
Package: presik-pos
Version: 6.0.75+4
Maintainer: Prénom Nom <adresse@exemple.org>
```

## Étape 13. Vérifier que l'archive est saine

```bash
dpkg-deb --fsys-tarfile dist/presik-pos_6.0.75+4_amd64.deb > /dev/null && echo "archive OK"
dpkg-deb -c dist/presik-pos_6.0.75+4_amd64.deb | wc -l
```

Tu dois voir `archive OK`, puis un nombre autour de 4566. Aucun message d'erreur.
Si la compression a été coupée en route, c'est ici que ça se voit.

## Étape 14. Vérifier la configuration qui partira chez l'utilisateur

C'est le contrôle le plus important de toute la procédure.

```bash
dpkg-deb --fsys-tarfile dist/presik-pos_6.0.75+4_amd64.deb \
  | tar -xO ./opt/presik_pos/config_pos.ini | grep -E "^server=|^database=|^user="
```

Tu dois voir trois champs vides :

```
server=
database=
user=
```

Si une adresse apparaît, c'est celle de ta machine de travail. Reprends à
l'étape 3 et reconstruis. Ne diffuse pas ce paquet.

## Étape 15. Essayer le paquet sur la machine, puis noter son empreinte

```bash
sudo apt install ./dist/presik-pos_6.0.75+4_amd64.deb
dpkg -l presik-pos
md5sum dist/presik-pos_6.0.75+4_amd64.deb
```

Tu dois voir une ligne commençant par `ii`. Garde l'empreinte md5 : elle permet à
celui qui télécharge de vérifier que le fichier est arrivé entier.

Un paquet qui s'installe n'est pas encore un paquet qui marche. Déroule la
[procédure C](#c-configurer-et-démarrer) et fais une vente d'essai avant de le
diffuser.

# B. Installer un poste

Il te faut le fichier `.deb`, un compte qui a `sudo`, et un serveur Tryton
joignable depuis cette machine. Les conditions côté serveur sont rappelées à la
fin de cette procédure.

## Étape 1. Vérifier que la machine est compatible

```bash
dpkg --print-architecture
ldd --version | head -1
```

Tu dois voir `amd64`, et une version de glibc supérieure ou égale à 2.31, ce qui
veut dire Debian 11 ou plus, Ubuntu 20.04 ou plus.

En dessous de 2.31, le paquet s'installe mais l'application ne démarre pas. Sur
une autre architecture, n'installe pas du tout.

## Étape 2. Vérifier la place disponible

```bash
df -h /opt
```

Tu dois voir au moins 700 Mo de libre.

## Étape 3. Installer

```bash
sudo apt install ./presik-pos_6.0.75+4_amd64.deb
```

Attention au `./` devant le nom. Sans lui, `apt` va chercher un paquet appelé
« presik-pos » dans les dépôts, et te répond qu'il ne le trouve pas.

Tu dois voir, à la fin :

```
Dépaquetage de presik-pos (6.0.75+4) ...
Paramétrage de presik-pos (6.0.75+4) ...
Traitement des actions différées (« triggers ») pour hicolor-icon-theme ...
```

Compte une à deux minutes. Il y a 162 Mo à décompresser.

## Étape 4. Vérifier l'installation

```bash
dpkg -l presik-pos
```

Tu dois voir une ligne qui commence par `ii` :

```
ii  presik-pos  6.0.75+4  amd64  Client point de vente Presik POS (build local)
```

`ii` veut dire installé et configuré. Si tu vois autre chose, par exemple `iU`,
`iHR` ou `iF`, c'est que l'installation s'est interrompue. Va à la
[procédure F](#f-réparer-une-installation-coupée).

## Étape 5. Vérifier qu'aucun résidu ne traîne

```bash
find /opt/presik_pos -name '*.dpkg-*'
```

Tu ne dois rien voir du tout. Pas une ligne. Un fichier `.dpkg-new` ou
`.dpkg-tmp` signifie que le dépaquetage n'est pas allé au bout.

## Étape 6. Vérifier que le lanceur est en place

```bash
which presik-pos
ls /usr/share/applications/presik-pos.desktop
```

Tu dois voir `/usr/bin/presik-pos`, puis le chemin du fichier `.desktop`.

## Étape 7. Joindre le serveur par son nom (facultatif)

C'est utile quand l'adresse IP du serveur change, par exemple avec une box qui
redémarre.

```bash
sudo apt install -y avahi-daemon libnss-mdns
sudo systemctl enable --now avahi-daemon
ping -c 2 <nom-du-serveur>.local
```

Tu dois voir deux réponses au ping. À faire sur le poste et sur le serveur.

Si le ping reste muet, le réseau bloque sans doute le multicast, ce qui est
courant en Wi-Fi invité. Ne cherche pas plus loin, utilise l'adresse IP.

> Le serveur, de son côté, doit réunir trois conditions. Il tourne en
> Tryton 7.0 ou 8.0, pas 6.x. Les modules de vente de Presik y sont installés et
> activés sur la base. Enfin, un magasin, un terminal de caisse et un
> utilisateur rattaché aux deux existent déjà. Si tu n'administres pas le
> serveur, transmets cette liste à celui qui s'en occupe.

# C. Configurer et démarrer

## Étape 1. Premier lancement

```bash
presik-pos
```

Lance-le depuis un terminal la première fois. Les erreurs s'affichent là, et
nulle part ailleurs.

Tu dois voir l'écran de connexion, en français. Pour passer en espagnol, ouvre
le menu **Options**, puis **Langue**, puis **Espagnol** : l'écran change tout de
suite, sans redémarrage. En espagnol, le même menu s'appelle **Opciones**, puis
**Idioma**. Ton choix est gardé dans `~/.config/presik_pos/language.conf`.

## Étape 2. Renseigner le serveur

À l'écran de connexion, ouvre le menu **Options**, puis **Serveur**.
Saisis l'adresse IP ou le nom `.local` du serveur. C'est enregistré tout de
suite, sans redémarrage.

Si tu préfères passer par le fichier :

```bash
nano ~/.config/presik_pos/config_pos.ini
```

```ini
[General]
server=192.168.1.20          # IP ou nom du serveur
port=8000
mode=http                    # http ou https
database=nom_de_la_base      # le nom de la base Tryton
user=                        # utilisateur pré-rempli, facultatif
```

## Étape 3. Vérifier que le serveur répond

Fais-le avant d'essayer de te connecter.

```bash
nc -vz <serveur> 8000
```

Tu dois voir `Connected` ou `succeeded`. Si le port est injoignable, inutile
d'aller plus loin : le problème est dans le réseau ou dans le service serveur,
pas dans la caisse.

## Étape 4. Se connecter

| Champ | Ce qu'il attend | L'erreur classique |
|---|---|---|
| Entreprise | le nom de la base Tryton | y mettre le nom commercial de la société |
| Utilisateur | un utilisateur de caisse existant | y mettre `admin`, qui n'est pas un caissier |
| Mot de passe | le sien | |

Tu dois voir l'écran de vente s'ouvrir, avec les boutons de catégories.

Si l'application se ferme juste après la connexion, le problème est côté
serveur : il manque le magasin, le terminal, ou le rattachement de
l'utilisateur. Regarde l'[annexe 1](#annexe-1-les-messages-derreur).

## Étape 5. La vente d'essai

Avant de laisser la caisse à quelqu'un, déroule une vente complète : un article,
un encaissement, un ticket imprimé. C'est le seul test qui prouve que toute la
chaîne fonctionne.

## Étape 6. Les réglages de confort

```ini
enviroment=restaurant        # restaurant (tables) ou retail (comptoir)
theme=base                   # base pour le clair, dark pour le sombre
mode_window=maximized        # maximized ou fullscreen
print_receipt=automatic      # automatic ou manually
```

Redémarre l'application après avoir modifié le fichier, sinon elle garde les
anciennes valeurs.

# D. Brancher l'imprimante

## Étape 1. Identifier le branchement

```bash
ls /dev/usb/lp*          # imprimante en USB
lpstat -p                # imprimantes connues de CUPS
```

## Étape 2. Déclarer l'imprimante

Dans `~/.config/presik_pos/config_pos.ini`, une seule de ces trois lignes :

```ini
printer_sale_name=usb,/dev/usb/lp0           # USB direct
printer_sale_name=network,192.168.0.36:9100  # imprimante réseau
printer_sale_name=cups,NOM-DE-LIMPRIMANTE    # via CUPS
```

## Étape 3. Régler la largeur du ticket

```ini
row_characters=48     # rouleau de 80 mm
row_characters=32     # rouleau de 58 mm
```

Un mauvais réglage se voit dès le premier ticket : les lignes sont coupées ou
décalées.

## Étape 4. Les droits, en USB

Si rien ne sort de l'imprimante, c'est presque toujours ça.

```bash
sudo usermod -aG lp $USER
```

Déconnecte puis reconnecte ta session. Relancer l'application ne suffit pas :
l'appartenance aux groupes est lue à l'ouverture de session.

## Étape 5. Vérifier

```bash
groups | grep lp
```

Tu dois voir `lp` dans la liste. Imprime ensuite un ticket d'essai depuis
l'application.

# E. Mettre à jour un poste

## La méthode normale

```bash
sudo apt install ./presik-pos_<nouvelle-version>_amd64.deb
dpkg -l presik-pos
```

Tu dois voir `ii` et le nouveau numéro de version.

La configuration du poste est conservée. Le lanceur ne recopie le modèle que si
`~/.config/presik_pos/config_pos.ini` n'existe pas encore.

## Le correctif d'urgence

C'est pour changer une ligne de code sur une caisse en production sans
reconstruire les 162 Mo. L'application est gelée en bytecode, il faut donc
compiler le fichier et le poser à la main.

**Étape 1.** Compiler, depuis la racine des sources :

```bash
python3 -c "import py_compile; py_compile.compile('app/ui/payment_panel.py', \
  cfile='/tmp/payment_panel.pyc', dfile='app/ui/payment_panel.py', doraise=True)"
```

**Étape 2.** Vérifier la version de Python :

```bash
python3 --version
```

Elle doit être 3.13, la même que celle embarquée dans le paquet. Un bytecode
compilé par un autre Python porte un numéro magique différent, et l'application
refusera de démarrer.

**Étape 3.** Installer le fichier :

```bash
sudo install -m 644 /tmp/payment_panel.pyc /opt/presik_pos/lib/app/ui/payment_panel.pyc
```

**Étape 4.** Relancer l'application et vérifier que le correctif fait son effet.

> C'est du dépannage, pas une méthode. Au prochain `apt install`, ton correctif
> disparaît. Pense à le reporter dans les sources.

# F. Réparer une installation coupée

Utilise cette procédure après une coupure de courant, un PC éteint pendant
l'installation, ou un `Ctrl+C` malheureux.

## Étape 1. Confirmer le diagnostic

```bash
dpkg -l presik-pos
find /opt/presik_pos -name '*.dpkg-*' | head
```

Les symptômes : la ligne commence par `iHR`, `iU` ou `iF` au lieu de `ii`, et des
fichiers `.dpkg-new` ou `.dpkg-tmp` traînent dans `/opt/presik_pos`.

Ne lance pas l'application dans cet état. Elle est un mélange d'ancien et de
neuf.

## Étape 2. Terminer les paramétrages en attente

```bash
sudo dpkg --configure -a
```

## Étape 3. Réinstaller

```bash
sudo apt install -y --reinstall ./presik-pos_6.0.75+4_amd64.deb
```

L'étape 2 seule ne suffit pas quand c'est le dépaquetage lui-même qui a été
coupé. Il faut bien la réinstallation derrière.

## Étape 4. Vérifier

```bash
dpkg -l presik-pos                    # doit afficher ii
find /opt/presik_pos -name '*.dpkg-*' # ne doit rien afficher
```

Tu dois voir `ii`, et aucun résidu. Si `apt` refuse en parlant d'une version plus
ancienne, ajoute `--allow-downgrades`.

# G. Désinstaller

Pour retirer l'application en gardant la configuration :

```bash
sudo apt remove presik-pos
dpkg -l presik-pos | tail -1
```

Tu dois voir une ligne qui commence par `rc`. Le paquet est retiré, ses fichiers
de configuration sont gardés.

Pour tout retirer :

```bash
sudo apt remove presik-pos
rm -rf ~/.config/presik_pos
```

Cela efface aussi l'adresse du serveur et le choix de la langue de cet
utilisateur.

# Annexe 1. Les messages d'erreur

Lance toujours la caisse depuis un terminal pour voir passer les erreurs.

```bash
presik-pos
```

| Ce que tu lis | Ce que c'est vraiment | Quoi faire |
|---|---|---|
| `TypeError: string indices must be integers` | un appel au serveur a échoué, le code a reçu une chaîne d'erreur réseau là où il attendait une liste | vérifier le réseau avec `nc -vz <serveur> 8000` |
| `JSONDecodeError: zero-length document` | le serveur a répondu 401 avec un corps vide, le jeton de session n'est pas passé | vérifier le mot de passe, puis que le serveur est bien en Tryton 7.0 ou 8.0 |
| `JSONDecodeError: unexpected character` | le serveur a renvoyé une page d'erreur HTML au lieu de données | côté serveur, un module de vente manque ou n'est pas activé |
| `ValueError: too many values to unpack` | la configuration de vente est incomplète côté serveur | à faire compléter par l'administrateur |
| l'application se ferme juste après la connexion | le magasin, le terminal ou le rattachement de l'utilisateur manque | à créer côté serveur |
| la fenêtre ne s'ouvre pas, erreur `libxcb` | une bibliothèque système manque | `sudo apt install -f` |

L'ordre de diagnostic, du plus proche au plus lointain :

```bash
ping <serveur>                       # 1. la machine répond-elle
nc -vz <serveur> 8000                # 2. le port est-il ouvert
avahi-resolve -n <serveur>.local     # 3. le nom se résout-il
presik-pos                           # 4. que dit l'application
```

Ça évite de chercher un problème de configuration alors que le câble est
débranché.

# Annexe 2. Ce que corrige ce build

**L'authentification.** C'est le correctif central. Le client appelait
`/<base>/fast_login` et lisait le jeton de session dans le corps de la réponse.
Cette adresse n'existe plus. Le client essaie d'abord `/<base>/session/login`,
la forme de Tryton 8 : le corps ne contient que l'identifiant de l'utilisateur
et le jeton arrive dans un cookie `tryton_session=login:user_id:token`. Si le
serveur ne connaît pas cette adresse, il répond 404 ou 405, et le client
repasse alors par `/<base>/`, la forme de Tryton 7.0, qui renvoie l'identifiant
et le jeton ensemble dans le corps. Dans les deux cas la méthode appelée est
`common.db.login` et la suite des échanges part avec un en-tête
`Authorization: Session`. Sans ce correctif, chaque appel suivant repart sans
en-tête et reçoit un 401 au corps vide.

**La traduction française.** 591 chaînes : écran de connexion, écran de vente,
recherches, facturation, panneau de contrôle, aide, rapports, dialogues et
lignes de ticket imprimées. Les boutons standard des boîtes de dialogue (Oui,
Non, Annuler) sont traduits aussi, grâce aux traductions fournies avec Qt.

**Le choix de la langue dans le menu Options**, écrit dans la langue active,
appliqué sans redémarrage et gardé dans `~/.config/presik_pos/language.conf`.

**L'écran de connexion réorganisé.** Les étiquettes ne chevauchent plus les
champs, et le bouton Quitter est devenu discret.

**Le réglage du serveur depuis l'écran de connexion**, pour ne pas avoir à
éditer le fichier INI sur chaque poste.

**L'alias de thème.** `theme=base` désigne explicitement le thème clair.

# Annexe 3. Ce qu'il y a dans le paquet

162 Mo, 4566 fichiers, architecture amd64.

| Où | Quoi |
|---|---|
| `/opt/presik_pos/` | l'application gelée, avec Python 3.13 et PySide6 embarqués |
| `/opt/presik_pos/config_pos.ini` | le modèle de configuration, livré vide |
| `/opt/presik_pos/locale/` | les traductions compilées |
| `/usr/bin/presik-pos` | le lanceur |
| `/usr/share/applications/` | l'entrée du menu des applications |
| `/usr/share/icons/hicolor/256x256/apps/` | l'icône |

Le lanceur fait une seule chose avant de démarrer l'application : il copie le
modèle de configuration vers `~/.config/presik_pos/config_pos.ini`, mais
seulement si ce fichier n'existe pas encore. C'est ce qui garantit qu'une mise à
jour n'écrase jamais la configuration d'un poste.

Les dépendances système sont récupérées par `apt` à l'installation :
`libxcb-cursor0`, `libegl1`, `libglib2.0-0`, `libdbus-1-3`, `libfontconfig1`,
`libfreetype6` et `libxkbcommon0`. La seule exigence non négociable est glibc
2.31 au minimum.

# Annexe 4. Les limites connues

**Le client dépend entièrement des modules de vente de Presik.** Ce paquet ne
les contient pas et ne peut pas les remplacer. Si la base ne les a pas, la
caisse ne dépassera pas l'écran de connexion.

**Certains champs de ticket restent vides** quand le serveur ne les alimente
pas. Ce sont des mentions fiscales propres au pays d'origine du logiciel.

**Les paiements électroniques et la facturation hôtelière** dépendent de modules
serveur distincts. Le client ne les interroge que s'ils sont déclarés actifs.

**La langue ne se change qu'à l'écran de connexion.** L'écran de vente n'a pas de
barre de menu. Déconnecte-toi pour en changer.

**Le paquet est en amd64 seulement.** Pas d'ARM, donc pas de Raspberry Pi.

**Ne lance jamais `lupdate` sur les sources sans avoir sauvegardé
`locale/*.ts`.** Son analyseur ne reconnaît pas toutes les formes d'appel à
`self.tr()`, et il supprime les traductions qu'il juge obsolètes.

*Guivens Chery, Presik POS 6.0.75+4*
