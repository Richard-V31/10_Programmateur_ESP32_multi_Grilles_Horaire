# Programmateur horaire ESP32 — multi-relais avec interface web (grille 24 h)

Ce firmware transforme un ESP32 en programmateur horaire connecté, capable de piloter un nombre **configurable** de relais indépendants, chacun programmable via une page web embarquée (aucune carte SD, aucun système de fichiers requis pour le HTML/CSS/JS : tout est stocké en mémoire flash).

Fichier principal : `Programmateur_horaire_ESP32_multi_relais_Web_grille.ino`

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Matériel nécessaire](#matériel-nécessaire)
- [Câblage (broches par défaut)](#câblage-broches-par-défaut)
- [Bibliothèques Arduino requises](#bibliothèques-arduino-requises)
- [Fichier `arduino_secrets.h`](#fichier-arduino_secretsh)
- [Premier flashage](#premier-flashage)
- [Configuration des relais](#configuration-des-relais)
- [Utilisation de l'interface web](#utilisation-de-linterface-web)
- [Routes HTTP (API)](#routes-http-api)
- [Mise à jour OTA (sans câble USB)](#mise-à-jour-ota-sans-câble-usb)
- [Sauvegarde des réglages (NVS)](#sauvegarde-des-réglages-nvs)
- [Écran OLED](#écran-oled)
- [Limites et points d'attention](#limites-et-points-dattention)

## Fonctionnalités

- **Nombre de relais configurable** : on ajoute/retire une ligne dans le tableau `programmateurs[]`, rien d'autre à modifier — la page web, les routes HTTP, la sauvegarde flash et l'écran OLED s'adaptent automatiquement.
- **Mode AUTOMATIQUE ou MANUEL** par relais : en auto, le relais suit sa grille horaire ; en manuel, l'utilisateur (page web ou bouton poussoir) force l'état ON/OFF.
- **Grille horaire sur 24 h** découpée en créneaux de 15 min par défaut (réglable via `SLOT_MIN`) : on définit autant de plages que l'on veut par relais, y compris à cheval sur minuit (ex : `22:45-06:15`), en cochant des cases sur la page web ou en saisissant une plage "début/fin".
- **Interface web responsive** embarquée dans le firmware, accessible depuis n'importe quel navigateur du réseau local : bascule AUTO/MANUEL, forçage ON/OFF, actions groupées (Tout ON / Tout OFF / Tout AUTO), résumé des plages avec la **plage en cours en vert** et la **prochaine plage en rouge**, compte à rebours avant le prochain changement d'état, popup "Infos système" (WiFi, IP, MAC, build).
- **Boutons poussoirs physiques** de forçage (optionnels, un par relais) avec anti-rebond logiciel.
- **Écran OLED I2C (SSD1306)** affichant le réseau WiFi, l'IP, et l'état de chaque relais (pages tournantes si plus de 5 relais).
- **WiFi multi-réseaux** : jusqu'à 3-4 réseaux connus, connexion automatique au meilleur signal, bascule non bloquante en cas de coupure, retour automatique vers le réseau habituel quand il redevient disponible.
- **Mise à jour OTA** (par WiFi, depuis l'IDE Arduino), protégée par mot de passe, avec affichage de la progression sur l'écran OLED.
- **Persistance en mémoire flash (NVS)** des grilles horaires, modes et états, pour survivre aux coupures de courant et aux redémarrages.
- **mDNS** : l'ESP32 est joignable via `http://richardv.local` en plus de son adresse IP.

## Matériel nécessaire

- Une carte ESP32 (DevKit classique).
- Un ou plusieurs modules relais (autant que de lignes dans `programmateurs[]`).
- Un écran OLED I2C SSD1306 0.96" (128×64, adresse `0x3C`) — optionnel, le firmware fonctionne sans mais désactive l'affichage si non détecté.
- Des boutons poussoirs (optionnels) pour le forçage physique de chaque relais.

## Câblage (broches par défaut)

| Relais | Broche GPIO | Bouton poussoir (GPIO) |
|---|---|---|
| Programmation 1 – Cuisine | 32 | 14 |
| Programmation 2 – Portail | 33 | 16 |
| Programmation 3 | 25 | 17 |
| Programmation 4 | 26 | 18 |

- Écran OLED (I2C) : SDA = GPIO 21, SCL = GPIO 22 (broches par défaut de l'ESP32).
- Bouton poussoir : une broche sur GND, l'autre sur le GPIO indiqué (résistance de tirage interne activée en `INPUT_PULLUP`, aucune résistance externe nécessaire).

⚠️ Broches GPIO utilisables en sortie sur un ESP32 DevKit classique : `4, 5, 13, 14, 16, 17, 18, 19, 21, 22, 23, 25, 26, 27, 32, 33` (21/22 déjà utilisées par l'écran OLED). À éviter : GPIO 34-39 (entrée seule) et les broches de boot (0, 2, 12, 15).

## Bibliothèques Arduino requises

À installer via le gestionnaire de bibliothèques de l'IDE Arduino :

- `WiFi` (fournie avec le core ESP32)
- `ESPAsyncWebServer` (+ sa dépendance `AsyncTCP`)
- `Preferences` (fournie avec le core ESP32)
- `ArduinoJson` (v7)
- `ESPmDNS` (fournie avec le core ESP32)
- `ArduinoOTA` (fournie avec le core ESP32)
- `Wire` (fournie avec le core ESP32)
- `Adafruit GFX Library`
- `Adafruit SSD1306`

## Fichier `arduino_secrets.h`

Ce fichier n'est **pas fourni** (il contient vos identifiants WiFi) et doit être créé dans le même dossier que le `.ino`, sur ce modèle :

```cpp
#define SECRET_SSID  "nom_du_reseau_1"
#define SECRET_PASS  "mot_de_passe_1"

#define SECRET_SSID2 "nom_du_reseau_2"
#define SECRET_PASS2 "mot_de_passe_2"

// Optionnel : décommentez pour un 3e réseau connu
// #define SECRET_SSID3 "nom_du_reseau_3"
// #define SECRET_PASS3 "mot_de_passe_3"

// Fortement recommandé : mot de passe pour sécuriser les mises à jour OTA
#define SECRET_OTA_PASSWORD "votre_mot_de_passe_ota"
```

L'ESP32 scanne les réseaux connus et se connecte automatiquement à celui qui offre le meilleur signal.

## Premier flashage

1. Créez `arduino_secrets.h` (voir ci-dessus).
2. Adaptez si besoin le tableau `programmateurs[]` (voir [Configuration des relais](#configuration-des-relais)).
3. Flashez une première fois **par câble USB** depuis l'IDE Arduino.
4. Une fois démarré et connecté au WiFi, l'ESP32 apparaît comme un port réseau dans **Outils > Port** : les mises à jour suivantes peuvent se faire par OTA (voir plus bas).
5. Accédez à l'interface depuis un navigateur du même réseau : `http://richardv.local` (ou l'adresse IP affichée sur le moniteur série / l'écran OLED).

## Configuration des relais

Tout se règle dans le tableau `programmateurs[]`, en haut du fichier :

```cpp
Programmateur programmateurs[] = {
  { "1.", "Programmation 1", "Cuisine",  "#f59e0b", 32, "06:30-08:00,11:30-13:15,18:45-22:30", true, false, 14 },
  { "2.", "Programmation 2", "Portail",  "#06b6d4", 33, "07:00-09:00,17:00-19:30",             true, false, 16 },
  // ...
};
```

Champs, dans l'ordre : identifiant unique (court, sans espace), nom affiché, sous-titre, couleur d'accent (hex), broche GPIO du relais, plages horaires par défaut (texte, séparées par des virgules, `""` = aucune plage), mode auto par défaut, état par défaut, broche GPIO du bouton poussoir (`-1` = aucun).

- **Ajouter un relais** : dupliquez une ligne, changez au minimum l'id et la broche GPIO.
- **Retirer un relais** : supprimez la ligne correspondante.
- **Changer la finesse des créneaux** : modifiez la constante `SLOT_MIN` (30, 15, 10 ou 5 minutes — doit être un diviseur de 1440). Toute l'interface (page web, NVS, OLED) s'adapte automatiquement.

⚠️ Une fois qu'une grille a été enregistrée depuis la page web, elle est stockée en NVS (mémoire flash) et **prend le pas** sur la valeur `plagesDefaut` du code à chaque redémarrage — sauf juste après une mise à jour OTA, où la NVS est automatiquement réinitialisée (voir [Sauvegarde des réglages](#sauvegarde-des-réglages-nvs)).

## Utilisation de l'interface web

- **Bascule AUTO / MANUEL** : bouton rectangulaire sur chaque ligne de relais.
- **Forçage ON/OFF** (mode manuel) : bouton "⚡ Forcer ON ou OFF".
- **Actions groupées** : "Tout ON", "Tout OFF", "Tout AUTO" en haut de page.
- **Édition de la grille horaire** : un appui sur le résumé des plages (ex. "06:30-08:00, 18:45-22:30") ouvre une fenêtre avec les 24 lignes horaires (une case = un créneau). On peut cocher/décocher case par case, cocher une heure entière, saisir rapidement une plage début/fin, tout effacer, tout cocher (24h/24) ou inverser la sélection, puis valider avec "Enregistrer ✓".
- **Résumé coloré** : sur l'écran principal, la plage actuellement active s'affiche en **vert**, la prochaine plage à venir en **rouge**.
- **Compte à rebours** : "Extinction dans …" / "Allumage dans …", calculé par l'ESP32.
- **Popup "Infos système"** (icône 🛜) : état WiFi, SSID, nom mDNS, IP, adresse MAC, puissance du signal, date/heure de compilation du firmware (utile pour vérifier qu'une OTA a bien pris effet).

## Routes HTTP (API)

| Route | Méthode | Description |
|---|---|---|
| `/` | GET | Sert la page web principale. |
| `/get-config` | GET | Liste des relais (id, nom, sous-titre, couleur) + résolution de la grille (`slotMin`, `nbSlots`). |
| `/get-data` | GET | État complet de tous les relais (grille, résumé, mode, état, temps restant) + heure courante + qualité du signal WiFi. Interrogée chaque seconde par la page web. |
| `/get-info` | GET | Informations système (WiFi, IP, MAC, RSSI, build). |
| `/toggle-mode?id=...` | GET | Bascule un relais entre AUTO et MANUEL. |
| `/force-state?id=...` | GET | Force l'inversion de l'état ON/OFF d'un relais (passe en mode manuel). |
| `/save?id=...` | POST | Enregistre la grille horaire d'un relais. Paramètre `grille` (chaîne hexadécimale complète) **ou** `plages` (texte `"06:30-08:00,18:45-22:30"`). |
| `/reset-auto` | GET | Repasse tous les relais en mode AUTOMATIQUE (sans toucher aux horaires enregistrés). |

Exemple d'appel en ligne de commande, sans passer par la page web :

```bash
curl -X POST "http://richardv.local/save?id=1." -d "plages=06:30-08:00,18:45-22:30"
```

## Mise à jour OTA (sans câble USB)

Une fois l'ESP32 flashé une première fois par USB et connecté au WiFi :

1. Dans l'IDE Arduino, sélectionnez le port réseau correspondant à `richardv` (dans **Outils > Port**).
2. Recompilez puis lancez le téléversement comme d'habitude : la progression s'affiche sur l'écran OLED (et dans le moniteur série).
3. L'OTA est protégée par `SECRET_OTA_PASSWORD` (voir `arduino_secrets.h`) — sans ce mot de passe, n'importe quel appareil du réseau pourrait reflasher l'ESP32.

⚠️ Recompilez avant **chaque** upload OTA : la signature de build (`FIRMWARE_BUILD`, visible dans le popup "Infos système") est le seul moyen fiable de vérifier après coup que le nouveau firmware est bien celui qui tourne.

## Sauvegarde des réglages (NVS)

Les grilles horaires, modes et états sont enregistrés dans la mémoire flash NVS (bibliothèque `Preferences`), namespace `config`. Ils survivent aux coupures de courant et aux redémarrages.

À chaque mise à jour OTA, le firmware compare sa date de compilation à celle mémorisée : si elle a changé, la NVS est automatiquement effacée, pour que les nouvelles valeurs par défaut du code (`programmateurs[]`) ne soient pas masquées par d'anciens réglages enregistrés depuis la page web.

## Écran OLED

Affiche en en-tête le réseau WiFi utilisé et l'heure, puis l'adresse IP, puis une ligne par relais (mode A/M, état ON/OFF, première plage programmée). Si le nombre de relais dépasse 5, l'affichage bascule automatiquement en pages tournantes (une page toutes les 8 secondes). Si l'écran n'est pas détecté au démarrage, le firmware continue de fonctionner normalement sans lui.

## Limites et points d'attention

- Le nombre maximal de relais dépend surtout du nombre de broches GPIO libres sur la carte (une quinzaine sur un ESP32 classique) ; au-delà, il faudrait passer par un module d'extension I2C (non inclus).
- Les boutons poussoirs de forçage physique ne sont câblés par défaut que sur les relais 1 à 4 dans l'exemple fourni (adapter `pinBP` pour d'autres relais).
- L'heure est synchronisée par NTP au démarrage (fuseau horaire France, passage heure été/hiver automatique) ; sans accès Internet, l'ESP32 démarre après un court délai sans heure valide et la logique horaire automatique reste en attente.
