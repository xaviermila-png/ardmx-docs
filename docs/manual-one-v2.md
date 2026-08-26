# Manual d'usuari — ARDMX One v2

<!-- nota: firmware repo c:\ARDMX\ardmx-one-firmware, src/main.cpp. Text de versió
     que respon el dispositiu (V62 / TIndex.firmwareVersion): "ARDMX One v2.0".
     Handshake d'identificació (V64): tipus JSON "tipus":"ARDMX_ONE",
     "firmware":"2.0.0" — l'app distingeix v1 (unitats de camp, mai reflashejades)
     de v2 pel número MAJOR d'aquest camp semver, no pel "tipus" (idèntic a
     totes dues). Veure DeviceIdentificationService._parseMajorVersion a l'app. -->

## Introducció

<!-- nota: "Una única escena DMX estàtica" és l'ARDMX One v1 (encara desplegat
     en unitats reals, mai es reflashegen). Aquest manual és per a v2:
     4 escenes estàtiques + 4 transicions (una per parell consecutiu,
     cíclica 4->1), cadascuna amb tipus + durada pròpia. -->

## Contingut de la caixa / requisits

<!-- nota tècnica maquinari (capçalera de main.cpp):
     - ESP32 DevKit V1
     - Mòdul MAX485 amb direcció automàtica (sense pin DE/RE)
     - Univers DMX512 fins a 512 canals (MAX_DMX_CHANNEL), sempre arrodonit
       avall a múltiple de 3 (roundDownToMultipleOf3) perquè els 3 sliders
       de l'app puguin avançar per grups sencers.
     - App ARDMX (Android, Flutter) amb Bluetooth Low Energy (BLE) —
       NO Bluetooth Classic. Requereix Android amb suport BLE. -->

## Instal·lació física

<!-- nota tècnica pins (main.cpp, "Configuració de maquinari"):
     - DMX_TX_PIN = GPIO22 (sortida cap al MAX485)
     - DMX_RX_PIN = GPIO21 (no s'usa activament, l'ARDMX One només envia DMX)
     - DMX_RTS_PIN = -1 (el MAX485 té commutació de direcció automàtica,
       no cal pin DE/RE)
     - LED d'estat: GPIO2 (fix encès sense client BLE, parpelleja amb
       client connectat, LED_BLINK_MS=500) -->

## Primera connexió (BLE)

<!-- nota tècnica protocol BLE (capçalera de main.cpp, "Disseny GATT"):
     - Nom Bluetooth per defecte: "ARDMXOne" (DEFAULT_BLUETOOTH_NAME),
       editable des de l'app (V63), màxim 15 caràcters, només
       lletres/dígits/'_' (sanitizeName()) — canviar-lo reinicia l'ESP32.
     - UUIDs GATT (compartits amb l'EVO a propòsit, mateix transport,
       el producte es distingeix pel handshake V64):
         Servei:      74fdf89b-a063-48f4-837d-03462d2b3687
         Escriptura:  c7e05764-94cb-4a2f-8cd4-4751163c58ad
         Notificació: dd2a9ece-4964-4f42-b986-36719d38b2a3
     - L'app escaneja filtrant per aquest servei — troba ARDMX One i
       ARDMX EVO en el mateix escaneig (BleBluetoothTransport.serviceUuid).
     - PIN de connexió opcional (V73/74/75/76, PIN_LENGTH=4): si està
       activat, cap altra petició es respon fins enviar el PIN correcte
       per V73 — pantalla "PIN de connexió" a Configuració. -->

## Interfície de l'app: pantalles i navegació

<!-- nota tècnica pantalles reals (títols AppScaffold + rutes a app_router.dart):
     - "Menú Principal" (ardmx_one_v2_main_menu_screen.dart) — selector
       principal (dial): Escena 1-4 (fixa), Automàtic, Configuració.
       Sense "Manual" (Trigger) — aquest maquinari no té pin físic per a
       un disparador extern (DialSelector.showManual=false).
     - "Escena / Canals" (ardmx_one_v2_scene_channels_screen.dart) —
       navegador d'escena (SceneNavigator) + navegador de grup de 3
       canals (ChannelNumberBar) + sliders/camp numèric + editor de
       transicions.
     - "Programació Cicles" (ardmx_one_v2_cycle_programming_screen.dart)
       — durades acumulades dels 8 períodes del cicle (Escena/Transició
       alternades), Play/Pausa.
     - "Paràmetres" (ardmx_one_v2_parameters_screen.dart) — nom del
       pessebre, descripció, nombre de canals gestionables, nombre
       d'escenes actives.
     - "Configuració" (ardmx_one_v2_system_config_screen.dart) — nom
       Bluetooth, PIN de connexió, exportació/importació, reset de fàbrica.
     - "Simulació" (lib/features/simulacio/, compartida amb l'EVO) —
       visualitzador gràfic de les corbes del cicle, fins a 12 canals
       alhora, Play/Pausa/Stop i marcador de posició en directe. Primer
       botó del submenú de Configuració. Força pantalla horitzontal en
       obrir-se.
     - Aquestes pantalles només són accessibles amb el selector
       principal en mode "Configuració" (V11=7) — el firmware només hi
       reacciona (Escenes()/Cicle()/ConfiguracioParametres()) mentre
       EstatSelector==7. -->

## Gestió d'escenes

<!-- nota tècnica (ardmx-one-firmware/src/main.cpp):
     - Sempre exactament 4 posicions d'escena a la NVS, però només les
       "actives" (NumeroEscenes, V18, 1-4, configurable a Paràmetres) es
       poden seleccionar/reproduir — la navegació (SceneNavigator, fletxes
       V35) queda ara clampada a [1, NumeroEscenes], no permet passar de
       l'última activa.
     - Amb NumeroEscenes==1: el mode "Automàtic" i la pantalla "Cicle"
       queden desactivats a l'app (no hi ha res entre què fer cicle) —
       el firmware ja ho rebutjava internament (torna V11 a "Escena 1").
     - Escena activa = V09; cada canal desa el seu valor (0-255) per a
       cadascuna de les 4 escenes (CanalData.valors[4]). -->

## Gestió de transicions

<!-- nota tècnica clau (aquesta és la que més val la pena il·lustrar amb vídeo):
     - Cada CANAL DMX té les seves pròpies 4 transicions (tipus + %salt),
       NO compartides amb la resta de canals — un canal pot fer un SALT
       sobtat mentre un altre fa un LINEAL suau durant la mateixa
       transició d'escena (CanalData.transicions[4]).
     - La transició configurada "a" l'escena N és la de SORTIDA cap a la
       següent (transicions[i]: escena i+1 -> escena ((i+1)%4)+1, cíclic
       4->1) — l'editor de l'app ho titula literalment "Transició
       Escena N -> Escena M".
     - 4 tipus (enum TipusTransicio, ordre de mostra a l'app —
       _displayOrder a channel_transition_editor.dart — DIFERENT de
       l'ordre intern, no canviar-ho): Lineal, Fi suau (EASE_OUT),
       Inici suau (EASE_IN), Salt. Salt és l'únic amb un % editable
       (moment, dins la transició, en què el canal salta instantàniament).
     - Editor: una columna per canal visible (3), fletxes per navegar
       entre les 4 transicions — comença mostrant la de l'escena
       activa i es resincronitza en canviar-la (ChannelTransitionEditor). -->

## Edició de canals

<!-- nota tècnica (ChannelSliders / ChannelNumberBar):
     - 3 sliders visibles alhora (grup de 3 canals consecutius),
       navegables amb fletxes ("CANALS") — s'avança de 3 en 3
       (advanceChannelGroup), mai un canal solt.
     - Cada slider té slider + camp numèric editable en paral·lel (0-255,
       clamp, no rebuig), sincronitzats: arrossegar l'un actualitza
       l'altre. Seleccionar tot el número en tocar el camp per editar-lo
       més ràpid.
     - Cada canal pot tenir un NOM propi (fins a 15 bytes, V65-67),
       mostrat sobre el slider. -->

## Resolució de problemes freqüents

<!-- nota tècnica (per a preguntes de suport reals trobades durant el
     desenvolupament — confirmar amb l'usuari quines calen documentar):
     - "No trobo el dispositiu en escanejar": comprovar que és BLE (no
       Classic) i que el mòbil té Bluetooth/ubicació activats (Android ho
       requereix per escanejar BLE).
     - "Demana un PIN que no sé": pantalla Configuració -> PIN de
       connexió -> restablir (V75), o reset de fàbrica (V41/V42).
     - "Els canvis no es desen / es perden en reiniciar": desat amb
       debounce (SAVE_DEBOUNCE_MS=500ms d'inactivitat) — cal esperar
       mig segon sense tocar res abans de treure l'alimentació. -->

## Especificacions tècniques

<!-- nota tècnica (constants de main.cpp):
     - Univers: fins a 512 canals DMX512 (MAX_DMX_CHANNEL), sempre
       múltiple de 3.
     - Freqüència d'enviament DMX: 40Hz (DMX_SEND_INTERVAL_MS=25ms).
     - 4 escenes, 4 transicions per canal (LINEAL/SALT/EASE_IN/EASE_OUT).
     - Persistència: NVS (memòria no volàtil interna de l'ESP32),
       partició ampliada a 64KB (vegeu partitions.csv) per encabir els
       512 canals x (valors+transicions) + noms.
     - Connexió: Bluetooth Low Energy (BLE/GATT), MTU preferit 247 bytes.
     - Nom Bluetooth i PIN de connexió configurables des de l'app. -->
