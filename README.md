# Presik POS 6.0.75+3, build local

Client de caisse Presik POS reconstruit pour un serveur Tryton 8.0.

> Ce n'est pas une distribution officielle de Presik SAS. Ce paquet est
> construit localement et contient des correctifs qui n'existent pas en amont :
> l'authentification par cookie de session de Tryton 8 et l'interface traduite
> en français.

## Télécharger et installer

Le paquet Debian est attaché à la [dernière release](../../releases/latest) :
`presik-pos_6.0.75+3_amd64.deb`, 162 Mo.

```bash
sudo apt install ./presik-pos_6.0.75+3_amd64.deb
```

Il faut Debian 11 ou plus, Ubuntu 20.04 ou plus, ou un dérivé, en amd64, avec
glibc 2.31 au minimum. Python et PySide6 sont embarqués dans le paquet, il n'y a
rien d'autre à installer.

Le paquet ne contient aucune adresse de serveur ni aucun compte. Chacun
renseigne son installation au premier démarrage, par le menu **Options**,
entrée **Serveur**. La langue se choisit dans le même menu, entrée **Langue**.

## Comment ce paquet a été créé

Voici les étapes qui ont produit le fichier `.deb`, dans l'ordre où elles ont
été faites. La version détaillée, avec la sortie attendue à chaque commande et
les pannes courantes, est dans [GUIDE_PRESIK_POS.md](GUIDE_PRESIK_POS.md).

### 1. Installer les outils de construction

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-dev \
    build-essential dpkg-dev libcups2-dev libusb-1.0-0-dev
```

`python3-venv` crée l'environnement isolé. `build-essential` et `python3-dev`
fournissent le compilateur et les en-têtes sans lesquels `pyusb` et `pycups` ne
se construisent pas. `dpkg-dev` apporte la commande `dpkg-deb`.

### 2. Fixer le numéro de version

```bash
echo "6.0.75+3" > VERSION
```

Ce fichier est la seule source du numéro. Il se retrouve dans le nom du `.deb`
et dans les métadonnées du paquet.

### 3. Vider la configuration d'exemple

```bash
grep -E "^server=|^database=|^user=" config_pos.ini
```

Les trois champs doivent être vides. Ce fichier est recopié chez chaque
utilisateur au premier démarrage : une adresse de serveur oubliée là partirait
avec le paquet, chez tout le monde.

### 4. Créer l'environnement Python et installer les dépendances

```bash
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m pip install "cx-Freeze>=8.4"
```

### 5. Geler l'application

C'est ici que Python et Qt entrent dans le paquet.

```bash
.venv/bin/python setup_freeze.py build_exe --build-exe build/exe
```

Compte cinq à huit minutes. Le dossier `build/exe` fait environ 614 Mo à la fin.

### 6. Construire l'arborescence du paquet

Un paquet Debian est un dossier qui reproduit l'arborescence d'installation,
plus un dossier `DEBIAN` pour les métadonnées.

```bash
VERSION="6.0.75+3"
DEB="dist/presik-pos_${VERSION}_amd64"

mkdir -p "$DEB/opt/presik_pos" \
         "$DEB/usr/bin" \
         "$DEB/usr/share/applications" \
         "$DEB/usr/share/icons/hicolor/256x256/apps" \
         "$DEB/DEBIAN"

cp -a build/exe/. "$DEB/opt/presik_pos/"
cp app/resources/share/icon.png \
   "$DEB/usr/share/icons/hicolor/256x256/apps/presik-pos.png"
```

### 7. Écrire le lanceur

Il sera installé dans `/usr/bin/presik-pos`.

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

La ligne `[ -f "$CONF" ] || cp "$GABARIT" "$CONF"` ne copie le modèle que si la
configuration de l'utilisateur n'existe pas encore. C'est ce qui garantit qu'une
mise à jour du paquet n'écrase jamais les réglages d'un poste.

### 8. Écrire l'entrée du menu des applications

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

### 9. Écrire le fichier de contrôle

C'est la carte d'identité du paquet.

```bash
cat > "$DEB/DEBIAN/control" <<EOF
Package: presik-pos
Version: $VERSION
Section: office
Priority: optional
Architecture: amd64
Depends: libc6 (>= 2.31), libxkbcommon0, libegl1, libglib2.0-0, libdbus-1-3, libfontconfig1, libfreetype6, libxcb-cursor0
Recommends: cups, libusb-1.0-0, avahi-daemon, libnss-mdns
Maintainer: Guivens Chery <cheryguy030@gmail.com>
Homepage: https://www.presik.com
Description: Client point de vente Presik POS (build local)
 Build local de Presik POS 6.0.75, adapté à un serveur Tryton 8.0.
 Il n'est pas distribué par Presik SAS et contient des correctifs locaux.
 .
 Le paquet embarque Python, PySide6 et toutes les dépendances applicatives.
EOF
```

`Depends` liste les bibliothèques que `apt` installera automatiquement.
`libc6 (>= 2.31)` empêche l'installation sur une machine trop ancienne.

### 10. Écrire les scripts d'installation

Ils rafraîchissent le menu des applications et le cache des icônes.

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

### 11. Fabriquer le fichier .deb

```bash
dpkg-deb --build --root-owner-group "$DEB" "dist/presik-pos_${VERSION}_amd64.deb"
```

La commande ne montre rien pendant trois à cinq minutes : c'est la compression
des 614 Mo, elle n'est pas bloquée. Le fichier final fait environ 162 Mo.

L'option `--root-owner-group` est importante. Sans elle, les fichiers du paquet
appartiendraient au compte qui a construit, et l'installation poserait des
fichiers avec un propriétaire inexistant sur la machine de destination.

### 12. Vérifier le paquet avant de le diffuser

```bash
dpkg-deb -f dist/presik-pos_6.0.75+3_amd64.deb Package Version Maintainer
dpkg-deb --fsys-tarfile dist/presik-pos_6.0.75+3_amd64.deb > /dev/null && echo "archive OK"
dpkg-deb -c dist/presik-pos_6.0.75+3_amd64.deb | wc -l
```

Les métadonnées doivent être justes, l'archive doit se décompresser sans erreur,
et le nombre de fichiers tourner autour de 4565.

Puis le contrôle le plus important, celui qui montre la configuration qui
partira chez l'utilisateur :

```bash
dpkg-deb --fsys-tarfile dist/presik-pos_6.0.75+3_amd64.deb \
  | tar -xO ./opt/presik_pos/config_pos.ini | grep -E "^server=|^database=|^user="
```

Les trois champs doivent être vides. Si une adresse apparaît, c'est celle de la
machine de construction : il faut reprendre à l'étape 3 et reconstruire.

### 13. Essayer le paquet

```bash
sudo apt install ./dist/presik-pos_6.0.75+3_amd64.deb
dpkg -l presik-pos
md5sum dist/presik-pos_6.0.75+3_amd64.deb
```

La ligne doit commencer par `ii`, ce qui veut dire installé et configuré.
L'empreinte md5 permet à celui qui télécharge de vérifier que le fichier est
arrivé entier.

Un paquet qui s'installe n'est pas encore un paquet qui marche : il reste à le
connecter à un serveur et à dérouler une vente d'essai.

## Le guide complet

[GUIDE_PRESIK_POS.md](GUIDE_PRESIK_POS.md) reprend tout cela en détail, avec la
sortie attendue de chaque commande, et ajoute six autres procédures :

| Procédure | Pour qui |
|---|---|
| A. Construire le paquet | celui qui fabrique |
| B. Installer un poste | celui qui déploie |
| C. Configurer et démarrer | celui qui déploie |
| D. Brancher l'imprimante | celui qui déploie |
| E. Mettre à jour un poste | maintenance |
| F. Réparer une installation coupée | maintenance |
| G. Désinstaller | maintenance |

Puis quatre annexes : les messages d'erreur et leur traduction, ce que corrige
ce build, ce qu'il y a dans le paquet, et les limites connues.

## Licence

GPL v3, voir [LICENSE](LICENSE).

*Guivens Chery*
