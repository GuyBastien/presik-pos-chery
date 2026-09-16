# Presik POS 6.0.75+1 — build local

Client de caisse Presik POS reconstruit pour un serveur **Tryton 8.0**.

> Ce n'est pas une distribution officielle de Presik SAS. Ce paquet est
> construit localement et contient des correctifs qui n'existent pas en amont :
> authentification par cookie de session (Tryton 8), import Qt manquant dans
> 23 fichiers, et interface traduite en français.

## Télécharger

Le paquet Debian est attaché à la [dernière release](../../releases/latest) :
`presik-pos_6.0.75+1_amd64.deb` (162 Mo).

## Installer

```bash
sudo apt install ./presik-pos_6.0.75+1_amd64.deb
```

Debian 11+ / Ubuntu 20.04+ ou dérivé, **amd64**, glibc 2.31 minimum.
Python et PySide6 sont embarqués : rien d'autre à installer.

## Le guide

Tout est dans **[GUIDE_PRESIK_POS.md](GUIDE_PRESIK_POS.md)** : ce que le serveur
doit fournir, l'installation du poste, la configuration, le premier démarrage,
le dépannage et la reconstruction du paquet.

Le paquet ne contient **aucune adresse de serveur ni aucun compte** : tu
renseignes ta propre installation au premier démarrage.

## Licence

GPL v3 — voir [LICENSE](LICENSE).

---

*Guivens Chery*
