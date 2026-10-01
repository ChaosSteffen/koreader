# Wi-Fi-Restore auf PocketBook – Befundbericht

Gerät: PocketBook Era Lite (PB710), Firmware U710.6.11.2124, KOReader v2026.07.2.
Messzeitraum: 30.09.2026, 13:58 CEST bis 01.10.2026, 09:47 CEST.

## Kurzfassung

- `hasWifiRestore` lässt sich auf PocketBook **ohne Shell-Skript und ohne Blockieren der UI** umsetzen, mit `NetConnectAsync()`
  aus `libinkview.so`. Nach dem Aufwachen ist das WLAN nach etwa 1,5–2,5 s wieder da.
- Das WLAN geht auf PocketBook nicht nur im Pause-Modus verloren. Es verschwindet auch in den **kurzen Kernel-Suspends
  zwischen zwei Seiten**, sobald kein Keepalive läuft. Von diesen Suspends erfährt KOReader nichts.
- **Pause, Ausschalten per Knopf und automatisches Ausschalten** liefern KOReader alle nur `EVT_BACKGROUND` (→ Suspend).
  Das ist der letzte Moment für einen Sync. Es gibt keinen Hook davor, keinen Zugriff auf die Power-Taste und kein anderes Event.
- **Pause:** Die Tastensperre ist aktiv, `NetConnect`/`NetConnectAsync` werden verweigert, **`NetConnectSilent` funktioniert**.
  Das System baut das WLAN 5–10 s nach `EVT_BACKGROUND` wieder ab.
- **Ausschalten:** `monitor.app` setzt ein „wird heruntergefahren“-Flag. Danach verweigern inkview *und* netagents `connect`-Befehl
  jede Verbindung, netagent schaltet das WLAN sogar wieder ab. **`WiFiPower(1)` (`netagent wifi on`) prüft das Flag nicht.**
  Danach verbindet sich das WLAN von selbst. KOReader lebt nach `EVT_BACKGROUND` noch 10–12 s, Verbinden plus Senden braucht ~4 s.
- Umsetzung: drei PRs im Fork (siehe „Umsetzung“), am Gerät getestet.

## Was gemessen wurde

| Methode | Zweck |
|---|---|
| `strings`, `nm -D`, `objdump -d` auf Kopien von `netagent`, `libinkview.so`, `monitor.app`, `pbcloud_sync.app` u. a. (lokal), String-Zugriffe per Skript aufgelöst | Welche APIs gibt es, was tun sie wirklich, wie läuft das Ausschalten ab |
| Header-Sammlung [koreader/pocketbook-inkview](https://github.com/koreader/pocketbook-inkview) | Signaturen und ab welcher Firmware eine Funktion existiert |
| KOReader-Debug-Logging (`PB event => …` in `ffi/input_pocketbook.lua`) | Event-Reihenfolge bei Pause, App-Wechsel, Ausschalten |
| Mess-Schleife am Gerät (`wifi_test/probe.sh`, alle 5 s: uptime, operstate `eth0`, Default-Route, Link-Qualität, Anzahl netagent-Prozesse) | WLAN-Zustand, auch ohne SSH. Lücken in den Zeitstempeln = Gerät schläft |
| Mess-Schleife fürs Ausschalten (`wifi_test/shutdown_probe.sh`, alle ~0,6 s: operstate, Route, relevante Prozesse, `sync` nach jeder Zeile) | Ablauf des Ausschaltens, übersteht den Stromverlust |
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
  `WiFiPower(1)` → `netagent wifi on` (ohne jede Prüfung), `NetConnect()` → `netagent connect`,
  `NetConnectSilent()` → `netagent connect_silent`, `NetMgrPing()` → `netagent ping`.
- `hw_net_connect(name, silent, hourglass)` ist der gemeinsame Kern von `NetConnect`, `NetConnect2` und `NetConnectSilent`:
  - Tastensperre (`hw_get_keylock`) → Abbruch mit `NET_ABORTED`, **nur wenn `silent == 0`**.
  - Herunterfahren (`hw_shutting_down`) → Abbruch mit `NET_ABORTED`, **in jedem Modus**.
  - `BanSleep(5)`.
  - Flugmodus → −39. Nicht still: Dialog `@TurnOffFlightModeAndLaunch`.
  - Netzwerkmanager aus (`NetMgrStatus() <= 0`) → −12. Nicht still: Dialog `@NeedInternet … @TurnOnWiFi`.
  - `system("netagent connect[_silent]")`, bei `hourglass` mit Sanduhr.
- **netagents `connect`/`connect_silent` selbst** (Funktion um `0x28f20`): `wifi on`, dann bis zu 80 × 100 ms auf `netstate == 2` warten.
  Die Schleife bricht ab, sobald Tastensperre (`shm[0x9]`) oder Herunterfahren (`shm[0x1]`) gesetzt ist. Danach gilt:
  Ist `shm[0x1]` gesetzt, **schaltet netagent das WLAN ab und endet mit Exit-Code 12**, auch wenn es schon verbunden war.
  Bei Tastensperre im stillen Modus wartet es stattdessen bis zu weitere 20 s auf die Verbindung. Deshalb funktioniert `NetConnectSilent` bei der Pause.
- `NetConnectAsync(cb)` startet einen abgekoppelten `pthread` und kehrt sofort zurück. Der Thread prüft die Tastensperre
  **immer**. Bei ausgeschaltetem Netzwerkmanager zeigt er den Dialog `/ebrmain/bin/dialog 5 @WantToConnect @WantToConnectMessage @No @Yes`
  („Mit dem Netzwerk verbinden, um die Arbeit mit der Anwendung fortzusetzen?“). Danach ruft er `netagent connect` auf und dann `cb(status)`.
  Der Callback ist an allen Stellen gegen NULL geprüft. **`NetConnectAsync(nil)` ist daher sicher**: Kein Rücksprung in den Lua-State
  aus einem fremden Thread.
- Shared Memory des Systems: Schlüssel `0x00a12302`, 34064 Bytes, root, Rechte `666`. `NetMgrStatus()` (−1 = kein Netzgerät,
  0 = aus, 1 = läuft), `GetNetState()`, `GetLastNetConnectionError()`, `hw_get_keylock()` und `hw_shutting_down()` lesen nur Felder daraus.
  `hw_shutting_down()` (liest `shm[0x1]`) ist exportiert, steht aber nicht im SDK-Header.
- Verfügbarkeit laut Headern: `NetConnectAsync`, `NetConnectSilent`, `NetMgrStatus`, `NetMgrPing` ab 5.8, `BanSleep` ab 5.19,
  `iv_sleep_preventor_*` ab 6.08 (dort schon als veraltet markiert zugunsten von `sleep_preventor.h`), `PostponeTimedPoweroff` in 6.11.
  KOReaders `ffi/inkview_h.lua` deklariert davon nur `NetMgrPing`.

### Events (Debug-Log)

| Aktion | inkview-Events | KOReader |
|---|---|---|
| Pause (Power doppelt drücken) | `EVT_BACKGROUND` (par1 = PID), beim Aufwachen `EVT_UPDATE` → `EVT_FOREGROUND` | Suspend / Resume |
| Bibliothek und zurück | identisch zur Pause | Suspend / Resume, **nicht unterscheidbar** |
| Power lange drücken (Ausschalten) | nur `EVT_BACKGROUND`, 10–12 s später beendet | **kein `EVT_EXIT`, kein `Close`** |
| Automatisches Ausschalten (hier exakt 20:00 min nach der letzten Eingabe) | nur `EVT_BACKGROUND`, danach beendet | **kein `EVT_EXIT`, kein `Close`** |
| KOReader beendet sich selbst (z. B. Neustart) | `EVT_HIDE` → `EVT_EXIT`, *nach* dem Teardown | nur ein Echo |

- `EVT_HIDE`/`EVT_SHOW` kommen nur beim Start und beim Beenden vor. Einstellungen werden bei Ausschalten nur durch `flushSettings()`
  im Hintergrund-Handler gesichert.
- `EVT_SAVESTATE` (155) existiert im SDK, kam aber in keinem Log vor. `EVT_POSTPONE_TIMED_POWEROFF` (217) kommt wenige Sekunden
  nach dem Verbinden, vermutlich von `pbcloud_sync.app` (`PostponeTimedPoweroff`).
- **Der Power-Knopf erreicht KOReader nie.** In keinem Log gibt es ein `EVT_KEYPRESS` vor `EVT_BACKGROUND`, nur die Tasten 23–25.
  Der Knopf ist `axp22-powerkey` (`/dev/input/event2`). Alle `/dev/input/event*` gehören `root:root` mit Rechten `0660`, und KOReader
  läuft als `reader`. Abfangen (`EVIOCGRAB`) ist ohne root also nicht möglich. Die Tasten verarbeitet `monitor.app` (läuft als root).
- Der Keepalive verhindert das automatische Ausschalten nicht (20 Minuten lang Pings, trotzdem abgeschaltet).

### Ausschalten (`monitor.app`)

Ablauf beim Ausschalten per Knopf (Funktion `0x1d378`, gestripptes PIE-Binary, Funktionsnamen aus den Log-Strings):

1. Flag „wird heruntergefahren“ setzen (`shm[0x1] = 1`, zusätzlich eine eigene statische Kopie). Bei Tastensperre zuerst `REQ_KEYUNLOCK`.
2. `power_off_logo.app -s` anzeigen. KOReader bekommt dabei `EVT_BACKGROUND`.
3. 2 s + 1 s warten (gegebenenfalls Ton, Boot-Logo-Konfiguration).
4. `shutdown_all_tasks(0)` (`0x2d594`):
   - Sind `PBCloud_IsLoggedIn()` und `PBCloud_IsAutoSyncronize()` wahr und ist die aktive App ein bestimmter Task-Eintrag
     (Bedingung nicht vollständig entschlüsselt), ruft es `/ebrmain/bin/netagent connect_silent` auf. Das scheitert wegen des Flags.
     Beobachtet nur, wenn das WLAN zu diesem Zeitpunkt noch nicht verbunden war.
   - Alle Apps bekommen **SIGINT** (`killpg(pgid, 2)`). Pro App wartet es bis zu 59 × 100 ms, dann folgt SIGKILL für die übrigen.
   - Dann `sync_cloud` (startet `pbcloud_sync.app`) und `netagent disconnect`.
5. `sync` in einem Kindprozess (bis ~3 s), `/etc/init.d/rc.shutdown` (falls vorhanden), `poweroff`.

Weitere Befunde:
- **PocketBooks eigene Sync beim Ausschalten funktioniert nur, wenn das WLAN in dem Moment schon steht.** Ihre eigenen Verbindungsversuche
  scheitern am selben Flag (beobachtet um 08:53:57 und 09:05:38).
- KOReader reagiert vermutlich nicht auf das SIGINT, solange es untätig in C wartet. Im Log steht kein „interrupted!“, und der Zeitpunkt des
  Endes passt zum SIGKILL nach der Wartezeit. Das SIGINT selbst wurde nicht direkt beobachtet.
- `TASK_NOFORCEDKILL` (1 << 6, setzbar über `SetTaskParameters`) wird in `shutdown_all_tasks` nicht ausgewertet.
- Keine nutzbaren Hooks: `/etc/init.d/rc.shutdown` (Systembereich), `/mnt/secure/runonce/*.sh` (beim Booten, für `reader` nicht beschreibbar),
  `/ebrmain/share/netscript.d/*.sh` (connect/disconnect, Firmware).
- Ein Eingriff in `shm[0x1]` wäre technisch möglich (Rechte `666`). Er wurde bewusst nicht versucht: systemweiter Zustand, undokumentiertes Layout.

### Wann das WLAN verschwindet

- **Kernel-Suspend im Leerlauf:** Der Kernel hat `autosleep`/`wake_lock` (Android-Stil). KOReader erlaubt mit
  `UIManager:allowStandby()` → `PocketBook:setAutoStandby(true)` → `iv_sleepmode(1)`, dass das System bei Untätigkeit
  suspendiert. Das passiert **ohne jedes Event an KOReader**. Nach dem Aufwachen schläft das Gerät oft schon nach etwa 3 s wieder
  ein (in `dmesg` fünf Zyklen zwischen 14:46:09 und 14:48:07, während KOReader im Vordergrund war).
- Ohne Keepalive ist das WLAN nach der ersten Leerlauf-Phase weg (SSH um 14:35:12 gestoppt, 14:35:45–14:36:11 Suspend, danach kein `eth0`).
- Mit Keepalive (`NetMgrPing()` alle 30 s) schläft das Gerät im Leerlauf nicht, und das WLAN bleibt. Der Preis ist ein höherer
  Akkuverbrauch, solange das WLAN an ist. Der Ping selbst sendet nichts: `tx_packets` auf `eth0` blieb über 3 Minuten mit 6 Pings unverändert.
- **Nach `EVT_BACKGROUND` (Pause) baut das System das WLAN nach 5–10 s ab.** Das gilt in allen Messungen (15:04, 15:16, 21:25, 21:30, 21:37),
  auch wenn gerade gepingt wurde (21:25:30, 15:04:14), auch mit `BanSleep(10)` (verzögert nur den Kernel-Suspend auf ~12 s nach der Pause)
  und auch nach frischem Verbinden mit `NetConnectSilent`.

### PocketBooks eigene Cloud-Sync

- `pbcloud_sync.app` (18 KB, `/ebrmain/bin`) importiert `NetConnectSilent`, `BanSleep`, `PostponeTimedPoweroff`, `NetMgrPing`,
  `NetMgrStatus` und `QueryNetwork`. Sie sperrt sich über `/tmp/pb_cloud_sync_lock` gegen Doppelstarts.
- Gestartet wird sie von `monitor.app` (Funktion `sync_cloud`), wenn `PBCloud_IsLoggedIn`/`PBCloud_IsAutoSyncronize` wahr sind.
  Beobachtet wurde sie außerdem kurz nach dem Verbinden, wer sie dann startet, ist offen. `taskmgr.app` beendet sie bei Bedarf mit `killall -9`.
- Beim Einschlafen kommt etwa 6 s nach `EVT_BACKGROUND` das Event `EVT_READ_PROGRESS_CHANGED`, dann erscheint
  „Synchronisierung mit der PocketBook Cloud“.
- Um 14:36 (WLAN beim Einschlafen weg) stand die Meldung etwa 3 Minuten, und die Mess-Schleife zeigte die ganze Zeit kein `eth0`.
  Bei stehendem WLAN verschwand sie nach kurzer Zeit.
- `monitor.app` prüft auch `/mnt/ext1/system/bin/bookshelf.app`. Das deutet auf ersetzbare System-Apps im beschreibbaren Bereich hin.
  Nicht weiter untersucht.

### Messungen

**Restore nach dem Aufwachen:**

| Variante | Blockiert UI | Bis `NET_CONNECTED` + Default-Route |
|---|---|---|
| `WiFiPower(1)` + `NetConnectAsync(nil)` (15:09:10) | `WiFiPower(1)` 1058 ms, `NetConnectAsync` 0 ms | 2077 ms |
| nur `NetConnectAsync(nil)` (15:19:48) | 0 ms | 2555 ms (`QueryNetwork` 0x3 → 0x203) |
| finale Fassung (15:57:25, 21:26:37, 21:31:41, 21:38:26) | 0 ms | 1,5–2 s |

- `WiFiPower(1)` ist dafür nicht nötig. Der Worker-Thread schaltet das WLAN selbst ein, obwohl der Treiber vorher entladen war (`noif`).
- Ohne `UIManager:preventStandby()` würde das Gerät während des Verbindungsaufbaus wieder einschlafen.

**Kaltstart:**
- Ohne Wartebedingung (16:22:52): `NetConnectAsync` direkt nach dem Booten → Systemdialog „Mit dem Netzwerk verbinden …“.
- Mit Wartebedingung (21:13:22): `NetMgrStatus: 0` → gewartet → 21:13:29 hatte das System selbst verbunden → kein Dialog.

**Senden beim Einschlafen:**

| Test | WLAN vorher | Ablauf nach `EVT_BACKGROUND` | Ergebnis |
|---|---|---|---|
| A (21:25:30) | an (Keepalive) | Hardcover sendet nach 3 s. WLAN um :35 noch da, um :40 weg. Suspend :43 (`BanSleep(10)` aktiv) | ✔ |
| B (21:30:26) | per `netagent disconnect` getrennt | `NetConnectSilent` → verbunden nach 2243 ms. WLAN um :31 da, um :36 weg | ✔ Verbindung (Plugins hatten nichts zu senden) |
| B2 (21:37:19) | getrennt, vorher umgeblättert | `NetConnectSilent` 2086 ms → pbcloudsync gesendet (+1 s) → Hardcover gesendet (+2 s). WLAN um :24 da, um :29 weg | ✔ beide Plugins |

**Senden beim Ausschalten (Power lange gedrückt, WLAN vorher aus):**

| Test | KOReader-Stand | Ablauf | Ergebnis |
|---|---|---|---|
| 08:53:53 | `NetConnectSilent` | −12 sofort. `monitor.app` versucht es um +4 s selbst (`connect_silent`, scheitert). KOReader endet um +13 s | ✘ |
| 09:05:35 | Prototyp: `netagent connect_silent` direkt | Exit-Code 12 nach 2629 ms, `eth0` kurz `down`, dann weg. `monitor.app` scheitert ebenso | ✘ |
| 09:46:50 | `WiFiPower(1)` + Warten | verbunden nach 2136 ms → pbcloudsync gesendet (+1 s) → Hardcover gesendet (+2 s). WLAN bis zum `netagent disconnect` um +18 s. `monitor.app` versuchte diesmal nicht selbst zu verbinden | ✔ beide Plugins |

## Umsetzung (PRs im Fork, noch nicht upstream)

**PR 1: `claude/pocketbook-wifi-restore-pr` → `master`** ([ChaosSteffen/koreader#1](https://github.com/ChaosSteffen/koreader/pull/1))
- `pocketbook: support restoring Wi-Fi on resume`
  - `ffi.cdef` für `NetConnectAsync` und `NetMgrStatus` über `pcall`. `hasWifiRestore` ist nur `yes`, wenn beide auflösbar sind.
    PB613 ohne WLAN-Toggle: `no`.
  - `restoreWifiAsync`:
    - Ist das WLAN schon an, passiert nichts.
    - Sonst: `preventStandby()` und warten, bis `NetMgrStatus() > 0` (höchstens 15 s, sonst still aufgeben).
    - Hat das System dann schon verbunden, nur abwarten. Sonst `NetConnectAsync(nil)`.
    - Verbindung abfragen (alle 0,5 s, bis 30 s), dann `allowStandby()`.
  - **Kein Keepalive** nach dem Restore: Nach dem Sync darf das System das WLAN im nächsten Leerlauf wieder abbauen.
  - `turnOffWifi` bricht einen laufenden Restore ab.

**PR 2: `claude/pocketbook-wifi-before-suspend` → PR 1** (gestapelt, [ChaosSteffen/koreader#2](https://github.com/ChaosSteffen/koreader/pull/2))
1. `pocketbook: bring Wi-Fi up before plugins get Suspend`: Im Suspend-Handler, *vor* dem Broadcast an die Plugins: Ist das WLAN aus,
   aber `wifi_was_on` und `auto_restore_wifi` gesetzt, `NetConnectSilent(nil)` (blockiert ~2 s, Oberfläche nicht sichtbar). Kein `BanSleep`.
2. `pocketbook: also bring Wi-Fi up before powering off`: Meldet `hw_shutting_down()` (per Symbol-Probing) ein Ausschalten,
   `WiFiPower(1)` statt `NetConnectSilent` und bis zu 6 s auf die Verbindung warten.

**PR 3: `claude/pocketbook-wifi-auto-disable` → `master`** ([ChaosSteffen/koreader#3](https://github.com/ChaosSteffen/koreader/pull/3))
- `pocketbook: allow disabling Wi-Fi when inactive`: `getNetworkInterfaceName()` liefert `"eth0"`. Damit steht „Disable Wi-Fi connection
  when inactive“ zur Verfügung. Es fängt den Keepalive ab, den KOReader nach dem Einschalten per Menü bzw. bei Bedarf startet.

Sauberer wäre es, die neuen Funktionen in koreader-base (`ffi/inkview_h.lua`) zu deklarieren. Die `pcall`-cdefs funktionieren aber auch ohne das.

## Offen

- **Automatisches Ausschalten mit der finalen Fassung:** nicht per Log ausgewertet. Im Alltag (01.10.) kam bei Pause, Ausschalten per Knopf
  und automatischem Ausschalten laut Steffen zuverlässig ein neuer Stand bei Hardcover an.
- **Wettlauf mit `monitor.app` beim Ausschalten:** Ist das WLAN ~3 s nach dem Ausschalt-Logo noch nicht verbunden, z. B. bei schwachem Signal,
  versucht `monitor.app` selbst zu verbinden. Dieser Versuch schaltet das WLAN wegen des Flags wieder ab. Das gilt nur, wenn die
  PocketBook-Cloud-Autosync an ist. Ohne Autosync entfällt der Versuch. pbcloudsync braucht die Autosync laut dessen Session nicht,
  außer dass neue Bücher nur über sie in die Cloud kommen. Nicht getestet (Test 2).
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
- **„WLAN bei Inaktivität trennen“ (PR 3)** hat in den Tests noch nicht ausgelöst. Die Prüfung wurde jedes Mal vor Ablauf der 5 Minuten
  durch Pause oder Neustart abgebrochen.
- `pkill -TERM restore-wifi-async.sh` in `_abortWifiConnection`/`disableWifi` läuft auf PocketBook ins Leere. Das ist harmlos.

## Nebenbefunde

- **Toter Code:** Der `NetworkMgr:init`-Override in `PocketBook:initNetworkManager` (`waitForRoute`, #15244) wird nie ausgeführt.
  `initNetworkManager` wird *aus* dem laufenden `NetworkMgr:init()` aufgerufen und ersetzt die Methode erst, nachdem sie schon
  aufgelöst wurde. Dafür läuft eine eigene Session.
- **Hardcover-Plugin:** Der frühere Sync beim Suspend (`WiFiPower(1)` + `NetConnect`) kam um 14:36:15 durch, obwohl `NetConnect` an der
  Tastensperre scheiterte. Erklärung: `netagent wifi on` allein hatte verbunden. Das Plugin ist inzwischen auf das KOSync-Muster umgestellt
  (Warteschlange, Abarbeiten bei `NetworkConnected`). Ein Fehler beim Abschalten nach dem Senden ist dort behoben.
- **pbcloudsync-Plugin:** `main.lua:305` rief ein erst später deklariertes `format_percent` auf (21:24:04). In dessen Repo behoben (`1fb6f30`).
- `scp -O` zum Gerät brach bei `device.lua` wiederholt mit „Connection closed by remote host“ ab. Übertragen wurde deshalb per
  `ssh … 'cat > datei.new'` mit md5-Abgleich vor dem `mv`.

## Am Gerät hinterlassen

- `/mnt/ext1/applications/koreader/wifi_restore_backup/device.lua` – Original (md5 `8099c0e6fa2751a51d380d759a3c40c8`)
- `/mnt/ext1/applications/koreader/frontend/device/pocketbook/device.lua` – finaler Stand aus PR 2 (Commit `489a707bd`) und PR 3
  (Commit `3731965d8`), angewendet auf die installierte v2026.07.2 (md5 `7303b4c0f74045db8f0f5ee21092c0ad`)
- `/mnt/ext1/applications/koreader/wifi_test/` – Mess-Skripte und Logs (`probe.sh`, `shutdown_probe.sh`, `shutdown*.log`, `tx.log`, `testb.log`).
  `shutdown_probe.sh` läuft bis zum nächsten Ausschalten.
