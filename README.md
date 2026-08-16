# Hack télécommande Somfy pure IO

## Introduction
Home Assistant ne permet pas d'émuler les télécommandes Somfy IO. La solution est de passer par des boitiers Somfy type Tahoma, connexonn io ou équivalents qui représentent un budget non négligeable mais qui restent la meilleure solution lorsque l'on a plusieurs appareils Somfy car ils permettent en outre de récupérer l'état réel de chaque appareil. 
Lorsque l'on a un ou deux appareils Somfy IO et le reste en RTS comme moi, il devient interessant de hacker une telecommande Somfy IO pour la rendre pilotable par Home Assistant, ceci avec un faible budget mais à condition de savoir souder. 
Il existe déjà des [montages sur Youtube](https://www.youtube.com/watch?v=AbRFvk2iZs8&t=708s) qui expliquent comment faire. Le montage que je propose est dérivé de celui ci avec quelques améliorations sur un modèle de télécommande plus récent : Somfy Situo 1 Pure IO.

## Principe
La télécommande possède 3 boutons pour envoyer les 3 signaux : UP, MY, DOWN. Sur le circuit imprimé de la técommande, chaque bouton se présente sous forme d'un cercle entouré d'un anneau. l'appui sur un bouton met en contact le cercle et l'anneau, ce qui correspond à une mise à la masse de l'anneau. Ce dernier est normalement à 3,3v.

Il faut donc simplement mettre a la masse un des 3 anneaux pour déclencher une des commandes (UP,MY, DOWN). On peut le faire de plusieurs manières soit avec des relais commandés par HA ou plus simplement avec un circuit de type ESP32 ou Wemo D1 programmé avec ESPHome. C'est la solution proposée dans la vidéo citée précedemment qui utilise un ESP32. L'auteur a soudé un fil sur chacun des trois anneaux relié à 3 sorties GPIO de l'ESP32 et cela marche très bien. On effectue alors la commande à partir de HA.
![Icon](https://github.com/loudemer/hack-somfy-io/blob/main/Images/cablage%20telecommande.png)

Etant prudent, je voulais pouvoir toujours utiliser la técommande au cas où il y aurait une panne sur HA. J'ai donc cherché les points correspondants aux 3 anneaux sur l'autre face du circuit imprimé pour y souder les 3 fils de commande et pouvoir ainsi garder les boutons de la técommande opérationnels.
J'ai utilisé un Wemo D1 pour la commande, c'est largement suffisant. 

Il y a alors 2 problèmes à résoudre :
 1. lorsqu'on utilise les boutons de la télécommande, on met a la masse l'output du GPIO du Wemo, ce qui peut l'endommager. Il faut donc isoler la sortie du wemo en interposant un transistor selon le schéma suivant (une diode ne fonctionne pas).
 2. HA doit être informé de l'appui du bouton lorsqu'on utilise la télécommande, donc il faut renvoyer l'information sur 3 GPIO du wemo configurés en Input.

L'alimentation de la télécommande se fait par le Wemo car la consommation est très basse. Il ne faut donc pas de pile dans la telecommande.

## Réalisation
Le montage du wemo et de son interface avec les transistors se fait au mieux sur un circuit imprimé. Un PCB est fourni en exemple pour sa réalisation. 
Les composants sont :
1. 3 transistors NPN S8050 ou BC547
2. 4 résistances 1/4 w 10K
3. 1 wemo D1
Il faut souder 5 fils sur la télécommande +, -, UP, DOWN et MY comme indiqué sur la photo et les relier aux 5 sorties du circuit imprimé. 
On peut mettre une prise sur le cablage pour pouvoir détacher le circuit si besoin.
Le circuit peut etre mis dans un boitier imprimé en 3D. Le fichier STL est joint.
Le boitier est solidarisé au dos de la télécommande avec un adhésif double face.

## Intégration ESPHome
Il faut flasher le wemo avec les paramètres suivants :
```yaml
# Board: Wemos D1 Mini (ESP8266)
# Hack télécommande Somfy Situo 1 IO Pure
#
# Sorties (simulation appuis boutons via transistors NPN S8050/BC547) :
#   GPIO5  (D1) → Résistance 10kΩ → Base transistor UP
#   GPIO4  (D2) → Résistance 10kΩ → Base transistor DOWN
#   GPIO14 (D5) → Résistance 10kΩ → Base transistor MY/Stop
#   Collecteur transistor → Pastille bouton cercle extérieur
#   Émetteur transistor → GND (pastille cercle intérieur)
#   inverted: false car le transistor NPN inverse déjà le signal
#
# Entrées (lecture appuis boutons physiques) :
#   Pastille → GPIO INPUT_PULLUP directement (sans diode)
#   Filtre anti-rebond 50ms uniquement
#   Flag global ha_command_active pour ignorer les échos des commandes HA
#   (au lieu de delayed_on 600ms qui nécessitait un appui trop long)
#
# Alimentation télécommande :
#   3V3 Wemos → Batterie + télécommande
#   GND Wemos → Batterie - télécommande

esphome:
  name: store-banne-io
  friendly_name: Store banne IO
  on_boot:
    priority: 600
    then:
      - switch.turn_off: store_banne_io_up
      - switch.turn_off: store_banne_io_down
      - switch.turn_off: store_banne_io_stop

esp8266:
  board: d1_mini

logger:

api:
  encryption:
    key: !secret store_banne_io__encryption_key

ota:
  - platform: esphome
    password: !secret store_banne_io__ota_password

wifi:
  min_auth_mode: WPA2
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: Store banne IO Fallback Hotspot
    password: "xxxxxxxxx"
  manual_ip:
    static_ip: 192.168.0.xx
    gateway: 192.168.0.1
    subnet: 255.255.255.0

# ---------------------------------------------------------------------------
# FLAG GLOBAL — Indique qu'une commande HA est en cours
# Permet aux binary_sensor d'ignorer les échos des commandes HA
# sans avoir besoin d'un delayed_on long
# ---------------------------------------------------------------------------

globals:
  - id: ha_command_active
    type: bool
    restore_value: false
    initial_value: 'false'

# ---------------------------------------------------------------------------
# SORTIES — Simulation appuis boutons via transistors NPN
# Le flag ha_command_active est mis à true pendant l'impulsion
# pour que les binary_sensor ignorent l'écho
# ---------------------------------------------------------------------------

switch:
  - platform: gpio
    id: store_banne_io_up
    name: "Bouton UP"
    entity_category: diagnostic
    icon: "mdi:arrow-up-bold"
    restore_mode: ALWAYS_OFF
    pin:
      number: GPIO5
      inverted: false
      mode:
        output: true
    interlock: [store_banne_io_stop, store_banne_io_down]
    on_turn_on:
      - globals.set:
          id: ha_command_active
          value: 'true'
      - delay: 300ms
      - switch.turn_off: store_banne_io_up
      - globals.set:
          id: ha_command_active
          value: 'false'

  - platform: gpio
    id: store_banne_io_down
    name: "Bouton DOWN"
    entity_category: diagnostic
    icon: "mdi:arrow-down-bold"
    restore_mode: ALWAYS_OFF
    pin:
      number: GPIO4
      inverted: false
      mode:
        output: true
    interlock: [store_banne_io_stop, store_banne_io_up]
    on_turn_on:
      - globals.set:
          id: ha_command_active
          value: 'true'
      - delay: 300ms
      - switch.turn_off: store_banne_io_down
      - globals.set:
          id: ha_command_active
          value: 'false'

  - platform: gpio
    id: store_banne_io_stop
    name: "Bouton Stop"
    entity_category: diagnostic
    icon: "mdi:square"
    restore_mode: ALWAYS_OFF
    pin:
      number: GPIO14
      inverted: false
      mode:
        output: true
    interlock: [store_banne_io_down, store_banne_io_up]
    on_turn_on:
      - globals.set:
          id: ha_command_active
          value: 'true'
      - delay: 300ms
      - switch.turn_off: store_banne_io_stop
      - globals.set:
          id: ha_command_active
          value: 'false'

# ---------------------------------------------------------------------------
# ENTREES — Lecture appuis boutons physiques
# Filtre anti-rebond 50ms uniquement — appui normal (~200ms) suffit
# Le flag ha_command_active filtre les échos des commandes HA
# La télécommande envoie directement la commande radio au store
# → on met à jour uniquement l'état HA via cover.template.publish
# ---------------------------------------------------------------------------

binary_sensor:
  - platform: gpio
    id: phys_up
    name: "Appui physique UP"
    entity_category: diagnostic
    pin:
      number: GPIO12 # D6
      mode:
        input: true
        pullup: true
      inverted: true
    filters:
      - delayed_on: 50ms
    on_press:
      then:
        - if:
            condition:
              lambda: 'return !id(ha_command_active);'
            then:
              - cover.template.publish:
                  id: store_banne_ouest
                  state: OPEN
                  current_operation: OPENING
              - logger.log: "Appui physique UP → état cover mis à jour OPEN"

  - platform: gpio
    id: phys_down
    name: "Appui physique DOWN"
    entity_category: diagnostic
    pin:
      number: GPIO2 # D4
      mode:
        input: true
        pullup: true
      inverted: true
    filters:
      - delayed_on: 50ms
    on_press:
      then:
        - if:
            condition:
              lambda: 'return !id(ha_command_active);'
            then:
              - cover.template.publish:
                  id: store_banne_ouest
                  state: CLOSED
                  current_operation: CLOSING
              - logger.log: "Appui physique DOWN → état cover mis à jour CLOSED"

  - platform: gpio
    id: phys_my
    name: "Appui physique Stop/My"
    entity_category: diagnostic
    pin:
      number: GPIO16 # D0
      mode:
        input: true
        # pas de pullup interne sur GPIO16 — résistance externe 10kΩ requise
      inverted: true
    filters:
      - delayed_on: 50ms
    on_press:
      then:
        - if:
            condition:
              lambda: 'return !id(ha_command_active);'
            then:
              - cover.template.publish:
                  id: store_banne_ouest
                  state: OPEN
                  current_operation: IDLE
              - logger.log: "Appui physique MY → état cover mis à jour IDLE"

# ---------------------------------------------------------------------------
# ENTITE COVER — cover: template + optimistic: true
# ---------------------------------------------------------------------------

cover:
  - platform: template
    id: store_banne_ouest
    name: "Store banne Ouest"
    optimistic: true
    icon: "mdi:awning-outline"
    device_class: awning                          # a modifier en fonction du device
    open_action:
      - switch.turn_on: store_banne_io_up
    close_action:
      - switch.turn_on: store_banne_io_down
    stop_action:
      - switch.turn_on: store_banne_io_stop
```
## Intégration dans HA
On a alors une entité cover qui permet de commander son appareil

## Limites 
On ne récupère pas l'état réel de son appareil. On récupère seulement la commande effectuée.
