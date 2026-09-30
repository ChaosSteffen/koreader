# Wi-Fi-Restore auf PocketBook – Befundbericht

Gerät: PocketBook Era Lite (PB710), Firmware U710.6.11.2124, KOReader v2026.07.2.
Messzeitraum: 30.09.2026, 13:58 bis 15:21 CEST.

## Kurzfassung

- `hasWifiRestore` lässt sich auf PocketBook **ohne Shell-Skript und ohne Blockieren der UI** umsetzen.
  Dafür nutzt man `NetConnectAsync()` aus `libinkview.so`. Nach dem Aufwachen ist das WLAN nach etwa 2,5 s wieder da.
- Das WLAN geht auf PocketBook nicht nur im Pause-Modus verloren. Es verschwindet auch in den **kurzen Kernel-Suspends
  zwischen zwei Seiten**, sobald kein Keepalive läuft. Von diesen Suspends erfährt KOReader nichts.
- Umsetzung: Branch `claude/pocketbook-wifi-restore-dada2a`, zwei Commits (siehe unten).
  Der Prototyp wurde am Gerät getestet, die finale Fassung noch nicht (siehe „Offen“).

## Was gemessen wurde

| Methode | Zweck |
|---|---|
| `strings` auf `netagent`, `nm -D`/`objdump -d` auf `libinkview.so` (lokal, Kopien der Binaries) | Welche APIs gibt es, wie verhält sich `NetConnectAsync` |
| KOReader-Debug-Logging (`PB event => …` in `ffi/input_pocketbook.lua`) | Event-Reihenfolge bei Pause, App-Wechsel, Ausschalten |
| Mess-Schleife am Gerät (`wifi_test/probe.sh`, alle 5 s: uptime, operstate `eth0`, Default-Route, Link-Qualität, Anzahl netagent-Prozesse) | WLAN-Zustand, auch wenn SSH nicht erreichbar ist. Lücken in den Zeitstempeln = Gerät schläft |
| `dmesg` (`PM: suspend entry/exit … UTC`) | Bestätigung echter Kernel-Suspends |
| Prototyp in `frontend/device/pocketbook/device.lua` mit Zeitmessungen (Log-Präfix `PBWR:`) | Dauer bis `NET_CONNECTED` / Default-Route, Blockierzeit |

## Befunde

### netagent und inkview

- `netagent` liegt unter `/ebrmain/cramfs/bin/root/netagent` (setuid root). `/ebrmain/bin/netagent` und `/usr/bin/netagent` sind Symlinks.
  Unterbefehle laut Binary: `wifi on|off`, `net on`, `connect`, `connect_silent`, `essid_silent`, `disconnect`, `wscan`, `wscanx`,
  `ping`, `net info`, `flightmode`, `bt on`. getopt-Optionen: `-s ssid -k key -p path -f -n`.
- Ein dauerhaft laufendes `netagent net on` (gestartet von `/usr/sbin/netmgr.sh`) überwacht das Netz. Es enthält einen
  Leerlauf-Timer (`run_idle_network_timer`, liest `/proc/net/tcp` und `/proc/net/dev`).
- Die inkview-Netzfunktionen rufen netagent selbst auf: `WiFiPower(1)` wird zu `netagent wifi on`, `NetConnect()` zu `netagent connect`,
  `NetMgrPing()` zu `netagent ping` (jeweils im Log belegt). `NetDisconnect()` wird vermutlich zu `netagent disconnect`
  (zeitlich korreliert, nicht direkt belegt). Ein eigener Shell-Weg über netagent bringt
  also nichts, was inkview nicht auch kann.
- `libinkview.so` exportiert unter anderem `NetConnectAsync`, `NetConnectSilent`, `NetConnect2`, `NetDisconnectAsync`,
  `GetNetState`, `GetWiFiPowerStatus` und `GetLastNetConnectionError`. KOReaders `ffi/inkview_h.lua` deklariert davon keine.
  Signatur laut SDK-Header: `int NetConnectAsync(int (*cb)(int status))`.
- Disassembliert: `hw_net_connect_async` startet per `pthread_create` einen abgekoppelten Thread und gibt sofort 0 zurück
  (bzw. −18 `NET_ETHREAD`, wenn `pthread_create` scheitert). Der Thread prüft den Callback an allen drei Aufrufstellen
  auf NULL. **`NetConnectAsync(nil)` ist daher sicher**: Es gibt keinen Rücksprung in den Lua-State aus einem fremden Thread.
- Der Thread bricht mit `NET_ABORTED` ab, wenn `hw_get_keylock()` oder `hw_shutting_down()` gesetzt ist.
  Dasselbe gilt für das blockierende `NetConnect` (im Log: `(hw_net_connect)connecting aborted`).

### Events (Debug-Log)

| Aktion | inkview-Events | KOReader |
|---|---|---|
| Pause (Power doppelt drücken) | `EVT_BACKGROUND` (par1 = PID), beim Aufwachen `EVT_UPDATE` → `EVT_FOREGROUND` | Suspend / Resume |
| Bibliothek und zurück | identisch zur Pause | Suspend / Resume, **nicht unterscheidbar** |
| Power lange drücken (Ausschalten) | nur `EVT_BACKGROUND`, dann wird der Prozess beendet | **kein `EVT_EXIT`, kein `Close`**. Einstellungen werden nur durch `flushSettings()` im Hintergrund-Handler gesichert |
| KOReader beendet sich selbst (z. B. Neustart) | `EVT_HIDE` → `EVT_EXIT`, *nach* dem Teardown | nur ein Echo |
| Automatisches Ausschalten | noch nicht gemessen | – |

`EVT_HIDE`/`EVT_SHOW` kommen nur beim Start und beim Beenden vor, nicht bei Pause oder App-Wechsel.

### Wann das WLAN verschwindet

- **Kernel-Suspend im Leerlauf:** Der Kernel hat `autosleep`/`wake_lock` (Android-Stil). KOReader erlaubt mit
  `UIManager:allowStandby()` → `PocketBook:setAutoStandby(true)` → `iv_sleepmode(1)`, dass das System bei Untätigkeit
  suspendiert. Das passiert **ohne jedes Event an KOReader**. Nach dem Aufwachen schläft das Gerät oft schon nach etwa 3 s wieder
  ein (in `dmesg` fünf Zyklen zwischen 14:46:09 und 14:48:07, während KOReader im Vordergrund war).
- Messung ohne Keepalive: SSH um 14:35:12 gestoppt → 14:35:45 bis 14:36:11 schläft das Gerät, danach ist `eth0` verschwunden,
  *bevor* die Pause ausgelöst wurde.
- Mit Keepalive (`NetMgrPing()` alle 30 s) schläft das Gerät im Leerlauf nicht, und das WLAN bleibt (Messungen 15:02–15:04
  und 15:15–15:16). Der Preis ist ein höherer Akkuverbrauch, solange das WLAN an ist.
- Die Pause beendet das WLAN immer (15:04:18 und 15:16:57 `noif`, obwohl der Keepalive lief).
- **Keepalive startete bisher nur nach KOReaders eigenem `turnOnWifi`.** Hat das System das WLAN hochgefahren (z. B. beim Booten),
  lief kein Keepalive, und die Verbindung war nach der ersten Leerlauf-Phase weg.

### Prototyp-Messungen

| Variante | Blockiert UI | Bis `NET_CONNECTED` + Default-Route |
|---|---|---|
| A: `WiFiPower(1)` + `NetConnectAsync(nil)` (15:09:10) | `WiFiPower(1)` 1058 ms, `NetConnectAsync` 0 ms | 2077 ms |
| B: nur `NetConnectAsync(nil)` (15:19:48) | 0 ms | 2555 ms (`QueryNetwork` 0x3 → 0x203) |

- Variante B braucht `WiFiPower(1)` nicht. Der Worker-Thread schaltet das WLAN selbst ein, obwohl der Treiber vorher
  entladen war (`noif`). Der eingebaute Fallback (`WiFiPower(1)` nach 5 s) wurde nicht ausgelöst.
- Ohne `UIManager:preventStandby()` würde das Gerät während des Verbindungsaufbaus wieder einschlafen (siehe oben).
  Mit dem Hold gab es zwischen Aufwachen und Verbindung keinen Suspend.
- App-Wechsel (Bibliothek → KOReader, 15:20:35): `restoreWifiAsync` erkennt das bestehende WLAN (`QueryNetwork = 515`)
  und frischt nur den Keepalive auf.
- `NetworkConnected` wird nach dem Restore über den generischen `connectivityCheck` gesendet. KOSync und andere Plugins
  sehen es also.

## Umsetzung (Branch)

1. `pocketbook: support restoring Wi-Fi on resume`
   - `ffi.cdef` für `NetConnectAsync` über `pcall`. `hasWifiRestore` ist nur `yes`, wenn das Symbol auflösbar ist
     (ältere Firmwares fallen damit automatisch auf das bisherige Verhalten zurück). PB613 ohne WLAN-Toggle: `no`.
   - `restoreWifiAsync`: Ist das WLAN schon an, frischt es nur den Keepalive auf. Sonst `NetConnectAsync(nil)`,
     `UIManager:preventStandby()` und alle 0,5 s eine Abfrage (bis 30 s). Bei Erfolg startet der Keepalive,
     in jedem Fall folgt danach `allowStandby()`.
   - `turnOffWifi` bricht einen laufenden Restore ab (gibt den Standby-Hold frei).
2. `pocketbook: keep Wi-Fi alive when it was already up at startup`
   - Startet den Keepalive auch, wenn das WLAN beim Start von KOReader schon verbunden ist.
   - Bewusst eigener Commit: Er verlängert die Wachphasen auch für Nutzer, die das WLAN nicht über KOReader eingeschaltet haben.

Für einen PR an koreader/koreader: beide Commits ohne diesen Bericht. Sauberer wäre es, `NetConnectAsync` zusätzlich
in koreader-base (`ffi/inkview_h.lua`) zu deklarieren. Das `pcall`-cdef hier funktioniert aber auch ohne diese Änderung.

## Offen

- **Finale Fassung am Gerät testen.** Getestet sind die Prototypen A und B, noch nicht die bereinigte Fassung aus dem Branch.
- **Automatisches Ausschalten** (`EVT_EXIT` vom Framework): Event-Reihenfolge noch nicht gemessen.
- **Restore beim Start** (`NetworkMgr:init` mit `wifi_was_on` und WLAN aus): nicht gezielt getestet.
- **Andere Modelle/Firmwares:** nur PB710 / 6.11 getestet. Ab welcher Firmware `NetConnectAsync` existiert, ist unbekannt.
  Das Symbol-Probing sorgt aber dafür, dass nichts kaputtgeht.
- **Doppelte `NetworkConnected`-Events:** Der generische `NetworkListener:onResume` plant bei jedem Resume einen
  `connectivityCheck`, auf PocketBook also auch bei jedem App-Wechsel. Das ist harmlos, aber redundant.
- **`wifi_was_on` wird bei einem gescheiterten Restore gelöscht** (generisches `_abortWifiConnection` nach 45 s).
  Danach stellt KOReader bis zum nächsten manuellen Einschalten nichts mehr wieder her.
- **Leerlauf-Suspends zwischen Seiten** lösen keinen Restore aus (kein Event). Dort greift weiterhin `beforeWifiAction`
  mit dem blockierenden `NetConnect` (~1–2 s), sofern der Keepalive nicht läuft.
- `pkill -TERM restore-wifi-async.sh` in `NetworkMgr:_abortWifiConnection`/`disableWifi` läuft auf PocketBook ins Leere
  (kein `pkill`, kein Skript). Das ist harmlos.

## Nebenbefunde

- **Toter Code:** Der `NetworkMgr:init`-Override in `PocketBook:initNetworkManager` (`waitForRoute`, #15244) wird nie
  ausgeführt. `initNetworkManager` wird *aus* dem laufenden `NetworkMgr:init()` aufgerufen und ersetzt die Methode erst,
  nachdem sie schon aufgelöst wurde.
- **Hardcover-Plugin:** Der Sync beim Suspend ruft `WiFiPower(1)` und `NetConnect` nach `EVT_BACKGROUND` auf. `NetConnect` scheitert
  an der Tastensperre („connecting aborted“, 14:36:13). Trotzdem kam der Sync um 14:36:15 laut Plugin mit echter API-Antwort durch.
  Plausibel ist, dass `netagent wifi on` (aus `WiFiPower(1)`) allein verbunden hat: netagent startet dabei `udhcpc`, und wpa_supplicant
  läuft ständig mit der gespeicherten Konfiguration. Die Mess-Schleife hat das Fenster zwischen 14:36:11 und 14:36:16 nicht erfasst.
  Verlässlich ist dieser Weg nicht. Das Plugin ist inzwischen auf das KOSync-Muster umgestellt (Warteschlange, Abarbeiten bei `NetworkConnected`).
- Die Meldung „Synchronisierung mit der PocketBook Cloud“ blieb im Lauf ohne Keepalive etwa 3 Minuten stehen
  und verzögerte den Suspend bis 14:39:02. Im Lauf mit Keepalive (15:04) verschwand sie nach kurzer Zeit.
  Die Ursache ist nicht geklärt.
- `scp -O` zum Gerät brach bei `device.lua` wiederholt mit „Connection closed by remote host“ ab. Übertragen wurde deshalb per
  `ssh … 'cat > datei.new'` mit md5-Abgleich vor dem `mv`.

## Am Gerät hinterlassen

- `/mnt/ext1/applications/koreader/wifi_restore_backup/device.lua` – Original (md5 `8099c0e6fa2751a51d380d759a3c40c8`)
- `/mnt/ext1/applications/koreader/frontend/device/pocketbook/device.lua` – derzeit Prototyp B (md5 `2c52f93d72830ed375931dfd127b398e`)
- `/mnt/ext1/applications/koreader/wifi_test/` – Mess-Schleife (`probe.sh`, `probe.log`, `probe.pid`). Die Schleife läuft bis zum nächsten Neustart.
