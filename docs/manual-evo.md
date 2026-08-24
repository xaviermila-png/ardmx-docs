# Manual d'usuari — ARDMX EVO

<!-- nota: firmware repo c:\ARDMX\ardmx4-evo-firmware, src/main.cpp. Text de
     versió que respon el dispositiu (V62): "ARDMX EVO v2.0". Equivalent
     funcional de l'ARDMX4 (Arduino Mega) amb un cor ESP32: mateix repertori
     de funcions (4 escenes+transicions, cicle sincronitzat amb música),
     migrat de Bluetooth Classic (Mega) a BLE. -->

## Introducció

<!-- nota: diferència clau amb l'ARDMX One v2 — l'EVO té reproducció
     d'àudio MP3 sincronitzada amb el cicle (DFPlayer Mini) i un mode
     "Manual" (Trigger extern, pin físic) que l'One no té. La resta del
     model (4 escenes, 4 transicions PER CANAL, tipus Lineal/Salt/Ease
     In/Ease Out) és idèntic — mateix disseny, portat deliberadament amb
     els mateixos noms de funció als dos firmwares. -->

## Contingut de la caixa / requisits

<!-- nota tècnica maquinari (capçalera de main.cpp):
     - ESP32 DevKit V1
     - Mòdul MAX485 amb direcció automàtica
     - Mòdul DFPlayer Mini (reproductor MP3 des de targeta microSD)
     - Univers DMX512 fins a 510 canals (MAX_CANALS=510, no 512 — marge
       fins al múltiple de 32 de CHANNEL_BUFFER_SIZE)
     - App ARDMX (Android, BLE) — mateixa app que l'ARDMX One -->

## Instal·lació física

<!-- nota tècnica pins (main.cpp, "Maquinari"):
     - DMX: MAX485 a GPIO22=TX, GPIO21=RX (no usat), direcció automàtica
     - DFPlayer Mini per UART1: GPIO17 (TX ESP32, amb resistència sèrie
       ~1kΩ) -> RX DFPlayer; TX DFPlayer -> divisor de tensió (1kΩ+2kΩ)
       -> GPIO16 (RX ESP32). Alimentat a 5V.
     - LED d'estat: GPIO2 (resistència 220-330Ω en sèrie)
     - Trigger extern (mode Manual, EstatSelector==2): GPIO4, INPUT_PULLUP
       — connectar un polsador entre GPIO4 i GND.
     - Targeta microSD del DFPlayer: fitxers MP3 numerats segons la
       convenció del DFPlayer (p.ex. carpeta 01/001.mp3...) — confirmar
       la convenció exacta amb Reproductor/GestioDFPlayer al codi abans
       de redactar aquesta secció. -->

## Primera connexió (BLE)

<!-- nota tècnica protocol BLE — IDÈNTIC a l'ARDMX One a propòsit (mateix
     transport, mateixos UUIDs, el producte es distingeix pel handshake
     V64 un cop connectat, no pel transport):
     - Nom Bluetooth per defecte: "ARDMXEVO" (DEFAULT_BLUETOOTH_NAME),
       editable des de l'app (V63), màxim 15 caràcters.
     - UUIDs GATT (compartits amb l'ARDMX One):
         Servei:      74fdf89b-a063-48f4-837d-03462d2b3687
         Escriptura:  c7e05764-94cb-4a2f-8cd4-4751163c58ad
         Notificació: dd2a9ece-4964-4f42-b986-36719d38b2a3
     - PIN de connexió opcional (V73/74/75/76), mateix mecanisme que
       l'ARDMX One. -->

## Interfície de l'app: pantalles i navegació

<!-- nota tècnica pantalles reals (títols AppScaffold):
     - "Menú Principal" (ardmx_evo_main_menu_screen.dart) — selector
       principal: Escena 1-4 (fixa), Automàtic, MANUAL (Trigger extern —
       només a l'EVO), Configuració. A sota, barra de progrés del cicle
       (CycleProgressBar, visible en Automàtic/Manual) i gestió de volum
       ("Cançó: N" + slider de volum DFPlayer), ara ancorada a baix de
       tot de la pantalla (contingut de dalt amb scroll).
     - "Escena / Canals" (ardmx_evo_scene_channels_screen.dart) — igual
       que l'ARDMX One: navegador d'escena + grup de 3 canals +
       sliders/camp numèric + editor de transicions per canal.
     - "Programació Cicles" (ardmx_evo_cycle_programming_screen.dart) —
       durades acumulades dels 8 períodes + Play/Pausa + volum.
     - "Paràmetres" (ardmx_evo_parameters_screen.dart) — nom del
       pessebre, descripció, CANÇÓ A REPRODUIR (específic de l'EVO),
       nombre de canals gestionables, nombre d'escenes actives.
     - "Configuració" (ardmx_evo_system_config_screen.dart) — nom
       Bluetooth, PIN, exportació/importació, reset de fàbrica.
     - Igual que l'One: les 4 últimes només accessibles amb el selector
       en mode "Configuració" (V11=7). -->

## Gestió d'escenes

<!-- nota tècnica — idèntic a l'ARDMX One v2 (mateix disseny, mateix
     codi portat): 4 posicions d'escena, NumeroEscenes (V18, 1-4)
     configurable, navegació clampada a l'última escena activa. Amb
     NumeroEscenes==1: Automàtic, Manual i la pantalla Cicle queden
     desactivats (res entre què fer cicle). -->

## Gestió de transicions

<!-- nota tècnica — idèntic a l'ARDMX One v2: transicions PER CANAL
     (CanalData.transicions[4], no compartides), la transició "a"
     l'escena N és la de SORTIDA cap a la següent, 4 tipus mostrats en
     aquest ordre a l'app: Lineal, Fi suau, Inici suau, Salt (l'únic
     amb % editable). Diferència amb l'EVO respecte a l'One: el cicle
     Automàtic aquí pot anar sincronitzat amb la reproducció de música
     (GestioCicles/Reproductor). -->

## Edició de canals

<!-- nota tècnica — idèntic a l'ARDMX One v2: 3 sliders + camp numèric
     (0-255) per grup, navegació per grups de 3, nom per canal (fins a
     15 bytes). -->

## Gestió d'àudio (específic de l'EVO)

<!-- nota: secció que NO existeix al manual de l'One v2 — cal detallar
     aquí la selecció de cançó (V00/"Cançó a reproduir" a Paràmetres),
     el control de volum (V16, 0-30) i com queda sincronitzada la
     reproducció amb l'inici/fi del cicle automàtic (IniciarReproduccio/
     PararReproduccio/ContinuarReproduccio a main.cpp). -->

## Resolució de problemes freqüents

<!-- nota tècnica candidata (confirmar amb l'usuari quines calen):
     - "No sona la música": comprovar la targeta microSD ben inserida i
       amb els MP3 numerats correctament; el LED/log ja avisa
       "DFPlayer no disponible" a l'arrencada si no el detecta.
     - "No trobo el dispositiu en escanejar" / "Demana un PIN que no
       sé" / "Els canvis no es desen": mateixes respostes que l'ARDMX
       One v2 (mateix mecanisme de connexió i desat). -->

## Especificacions tècniques

<!-- nota tècnica (constants de main.cpp):
     - Univers: fins a 510 canals DMX512 (MAX_CANALS).
     - Freqüència d'enviament DMX: 40Hz (DMX_SEND_INTERVAL_MS=25ms).
     - 4 escenes, 4 transicions per canal (LINEAL/SALT/EASE_IN/EASE_OUT).
     - Àudio: DFPlayer Mini (MP3 des de microSD), volum 0-30.
     - Trigger extern per mode Manual (GPIO4).
     - Persistència: NVS ampliada a 64KB (partitions.csv, app0/app1
       "factory"/3MB — flaix més gran que l'One per marge d'aquest
       firmware, més pesat en codi per l'àudio).
     - Connexió: BLE/GATT, mateix disseny que l'ARDMX One v2. -->
