# Firmware ESPHome pour carte d'extension Eletechsup ES32D26

> **Statut : travail en cours (work in progress).** Cette documentation reflète l'état actuel du projet et sera mise à jour au fur et à mesure de son évolution. Certaines fonctionnalités sont fonctionnelles et calibrées, d'autres sont encore en cours de validation ou de calibration (voir la section [Problèmes connus et travaux en cours](#problèmes-connus-et-travaux-en-cours)).

## Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Prérequis matériel](#prérequis-matériel)
- [Architecture générale](#architecture-générale)
- [Relais (sorties numériques)](#relais-sorties-numériques)
- [Entrées numériques](#entrées-numériques)
- [Sorties analogiques](#sorties-analogiques)
- [Entrées analogiques](#entrées-analogiques)
- [RS485 / Modbus](#rs485--modbus)
- [Broches GPIO libres](#broches-gpio-libres)
- [Diagnostics WiFi](#diagnostics-wifi)
- [Sécurité et secrets](#sécurité-et-secrets)
- [Problèmes connus et travaux en cours](#problèmes-connus-et-travaux-en-cours)

## Vue d'ensemble

Ce dépôt contient un firmware [ESPHome](https://esphome.io/) développé pour la carte d'extension industrielle **Eletechsup ES32D26**, intégrée à [Home Assistant](https://www.home-assistant.io/). La carte ES32D26 est un contrôleur d'entrées/sorties conçu autour d'un socket pour module ESP32, offrant :

- 8 sorties relais
- 8 entrées numériques
- 2 sorties analogiques (tension 0-10V ou courant 0-20mA, selon la position d'un DIP switch)
- 4 entrées analogiques en tension et 4 en courant
- Un port de communication RS485
- Plusieurs broches GPIO du module ESP32 amenées à un bornier libre, sans fonction prédéfinie, permettant d'ajouter des fonctions supplémentaires

Le firmware exposé ici pilote l'ensemble de ces fonctions via ESPHome, avec une attention particulière portée à ce que les réglages courants (modes de fonctionnement, calibration, comportement au démarrage) soient ajustables directement depuis l'interface Home Assistant, sans nécessiter de modification ou de re-flashage du firmware.

## Prérequis matériel

Le socket de la ES32D26 est conçu pour recevoir un module ESP32 au format **DevKit à 38 broches**. Ceci est une exigence stricte :

- **Utiliser un module ESP32 à 38 broches uniquement.** Les modules à 30 broches ou 36 broches (variantes ESP32-S, certains modèles "NodeMCU-32S", etc.) ne sont **pas compatibles** avec ce socket : leur brochage ne correspond pas à celui attendu par la carte, ce qui peut entraîner un mauvais alignement des broches critiques (horloges, lignes de données) et potentiellement endommager le module ou la carte.
- La ES32D26 accepte deux espacements de broches entre les deux rangées : **0.9 pouce ou 1.0 pouce**, selon le modèle de module ESP32 DevKit utilisé — les deux sont pris en charge par le socket.
- Modules recommandés : **ESP32-WROOM-32D**, ou **ESP32-WROOM-32U** si une antenne externe est souhaitée (connecteur IPEX/U.FL déporté plutôt qu'antenne PCB intégrée).
- **Avant tout achat**, vérifier que l'ordre des broches du module correspond bien à celui identifié sur le socket de la ES32D26 (généralement indiqué par sérigraphie sur la carte). Les fabricants de modules ESP32 ne respectent pas toujours un brochage strictement identique d'un modèle à l'autre.

## Architecture générale

Le firmware utilise les composants natifs ESPHome suivants comme fondation :

- [`sn74hc595`](https://esphome.io/components/sn74hc595) pour piloter les 8 relais via le registre à décalage de sortie.
- [`sn74hc165`](https://esphome.io/components/sn74hc165) pour lire les 8 entrées numériques via le registre à décalage d'entrée.
- [`esp32_dac`](https://esphome.io/components/output/esp32_dac) pour les 2 canaux de sortie analogique.
- [`adc`](https://esphome.io/components/sensor/adc) pour les entrées analogiques.
- `uart` / `modbus` pour l'infrastructure RS485.

Là où c'est pertinent, des composants `select`, `number` et `button` sont utilisés pour exposer des réglages ajustables (modes de fonctionnement, calibration, mesures ponctuelles) directement dans l'interface Home Assistant, avec persistance (`restore_value: true`) pour que ces réglages survivent à un redémarrage du module ESP32 ou de Home Assistant.

## Relais (sorties numériques)

Les 8 relais sont pilotés via un registre à décalage 74HC595 :

| Signal | GPIO |
|---|---|
| Data (SER) | GPIO12 |
| Clock (SRCLK) | GPIO22 |
| Latch (RCLK) | GPIO23 |
| Output Enable (OE) | GPIO13 |

Un interrupteur maître ("CH 1-8 - All Relais") permet d'activer ou désactiver les 8 relais simultanément, avec un état calculé reflétant si tous les relais sont dans le même état (ON, OFF, ou mixte).

Chaque relais dispose d'un sélecteur de **mode de démarrage**, choisissable depuis Home Assistant :

- **Toujours OFF** : le relais reste éteint après un redémarrage, peu importe son état précédent.
- **Toujours ON** : le relais s'active automatiquement au démarrage.
- **Dernier état** : le relais retrouve l'état dans lequel il se trouvait avant le redémarrage.

Ce choix persiste à travers les redémarrages du ESP32 et de Home Assistant.

### Comportement transitoire au démarrage

Un bref clignotement (activation puis désactivation immédiate) peut être observé sur certains relais au moment de la mise sous tension de la carte. Ce comportement est documenté et connu sur les circuits basés sur le 74HC595 : avant que le microcontrôleur n'ait terminé son initialisation et n'impose un état défini sur les broches de contrôle du registre, ce dernier peut se retrouver brièvement dans un état électrique indéterminé (broches de contrôle flottantes, montée progressive de la tension d'alimentation), ce qui peut activer une sortie de façon transitoire avant que le firmware ne prenne le relais.

Une solution matérielle existe (résistance de tirage sur la broche de reset du registre ou sur l'Output Enable) mais n'a pas été appliquée dans la version actuelle — le contournement retenu pour l'instant est logiciel, via la stratégie de pré-validation du firmware décrite ci-dessous.

## Entrées numériques

Les 8 entrées numériques sont lues via un registre à décalage 74HC165 :

| Signal | GPIO |
|---|---|
| Clock (CLK) | GPIO2 |
| Data (QH) | GPIO15 |
| Load (SH/LD) | GPIO0 |

Les entrées sont exposées comme capteurs binaires (`binary_sensor`) avec logique inversée (`inverted: true`), correspondant au comportement natif du registre observé sur cette carte.

**Note de validation matérielle** : GPIO0, GPIO2 et GPIO15 sont des broches de "strapping" sur le ESP32 (impliquées dans le mode de démarrage du module). Un défaut matériel sur un module ESP32 spécifique (GPIO2 court-circuité à la masse) a été rencontré durant le développement, causant une non-fonctionnalité totale du registre d'entrées — ce n'était pas un problème de configuration logicielle mais un défaut physique du module utilisé à ce moment. Il est recommandé de valider le bon fonctionnement du module ESP32 (en particulier ces broches de strapping) avant un déploiement prolongé.

## Sorties analogiques

La carte expose 2 canaux de sortie analogique, chacun pouvant fonctionner en tension (0-10V) ou en courant (0-20mA), sélectionné physiquement via le DIP switch SW1 de la carte :

| Canal | GPIO (DAC) |
|---|---|
| Vo1 / Io1 (mutuellement exclusifs) | GPIO25 |
| Vo2 / Io2 (mutuellement exclusifs) | GPIO26 |

Combinaisons possibles selon la position du DIP switch : Vo1+Vo2, Vo1+Io2, Io1+Vo2, Io1+Io2.

Pour chaque canal, un sélecteur ("Sortie analogique 1/2 - mode d'opération") indique au firmware quelle position occupe réellement le DIP switch, afin d'appliquer la bonne calibration. Un seul contrôle numérique ("Sortie analogique 1/2 %") pilote la sortie de 0 à 100%, peu importe le mode choisi.

### Calibration

Chaque canal dispose d'une calibration à 3 points (0%, 50%, 100%), en tension et en courant séparément, ajustable via des curseurs numériques dans Home Assistant (persistants à travers les redémarrages). La valeur réelle calculée (en V ou en mA selon le mode actif) est affichée via un capteur texte dédié par canal.

## Entrées analogiques

La carte expose 4 entrées en tension (Vi1-4) et 4 entrées en courant (Ii1-4) :

| Entrée | GPIO | Bloc ADC du ESP32 |
|---|---|---|
| Vi1 | GPIO14 | ADC2 |
| Vi2 | GPIO33 | ADC1 |
| Vi3 | GPIO27 | ADC2 |
| Vi4 | GPIO32 | ADC1 |
| Ii1 | GPIO34 | ADC1 |
| Ii2 | GPIO39 | ADC1 |
| Ii3 | GPIO35 | ADC1 |
| Ii4 | GPIO36 | ADC1 |

### Limitation matérielle ADC2 / WiFi

Le ESP32 partage son bloc ADC2 avec le pilote WiFi au niveau du silicium : lorsque le WiFi est actif (nécessaire pour la connexion à Home Assistant), les lectures sur les broches ADC2 (Vi1 et Vi3 dans ce cas) échouent de façon systématique, pas seulement occasionnelle. C'est une limitation documentée du ESP32, pas un défaut de ce firmware.

La solution retenue : Vi1 et Vi3 ne sont plus lus en continu. Un bouton dédié ("Vi1 / Vi3 - Mesure ponctuelle") coupe complètement le WiFi, effectue une lecture fiable sur les deux broches (plus aucun conflit possible), republie les valeurs, puis rétablit la connexion WiFi. L'appareil disparaît quelques secondes de Home Assistant pendant cette opération — c'est un comportement attendu, pas une erreur. Toute commande envoyée à un autre composant (par exemple un relais) pendant cette fenêtre serait perdue, sans réessai automatique.

Vi2 et Vi4, sur le bloc ADC1, ne sont pas affectés par cette limitation et sont lus en continu normalement.

### Calibration

Vi1, Vi2 et Vi4 disposent d'une calibration empirique (2 ou 3 points selon le canal). Les entrées de courant (Ii1-4) utilisent une échelle 4-20mA (plutôt que 0-20mA, comportement confirmé par test avec un générateur de signal externe), avec une correction à 3 points pour Ii1 actuellement implémentée. Voir la section [Problèmes connus](#problèmes-connus-et-travaux-en-cours) pour l'état de calibration actuel.

## RS485 / Modbus

L'infrastructure de communication RS485 est en place :

| Signal | GPIO |
|---|---|
| TX (UART0) | GPIO1 |
| RX (UART0) | GPIO3 |
| DE/RE (contrôle direction) | GPIO21 |

Un indicateur de diagnostic ("RS485 Prêt") confirme que le port s'est initialisé correctement au démarrage — il ne détecte pas la présence d'un appareil réel sur le bus (Modbus n'offre pas de mécanisme générique pour ça), seulement que le port lui-même est opérationnel.

**Note technique** : GPIO1 et GPIO3 étant les broches UART matérielles par défaut du ESP32 (également utilisées pour la programmation/console série par USB), la sortie des journaux (logs) par UART matériel a été désactivée (`logger: baud_rate: 0`) afin d'éviter tout conflit. Les journaux restent disponibles via WiFi/API, comme c'est l'usage courant avec ESPHome.

Seule l'infrastructure de base (UART + trame Modbus) est préparée à ce stade. La configuration des registres spécifiques à un appareil Modbus donné (adresse, type de donnée, échelle) devra être ajoutée au moment de connecter un appareil réel — chaque appareil Modbus RTU a sa propre carte de registres, propre au fabricant, qu'il n'est pas possible de préparer à l'avance de façon générique.

## Broches GPIO libres

En plus des fonctions déjà câblées par la carte, plusieurs broches du module ESP32 sont amenées à un bornier libre sur la ES32D26, sans fonction prédéfinie : **GPIO4, GPIO5, GPIO16, GPIO17, GPIO18 et GPIO19**.

Particularités à connaître pour chacune :

| Broche | Particularité |
|---|---|
| GPIO4 | Seule broche du groupe capable de lecture analogique (ADC2, canal 0) — soumise à la même limitation WiFi/ADC2 que Vi1 et Vi3 (voir [Entrées analogiques](#entrées-analogiques)) si utilisée comme entrée analogique. |
| GPIO5, GPIO18, GPIO19 | Correspondent par défaut au bus SPI matériel (VSPI) du ESP32, mais peuvent être réaffectées à d'autres fonctions grâce à la matrice de broches flexible du ESP32. |
| GPIO16, GPIO17 | Broches généralistes, libres sur un module ESP32-WROOM standard — à éviter sur un module de variante WROVER, où elles sont réservées à la mémoire PSRAM embarquée. |

### Exemples d'utilisation possible

Ces broches n'ont aucune fonction prédéfinie par la carte ou par ce firmware — elles sont disponibles pour des ajouts futurs, par exemple :

- **Bus I2C** (2 fils, SDA/SCL) pour ajouter un capteur externe (température/humidité, luminosité, etc.), ou un convertisseur analogique-numérique externe (comme un module ADS1115) afin de contourner entièrement la limitation ADC2/WiFi mentionnée pour Vi1 et Vi3.
- **Sortie numérique additionnelle** : relais externe, indicateur lumineux, avertisseur sonore.
- **Entrée numérique additionnelle** : bouton, interrupteur, capteur de proximité.
- **Bus SPI** (via GPIO5/18/19) pour un périphérique externe, par exemple un écran ou un lecteur de carte SD.
- **Port UART supplémentaire** (UART1 ou UART2 logiciel) si une seconde liaison série est nécessaire en parallèle du RS485.

## Diagnostics WiFi

Deux capteurs de diagnostic exposent la qualité du signal WiFi : la puissance brute en dBm, et un pourcentage de qualité calculé (formule standard : 0% à -100dBm ou moins, 100% à -50dBm ou plus, linéaire entre les deux).

## Sécurité et secrets

Ce firmware utilise les composants standards `api:` (avec chiffrement) et `ota:` (avec mot de passe) d'ESPHome, ainsi qu'un point d'accès de secours WiFi protégé par mot de passe. **Les identifiants et clés ne doivent jamais être commités en clair dans un dépôt public.** Il est fortement recommandé d'utiliser le mécanisme `secrets.yaml` d'ESPHome pour toutes les valeurs sensibles (clé de chiffrement API, mot de passe OTA, identifiants WiFi, mot de passe du point d'accès de secours), et de s'assurer que ce fichier est exclu du contrôle de version (`.gitignore`).

## Problèmes connus et travaux en cours

- **Calibration Ii1 sous 4mA** : la correction à 3 points appliquée à Ii1 reste imprécise pour les valeurs de courant inférieures à environ 4mA. Ceci a une importance limitée en pratique, puisqu'un appareil 4-20mA réel n'envoie normalement jamais un courant sous 4mA en fonctionnement normal (cette plage indique généralement une défaillance de câblage ou de capteur plutôt qu'une valeur mesurée valide) — mais la calibration exacte de cette zone est encore en cours d'ajustement.
- **Ii2, Ii3, Ii4** : n'ont pas encore reçu la même correction de calibration qu'Ii1 (échelle 4-20mA + calibration à 3 points). À faire une fois la méthode validée sur Ii1.
- **Vi3** : la calibration à 2/3 points n'a pas encore été effectuée sur ce canal (seul Vi1 a été calibré empiriquement jusqu'à présent).
- **Limitation ADC2/WiFi (Vi1, Vi3)** : non résolue de façon permanente par nature (limitation matérielle du ESP32) — contournée via la mesure ponctuelle par bouton, mais toute lecture continue automatique reste impossible tant que le WiFi est actif.
- **Clignotement des relais au démarrage** : contourné par une stratégie logicielle (pré-validation du firmware avant installation), sans correction matérielle appliquée à ce jour.
- **Tension "zone interdite" sur les entrées numériques** : une anomalie électrique a été observée sur les entrées parallèles du registre 74HC165 (tension au repos dans la zone ambiguë CMOS plutôt qu'un niveau logique franc). L'origine exacte (réseau de résistances de tirage) n'a pas été formellement confirmée; cette observation reste à valider si des lectures peu fiables sont constatées sur les entrées numériques.
- **RS485/Modbus** : infrastructure de base fonctionnelle, mais aucun appareil Modbus spécifique n'a encore été configuré ou testé.
