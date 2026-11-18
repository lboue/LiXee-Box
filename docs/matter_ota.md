# OTA Matter (cluster OTA Software Update)

Etat des lieux du cluster Matter `OTA Software Update Requestor`/`Provider` sur le pont, et ce
qu'il faudrait faire pour l'activer reellement. A ne pas confondre avec le systeme de mise a jour
"maison" (HTTP + auto-update), seul mecanisme operationnel aujourd'hui : voir
[maintenance_api.md](maintenance_api.md).

## Etat actuel : active dans la config, jamais cable

`sdkconfig.defaults:121-122` active `CONFIG_ENABLE_OTA_REQUESTOR=y`, ce qui remonte dans
`sdkconfig` :
- `CONFIG_SUPPORT_OTA_SOFTWARE_UPDATE_PROVIDER_CLUSTER=y` et
  `..._REQUESTOR_CLUSTER=y` (`sdkconfig:3045-3046`) : support compile dans le SDK `esp-matter`.
- `CONFIG_ESP_MATTER_OTA_PROVIDER_ENABLED` **non defini** (`sdkconfig:3118-3121`) : pas de
  Provider Matter local.
- `CONFIG_CHIP_OTA_IMAGE_BUILD` **non defini** (`sdkconfig:2863-2866`) : pas de generation
  d'image `.ota` au build.

Ce flag active seulement la *compilation* du support OTA dans `managed_components/espressif__esp_matter`.
Pour qu'un endpoint `OTA Software Update Requestor` existe reellement dans l'arbre Matter expose
(et donc soit visible par un controleur comme Home Assistant ou Apple Home), l'application doit
appeler explicitement, apres `esp_matter::start()` :
- `esp_matter_ota_requestor_init()` : cree le cluster Requestor + le cluster client Provider sur
  le root node.
- `esp_matter_ota_requestor_start()` : instancie `DefaultOTARequestor`, le `BDXDownloader` et
  l'`OTAImageProcessorImpl`.

(`managed_components/espressif__esp_matter/components/esp_matter/esp_matter_ota.h:58-63`)

**Aucun de ces deux appels n'existe dans le code du projet.** `matter_bridge_start()`
(`components/matter_bridge/matter_bridge.cpp:1153-1166`) appelle `esp_matter::start()` ligne 1162
puis enchaine directement sur la configuration WiFi/commissioning, sans jamais toucher a l'OTA
requestor. Consequence : le cluster n'apparait pas dans le device Matter du pont, meme si le
firmware est bien compile avec le support OTA.

## Ce qui gere reellement les mises a jour aujourd'hui

Le systeme "maintenance" (`components/maintenance`), independant de Matter : upload manuel d'une
image `.bin`/archive `.tar` via l'API web, ou mise a jour automatique par URL (GitHub Releases).
Partitions `ota_0`/`ota_1` classiques ESP-IDF (`esp_ota_*`), sans rapport avec le cluster Matter.
Detail complet : [maintenance_api.md](maintenance_api.md).

| | Systeme maison (actuel) | Matter OTA Requestor (non implemente) |
|---|---|---|
| Declencheur | Upload web ou tache auto (URL GitHub) | Controleur Matter (query/annonce Provider) |
| Transport | HTTP(S) | BDX (BDX over Matter, CASE) |
| Image | `firmware.bin` brut (en-tete ESP32 verifie) | `.ota` (en-tete CHIP : VID/PID/version + payload) |
| Partition cible | `ota_0`/`ota_1` (`esp_ota_*`) | memes partitions, via `OTAImageProcessorImpl` |
| Authenticite | TLS + URL configuree (pas de Secure Boot) | depend du Provider ; pas de Secure Boot non plus |

## Pour l'activer : guide d'implementation

1. **Cabler l'init/start.** Dans `matter_bridge.cpp`, apres le `if (ret != ESP_OK) {...} else {`
   de la ligne 1165 (une fois `s_matter_started = true`), ajouter :
   ```cpp
   esp_matter_ota_requestor_init();
   esp_matter_ota_requestor_start();
   ```
   Inclure `esp_matter_ota.h`. Ces appels doivent intervenir apres `esp_matter::start()` (deja le
   cas ici) ; `esp_matter_ota_requestor_set_config()` est optionnel, a appeler dans la foulee pour
   ajuster `periodic_query_timeout`/`watchdog_timeout` ou fournir un
   `esp_matter_ota_requestor_impl_t` custom (driver/consentement utilisateur/processeur d'image).
   Par defaut, l'`ExtendedOTARequestorDriver` du SDK suffit.

2. **Fournir un Provider.** `CONFIG_ESP_MATTER_OTA_PROVIDER_ENABLED` n'est pas active : le pont
   ne sert pas lui-meme d'image. Le Requestor doit interroger un Provider externe :
   - en test : `chip-ota-provider-app` (outil CLI du SDK connectedhomeip) sur le meme reseau,
     commissionne separement, avec l'image `.ota` a servir ;
   - en usage reel : le controleur de l'ecosysteme (Apple Home, Google Home, Home Assistant avec
     un addon Provider) doit exposer un Provider et en informer le Requestor
     (`AnnounceOTAProvider` ou decouverte par le controleur).

3. **Generer l'image `.ota`.** Avec `CONFIG_CHIP_OTA_IMAGE_BUILD` non active, rien n'est produit
   automatiquement. Il faut construire l'en-tete avec `chip-ota-image-tool` (outil du SDK
   connectedhomeip, pas present dans ce repo) a partir du binaire d'application
   (`build/LiXee-Box.bin` ou equivalent) et des identifiants deja definis :
   - VID/PID : `CONFIG_DEVICE_VENDOR_ID=0xFFF1` / `CONFIG_DEVICE_PRODUCT_ID=0x8000`
     (`sdkconfig:2787-2788`).
   - Version logicielle : `CONFIG_DEVICE_SOFTWARE_VERSION_NUMBER` (`sdkconfig:2790`,
     `sdkconfig.defaults:165`), tenu synchronise avec `PROJECT_VER` (`CMakeLists.txt:25-26`,
     actuellement `"3.2"` -> `302`). A incrementer a chaque release, cote `.ota` comme cote
     `esp_app_desc_t`.

4. **Image processor.** Par defaut, `OTAImageProcessorImpl` du SDK ecrit directement dans la
   partition OTA inactive via les API `esp_ota_*` standard (mêmes partitions `ota_0`/`ota_1` que
   le systeme maison) — aucun changement de `partitions/partitions.csv` necessaire. Le basculement
   de partition (`esp_ota_set_boot_partition`) et le anti-rollback
   (`esp_ota_mark_app_valid_cancel_rollback`, deja gere au boot pour le systeme maison dans
   `components/maintenance/maintenance.cpp:571-576`) s'appliquent de la meme facon.

5. **Suivi applicatif.** Aucun callback de progression custom n'existe aujourd'hui pour Matter
   (contrairement au systeme maison, `components/maintenance/maintenance.cpp:338-393`). Pour
   exposer un etat (page web, logs) il faut fournir un `esp_matter_ota_requestor_impl_t::driver`
   custom ou s'abonner aux evenements Matter d'etat OTA (`OTARequestorStateChange`, etc. via
   `matter_app_event_cb`).

## Points a trancher avant implementation

- **Coexistence des deux systemes.** Si le Matter OTA Requestor est active en plus du systeme
  maison, les deux ecrivent dans les memes partitions `ota_0`/`ota_1` : pas de conflit technique
  tant qu'une seule mise a jour tourne a la fois (le systeme maison verifie deja `busy` dans
  `/api/maint/status`), mais rien ne les synchronise aujourd'hui (versions annoncees, verrou
  commun). A prevoir si les deux chemins doivent cohabiter.
- **Pas de Secure Boot.** Comme pour le systeme maison, l'authenticite d'une image Matter ne
  reposerait que sur la confiance accordee au Provider (pas de verification cryptographique de
  l'image elle-meme cote firmware).
- **Provider a heberger.** Sans Provider (local ou tiers), activer le Requestor ne sert a rien en
  pratique : la decouverte echouera ou restera sans reponse. A clarifier qui hebergerait le
  Provider pour les utilisateurs finaux (pas de canal de publication Matter actuellement, comme
  note pour `updateUrl` dans `maintenance_api.md:62`).
