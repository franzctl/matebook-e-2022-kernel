# Linux auf dem Huawei MateBook E 2022

Fedora auf dem Huawei MateBook E 2022, mit selbst gebautem Kernel 6.1.4 statt schwarzem Bildschirm. Dieses Repo dokumentiert Fehlersuche, Lösung und Bau, damit sich das Tablet jederzeit neu aufsetzen lässt.

Entstanden ist das Projekt für einen Mitumschüler, dessen Tablet unter Linux kein Bild zeigte.

- **Gerät:** Huawei MateBook E 2022, Intel Core i3-1110G4 (Tiger Lake), Dual-Link-DSI-Panel mit 3072 × 1920
- **System:** Fedora 44 mit Vanilla-Kernel 6.1.4, als RPM paketiert

## Problem

Unter aktuellen Fedora-Kerneln bleibt der Bildschirm schwarz, das System selbst läuft weiter. Das Kernel-Log zeigt, warum (Auszug):

```text
mismatch in hw.pipe_mode.crtc_htotal (expected 2842, found 2728)
mismatch in hw.pipe_mode.crtc_vtotal (expected 1710, found 1800)
mismatch in pixel_rate (expected 291594, found 332418)
pipe state doesn't match!
```

Der Intel-Grafiktreiber i915 berechnet für das Dual-Link-DSI-Panel andere Timings, als er vom Panel zurückliest. Der State-Checker erkennt die Abweichung und bricht das Modesetting ab, das Panel bekommt kein Bild. Dasselbe Fehlerbild auf demselben Gerät ist upstream als [drm/i915 Issue #6607](https://gitlab.freedesktop.org/drm/i915/kernel/-/issues/6607) gemeldet. Laut Bugtracker zeigten Kernel bis 6.1.4 noch ein sauberes Bild.

## Weg zur Lösung

1. **Erster Ansatz:** ein Patch aus dem i915-Bugtracker für die GPIO- und Hotplug-Steuerung in `intel_dsi_vbt.c`. Er ließ sich sauber bauen und laden, das Bild blieb trotzdem aus.
2. **Genauer hinsehen:** Die Logs zeigten Timing-Abweichungen, kein GPIO-Problem. Der Patch setzte also an der falschen Stelle an.
3. **Bekannten guten Stand testen:** Vanilla-Kernel 6.1.4 gebaut. Ergebnis: sauberes Bild.
4. **Stolperstein Modulkompression:** Die Module lagen xz-komprimiert vor, und 6.1.4 konnte sie beim Laden nicht entpacken (`Decompression failed with status 6`). Gelöst durch unkomprimierte Module.
5. **Paketieren:** Kernel als RPM gebaut, damit er sich mit `dnf` installieren lässt, dazu eine Anleitung für die Neuinstallation.

## Kernel bauen

Gebaut auf einem Debian-Rechner, weil das Tablet mit zwei Kernen für einen Kernel-Build zu langsam ist.

```bash
# Quellen holen und Signatur prüfen
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.4.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.4.tar.sign
gpg --locate-keys torvalds@kernel.org gregkh@kernel.org
xz -cd linux-6.1.4.tar.xz | gpg --verify linux-6.1.4.tar.sign -
tar xf linux-6.1.4.tar.xz
cd linux-6.1.4

# Config des Fedora-Kernels als Basis übernehmen und auf 6.1.4 anpassen
# (auf dem Tablet vorher: cp /boot/config-$(uname -r) ~/config-fedora, dann auf den Build-Rechner kopieren)
cp ~/config-fedora .config
make olddefconfig

# Was auf dem Build-Rechner fehlt oder nur Zeit kostet, abschalten
scripts/config --disable DEBUG_INFO_BTF
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS

# Module unkomprimiert lassen, sonst scheitert das Laden unter 6.1.4
scripts/config --disable MODULE_COMPRESS_XZ
scripts/config --enable MODULE_COMPRESS_NONE
make olddefconfig

# Bauen und als RPM paketieren (auf Debian vorher: sudo apt install rpm)
make -j"$(nproc)" binrpm-pkg
```

Das Paket liegt danach unter `~/rpmbuild/RPMS/x86_64/`.

## Automatisch bauen lassen

Das RPM vom ersten Bau ging bei der Neuinstallation meines Laptops verloren. Statt es von Hand neu zu bauen, erledigt das jetzt GitHub Actions, genau nach den Schritten oben ([Workflow](.github/workflows/kernel.yml)):

- Quellcode von kernel.org laden und die Signatur prüfen. Die Fingerabdrücke von Greg Kroah-Hartman und Linus Torvalds stehen fest im Workflow.
- Die Konfiguration aus [`config/config-6.1.4`](config/config-6.1.4) übernehmen, die Änderungen von oben anwenden und prüfen, ob sie wirklich gelten
- In derselben Umgebung bauen wie beim ersten Mal: Debian 13 mit GCC 14
- Prüfen, dass im fertigen RPM keine xz-komprimierten Module stecken und der Grafiktreiber dabei ist

Eine Abweichung vom Bau von Hand: Die Debug-Informationen sind aus, sonst reicht der Speicherplatz auf GitHub nicht. Für den Betrieb spielen sie keine Rolle.

Bei einem Tag wie `v6.1.4-matebook.1` entsteht ein Release mit den RPMs, der verwendeten Konfiguration, dem Quellcode samt Signatur und Prüfsummen. Es bleibt als Vorabversion markiert, bis es auf dem Tablet getestet ist.

## Installieren

Auf dem Tablet:

1. **Secure Boot im BIOS ausschalten.** Der Kernel ist nicht von Fedora signiert und startet sonst nicht.
2. Fedora im Boot-Menü über **„Basic Graphics Mode“** installieren. Der Modus setzt `nomodeset`, umgeht den i915-Treiber und liefert ein einfaches Bild, das für den Installer reicht.
3. Das RPM aus den [Releases](../../releases) herunterladen, mit `sha256sum -c SHA256SUMS.txt --ignore-missing` prüfen, installieren, als Standard setzen und `nomodeset` für ihn entfernen, falls die Installation es übernommen hat:

   ```bash
   sudo dnf install ./kernel-6.1.4-*.rpm
   sudo grubby --set-default /boot/vmlinuz-6.1.4
   sudo grubby --update-kernel=/boot/vmlinuz-6.1.4 --remove-args="nomodeset"
   ```

4. Kernel-Updates sperren, damit kein neuer Kernel den Fix wieder verdrängt:

   ```bash
   echo "excludepkgs=kernel*" | sudo tee -a /etc/dnf/dnf.conf
   ```

5. Neu starten und die Lagesensoren prüfen. Die automatische Bilddrehung läuft über `iio-sensor-proxy`:

   ```bash
   monitor-sensor
   ```

## Stand

- Mit 6.1.4 läuft das Display sauber, die Lagesensoren für die Bilddrehung werden erkannt.
- Die Kamera (Intel IPU6) ist bewusst ausgeklammert. Sie bräuchte einen Kernel ab 6.10, das Display einen bis 6.1.4.

## Abwägung Sicherheit

Kernel 6.1.4 stammt vom Januar 2023. Als fester Stand bekommt er keine Sicherheitsupdates, er ist hier bewusst festgesetzt, weil das Gerät sonst kein Bild hat. Der nächste Test: ob spätere Versionen der LTS-Reihe 6.1 den Fehler ebenfalls zeigen. Dazu gehört der Blick ins Upstream-Issue, ob ein Fix kommt.

## Was ich daraus mitnehme

- Erst die Logs vollständig lesen, dann bauen. Ein Kernel-Build kostet Stunden, ein Patch an der falschen Stelle auch.
- Ein bekannter guter Stand als Referenz grenzt einen Fehler schneller ein als jeder Patch ins Blaue.

## Lizenz

Dokumentation unter MIT-Lizenz, siehe [LICENSE](LICENSE). Der Linux-Kernel steht unter GPL-2.0. Sein Quellcode liegt nicht im Repo, gehört aber zu jedem Release dazu, als Originalarchiv von kernel.org mit Signatur. Die Konfigurationsdatei unter `config/` stammt aus dem Kernel-Bau.
