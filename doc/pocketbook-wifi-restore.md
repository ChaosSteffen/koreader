# Wi-Fi-Restore auf PocketBook – Befundbericht

Gerät: PocketBook Era Lite (PB710), Firmware U710.6.11.2124, KOReader v2026.07.2.
Messzeitraum: 30.09.2026, 13:58 bis 21:38 CEST.

## Kurzfassung

- `hasWifiRestore` lässt sich auf PocketBook **ohne Shell-Skript und ohne Blockieren der UI** umsetzen, mit `NetConnectAsync()`
  aus `libinkview.so`. Nach dem Aufwachen ist das WLAN nach etwa 1,5–2,5 s wieder da.
- Das WLAN geht auf PocketBook nicht nur im Pause-Modus verloren. Es verschwindet auch in den **kurzen Kernel-Suspends
  zwischen zwei Seiten**, sobald kein Keepalive läuft. Von diesen Suspends erfährt KOReader nichts.
- **Pause, Ausschalten per Knopf und automatisches Ausschalten** liefern KOReader alle nur `EVT_BACKGROUND` (→ Suspend).
  Das ist der letzte Moment für einen Sync. Dann ist die Tastensperre schon aktiv, und `NetConnect`/`NetConnectAsync` werden
  verweigert. **`NetConnectSilent` funktioniert trotzdem.** PocketBooks eigene Cloud-Sync (`pbcloud_sync.app`) nutzt es genauso.
- Das System baut das WLAN **5–10 s nach `EVT_BACKGROUND`** ab, unabhängig von `BanSleep` und `NetMgrPing`.
- Umsetzung: zwei PRs im Fork (siehe „Umsetzung“), am Gerät getestet.

## Was gemessen wurde

| Methode | Zweck |
|---|---|
| `strings`, `nm -D`, `objdump -d` auf Kopien von `netagent`, `libinkview.so`, `pbcloud_sync.app` u. a. (lokal) | Welche APIs gibt es, was tun sie wirklich |
| Header-Sammlung [koreader/pocketbook-inkview](https://github.com/koreader/pocketbook-inkview) | Signaturen und ab welcher Firmware eine Funktion existiert |
| KOReader-Debug-Logging (`PB event => …` in `ffi/input_pocketbook.lua`) | Event-Reihenfolge bei Pause, App-Wechsel, Ausschalten |
| Mess-Schleife am Gerät (`wifi_test/probe.sh`, alle 5 s: uptime, operstate `eth0`, Default-Route, Link-Qualität, Anzahl netagent-Prozesse) | WLAN-Zustand, auch ohne SSH. Lücken in den Zeitstempeln = Gerät schläft |
| `dmesg` (`PM: suspend entry/exit … UTC`) | Bestätigung echter Kernel-Suspends |
| Prototypen in `frontend/device/pocketbook/device.lua` mit Zeitmessungen | Dauer bis `NET_CONNECTED` / Default-Route, Blockierzeit |
| `netagent disconnect` (zeitverzögert per Skript) | WLAN vor einem Test gezielt trennen |

## Befunde

### netagent und inkview

- `netagent` liegt unter `/ebrmain/cramfs/bin/root/netagent` (setuid root). `/ebrmain/bin/netagent` und `/usr/bin/netagent` sind Symlinks.
  Unterbefehle laut Binary: `wifi on|off`, `net on`, `connect`, `connect_silent`, `essid_silent`, `disconnect`, `wscan`, `wscanx`,
  `ping`, `net info`, `flightmode`, `bt on`. getopt-Optionen: `-s ssid -k key -p path -f -n`.
- Ein dauerhaft laufendes `netagent net on` (gestartet von `/usr/sbin/netmgr.sh`, 5 s nach dem Booten) überwacht das Netz.
- Die inkview-Netzfunktionen rufen netagent auf (Literale in `libinkview.so` aufgelöst, teils im Log belegt):
  `WiFiPower(1)` → `netagent wifi on`, `NetConnect()` → `netagent connect`, `NetConnectSilent()` → `netagent connect_silent`,
  `NetMgrPing()` → `netagent ping`.
- `hw_net_connect(name, silent, hourglass)` ist der gemeinsame Kern von `NetConnect`, `NetConnect2` und `NetConnectSilent`:
  - Tastensperre (`hw_get_keylock`) → Abbruch mit `NET_ABORTED`, **nur wenn `silent == 0`**.
  - Herunterfahren → Abbruch.
  - `BanSleep(5)`.
  - Flugmodus → −39. Nicht still: Dialog `@TurnOffFlightModeAndLaunch`.
  - Netzwerkmanager aus (`NetMgrStatus() <= 0`) → −12. Nicht still: Dialog `@NeedInternet … @TurnOnWiFi`.
  - `system("netagent connect[_silent]")`, bei `hourglass` mit Sanduhr.
- `NetConnectAsync(cb)` startet einen abgekoppelten `pthread` und kehrt sofort zurück. Der Thread prüft die Tastensperre
  **immer**. Bei ausgeschaltetem Netzwerkmanager zeigt er den Dialog `/ebrmain/bin/dialog 5 @WantToConnect @WantToConnectMessage @No @Yes`
  („Mit dem Netzwerk verbinden, um die Arbeit mit der Anwendung fortzusetzen?“). Danach ruft er `netagent connect` auf und dann `cb(status)`.
  Der Callback ist an allen Stellen gegen NULL geprüft. **`NetConnectAsync(nil)` ist daher sicher**: Kein Rücksprung in den Lua-State
  aus einem fremden Thread.
- `NetMgrStatus()` liest ein Feld im Shared Memory (vermutlich die PID des netagent-Daemons): −1 = kein Netzgerät, 0 = aus, 1 = läuft.
  `GetNetState()` und `GetLastNetConnectionError()` lesen ebenfalls nur Shared-Memory-Felder (`netstate`, `last_connection_error`).
  Welche Werte dort bei Fehlschlägen stehen, ist nicht gemessen.
- Verfügbarkeit laut Headern: `NetConnectAsync`, `NetConnectSilent`, `NetMgrStatus`, `NetMgrPing` ab 5.8, `BanSleep` ab 5.19,
  `iv_sleep_preventor_*` ab 6.08 (dort schon als veraltet markiert zugunsten von `sleep_preventor.h`), `PostponeTimedPoweroff` in 6.11.
  KOReaders `ffi/inkview_h.lua` deklariert davon nur `NetMgrPing`.

### Events (Debug-Log)

| Aktion | inkview-Events | KOReader |
|---|---|---|
| Pause (Power doppelt drücken) | `EVT_BACKGROUND` (par1 = PID), beim Aufwachen `EVT_UPDATE` → `EVT_FOREGROUND` | Suspend / Resume |
| Bibliothek und zurück | identisch zur Pause | Suspend / Resume, **nicht unterscheidbar** |
| Power lange drücken (Ausschalten) | nur `EVT_BACKGROUND`, danach wird der Prozess beendet (noch ≥ 6 s Log) | **kein `EVT_EXIT`, kein `Close`** |
| Automatisches Ausschalten (hier exakt 20:00 min nach der letzten Eingabe) | nur `EVT_BACKGROUND`, danach beendet (noch ≥ 3 s Log) | **kein `EVT_EXIT`, kein `Close`** |
| KOReader beendet sich selbst (z. B. Neustart) | `EVT_HIDE` → `EVT_EXIT`, *nach* dem Teardown | nur ein Echo |

- `EVT_HIDE`/`EVT_SHOW` kommen nur beim Start und beim Beenden vor. Einstellungen werden bei Ausschalten nur durch `flushSettings()`
  im Hintergrund-Handler gesichert.
- **Der Power-Knopf erreicht KOReader nie.** In keinem Log gibt es ein `EVT_KEYPRESS` vor `EVT_BACKGROUND`, nur die Tasten 23–25.
  Der Knopf ist `axp22-powerkey` (`/dev/input/event2`). Alle `/dev/input/event*` gehören `root:root` mit Rechten `0660`, und KOReader
  läuft als `reader`. Abfangen (`EVIOCGRAB`) ist ohne root also nicht möglich. Die Tasten verarbeitet `monitor.app` (läuft als root).
- Der Keepalive verhindert das automatische Ausschalten nicht (20 Minuten lang Pings, trotzdem abgeschaltet).

### Wann das WLAN verschwindet

- **Kernel-Suspend im Leerlauf:** Der Kernel hat `autosleep`/`wake_lock` (Android-Stil). KOReader erlaubt mit
  `UIManager:allowStandby()` → `PocketBook:setAutoStandby(true)` → `iv_sleepmode(1)`, dass das System bei Untätigkeit
  suspendiert. Das passiert **ohne jedes Event an KOReader**. Nach dem Aufwachen schläft das Gerät oft schon nach etwa 3 s wieder
  ein (in `dmesg` fünf Zyklen zwischen 14:46:09 und 14:48:07, während KOReader im Vordergrund war).
- Ohne Keepalive ist das WLAN nach der ersten Leerlauf-Phase weg (SSH um 14:35:12 gestoppt, 14:35:45–14:36:11 Suspend, danach kein `eth0`).
- Mit Keepalive (`NetMgrPing()` alle 30 s) schläft das Gerät im Leerlauf nicht, und das WLAN bleibt. Der Preis ist ein höherer
  Akkuverbrauch, solange das WLAN an ist.
- **Nach `EVT_BACKGROUND` baut das System das WLAN nach 5–10 s ab.** Das gilt in allen Messungen (15:04, 15:16, 21:25, 21:30, 21:37),
  auch wenn gerade gepingt wurde (21:25:30, 15:04:14), auch mit `BanSleep(10)` (verzögert nur den Kernel-Suspend auf ~12 s nach der Pause)
  und auch nach frischem Verbinden mit `NetConnectSilent`.

### PocketBooks eigene Cloud-Sync

- `pbcloud_sync.app` (18 KB, `/ebrmain/bin`) importiert `NetConnectSilent`, `BanSleep`, `PostponeTimedPoweroff`, `NetMgrPing`,
  `NetMgrStatus` und `QueryNetwork`. Sie sperrt sich über `/tmp/pb_cloud_sync_lock` gegen Doppelstarts.
- Gestartet wird sie von `monitor.app` (root, der Prozess `./pocketbook`), Funktion `sync_cloud`: Sie fragt über
  `/ebrmain/lib/libpbcloud_api.so` `PBCloud_IsLoggedIn`/`PBCloud_IsAutoSyncronize` ab und startet dann `/ebrmain/bin/pbcloud_sync.app`.
  Die benachbarten Strings (`task_to_background`) deuten auf das Auslösen beim Wechsel in den Hintergrund hin, nicht verifiziert.
  `taskmgr.app` beendet sie bei Bedarf mit `killall -9`. Beim Herunterfahren ruft `monitor.app` `netagent disconnect` auf.
- `monitor.app` prüft auch `/mnt/ext1/system/bin/bookshelf.app`. Das deutet auf ersetzbare System-Apps im beschreibbaren Bereich hin,
  ist aber kein Hook für das Einschlafen. Nicht weiter untersucht.
- Beim Einschlafen kommt etwa 6 s nach `EVT_BACKGROUND` das Event `EVT_READ_PROGRESS_CHANGED`, dann erscheint
  „Synchronisierung mit der PocketBook Cloud“.
- Um 14:36 (WLAN beim Einschlafen weg) stand die Meldung etwa 3 Minuten, und die Mess-Schleife zeigte die ganze Zeit kein `eth0`.
  Bei stehendem WLAN verschwand sie nach kurzer Zeit. Eine Stichprobe, davor hatte das Hardcover-Plugin `netagent disconnect` aufgerufen.
- **Hooks:** Einziger gefundener Hook ist `/ebrmain/share/netscript.d/*.sh` (von `netscript.sh` bei connect/disconnect aufgerufen).
  Das ist kein Hook für das Einschlafen, und `/ebrmain` ist Firmware (cramfs, schreibgeschützt). Das Gerät ist nicht gerootet.

### Messungen

**Restore nach dem Aufwachen:**

| Variante | Blockiert UI | Bis `NET_CONNECTED` + Default-Route |
|---|---|---|
| `WiFiPower(1)` + `NetConnectAsync(nil)` (15:09:10) | `WiFiPower(1)` 1058 ms, `NetConnectAsync` 0 ms | 2077 ms |
| nur `NetConnectAsync(nil)` (15:19:48) | 0 ms | 2555 ms (`QueryNetwork` 0x3 → 0x203) |
| finale Fassung (15:57:25, 21:26:37, 21:31:41, 21:38:26) | 0 ms | 1,5–2 s |

- `WiFiPower(1)` ist nicht nötig. Der Worker-Thread schaltet das WLAN selbst ein, obwohl der Treiber vorher entladen war (`noif`).
- Ohne `UIManager:preventStandby()` würde das Gerät während des Verbindungsaufbaus wieder einschlafen.
- App-Wechsel: `restoreWifiAsync` erkennt das bestehende WLAN (`QueryNetwork = 0x203`) und frischt nur den Keepalive auf.

**Kaltstart:**
- Ohne Wartebedingung (16:22:52): `NetConnectAsync` direkt nach dem Booten → Systemdialog „Mit dem Netzwerk verbinden …“.
- Mit Wartebedingung (21:13:22): `NetMgrStatus: 0` → gewartet → 21:13:29 hatte das System selbst verbunden → kein Dialog.

**Senden beim Einschlafen:**

| Test | WLAN vorher | Ablauf nach `EVT_BACKGROUND` | Ergebnis |
|---|---|---|---|
| A (21:25:30) | an (Keepalive) | Hardcover sendet nach 3 s. WLAN um :35 noch da, um :40 weg. Suspend :43 (`BanSleep(10)` aktiv) | ✔ |
| B (21:30:26) | per `netagent disconnect` getrennt | `NetConnectSilent` → verbunden nach 2243 ms. WLAN um :31 da, um :36 weg | ✔ Verbindung (Plugins hatten nichts zu senden) |
| B2 (21:37:19) | getrennt, vorher umgeblättert | `NetConnectSilent` 2086 ms → pbcloudsync gesendet (+1 s) → Hardcover gesendet (+2 s). WLAN um :24 da, um :29 weg | ✔ beide Plugins |

## Umsetzung (PRs im Fork, noch nicht upstream)

**PR 1: `claude/pocketbook-wifi-restore-pr` → `master`**
1. `pocketbook: support restoring Wi-Fi on resume`
   - `ffi.cdef` für `NetConnectAsync` und `NetMgrStatus` über `pcall`. `hasWifiRestore` ist nur `yes`, wenn beide auflösbar sind.
     PB613 ohne WLAN-Toggle: `no`.
   - `restoreWifiAsync`:
     - Ist das WLAN schon an, frischt es nur den Keepalive auf.
     - Sonst: `preventStandby()` und warten, bis `NetMgrStatus() > 0` (höchstens 15 s, sonst still aufgeben).
     - Hat das System dann schon verbunden, nur abwarten. Sonst `NetConnectAsync(nil)`.
     - Verbindung abfragen (alle 0,5 s, bis 30 s), dann Keepalive starten und `allowStandby()`.
   - `turnOffWifi` bricht einen laufenden Restore ab.
2. `pocketbook: keep Wi-Fi alive when it was already up at startup`
   - Bewusst eigener Commit: Er verlängert die Wachphasen auch für Nutzer, die das WLAN nicht über KOReader eingeschaltet haben.

**PR 2: `claude/pocketbook-wifi-before-suspend` → PR 1** (gestapelt)
3. `pocketbook: bring Wi-Fi up before plugins get Suspend`
   - Im Suspend-Handler, *vor* dem Broadcast an die Plugins: Ist das WLAN aus, aber `wifi_was_on` und `auto_restore_wifi` gesetzt,
     `NetConnectSilent(nil)` (blockiert ~2 s, Oberfläche ist nicht sichtbar).
   - Kein `BanSleep`: Es hält nur den Kernel-Suspend auf, nicht das WLAN.

Sauberer wäre es, die neuen Funktionen in koreader-base (`ffi/inkview_h.lua`) zu deklarieren. Die `pcall`-cdefs funktionieren aber auch ohne das.

## Offen

- **Ausschalten mit WLAN aus:** Verbinden (~2 s) plus Senden muss in die 3–6 s passen, die KOReader nach `EVT_BACKGROUND` noch hat. Nicht getestet.
- **Andere Modelle/Firmwares:** nur PB710 / 6.11 getestet. Das Symbol-Probing sorgt dafür, dass nichts kaputtgeht.
- **Zeitfenster beim Einschlafen:** Nach dem Verbinden bleiben 3–8 s für *alle* Plugins zusammen. Langsame TLS-Handshakes (pbcloudsync
  berichtet von > 10 s an manchen Tagen) passen nicht zuverlässig hinein. Die Warteschlange als Rückfallebene bleibt nötig.
- **Asynchrones `turnOnWifi`:** `beforeWifiAction` nutzt auf PocketBook weiter das blockierende `NetConnect` (1–2 s). Umstellbar auf
  `NetConnectAsync`, das Framework ist darauf ausgelegt (vgl. Android). Offen ist, wie man Fehlschläge ohne Callback früh erkennt
  (`GetNetState()`?). Eigener PR.
- **Doppelte `NetworkConnected`-Events:** Der generische `NetworkListener:onResume` plant bei jedem Resume einen `connectivityCheck`,
  auf PocketBook also auch bei jedem App-Wechsel. Harmlos, aber redundant.
- **`wifi_was_on` wird bei einem gescheiterten Restore gelöscht** (generisches `_abortWifiConnection` nach 45 s). Danach stellt KOReader
  bis zum nächsten manuellen Einschalten nichts mehr wieder her.
- **„WLAN bei Inaktivität trennen“** fehlt auf PocketBook nur, weil `getNetworkInterfaceName()` nichts liefert. Mit `eth0` wäre das ein
  Ausgleich für den Akkuverbrauch des Keepalives. Nicht getestet.
- `pkill -TERM restore-wifi-async.sh` in `_abortWifiConnection`/`disableWifi` läuft auf PocketBook ins Leere. Das ist harmlos.

## Nebenbefunde

- **Toter Code:** Der `NetworkMgr:init`-Override in `PocketBook:initNetworkManager` (`waitForRoute`, #15244) wird nie ausgeführt.
  `initNetworkManager` wird *aus* dem laufenden `NetworkMgr:init()` aufgerufen und ersetzt die Methode erst, nachdem sie schon
  aufgelöst wurde. Dafür läuft eine eigene Session.
- **Hardcover-Plugin:** Der frühere Sync beim Suspend (`WiFiPower(1)` + `NetConnect`) kam um 14:36:15 durch, obwohl `NetConnect` an der
  Tastensperre scheiterte. Plausibel ist, dass `netagent wifi on` allein verbunden hat (startet `udhcpc`, wpa_supplicant läuft ständig).
  Nicht verlässlich. Das Plugin ist inzwischen auf das KOSync-Muster umgestellt (Warteschlange, Abarbeiten bei `NetworkConnected`).
- **pbcloudsync-Plugin:** `main.lua:305` rief ein erst später deklariertes `format_percent` auf (21:24:04). In dessen Repo behoben (`1fb6f30`).
- `scp -O` zum Gerät brach bei `device.lua` wiederholt mit „Connection closed by remote host“ ab. Übertragen wurde deshalb per
  `ssh … 'cat > datei.new'` mit md5-Abgleich vor dem `mv`.

## Am Gerät hinterlassen

- `/mnt/ext1/applications/koreader/wifi_restore_backup/device.lua` – Original (md5 `8099c0e6fa2751a51d380d759a3c40c8`)
- `/mnt/ext1/applications/koreader/frontend/device/pocketbook/device.lua` – finaler Stand aus PR 2 (Commit `fe3991362`), angewendet auf die
  installierte v2026.07.2 (md5 `10e7358b04e882741c308252056a5620`)
- `/mnt/ext1/applications/koreader/wifi_test/` – Mess-Schleife (`probe.sh`, `probe.log`, `probe.pid`, `testb.log`). Die Schleife läuft bis zum nächsten Neustart.
