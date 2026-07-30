# HA Touch Panel

ESPHome firmware for Guition JC8048W550 (ESP32-S3, 800×480) Home Assistant room panels.

## First-time installation

Connect the panel via USB-C, then use the button below to flash factory firmware from your browser (Chrome or Edge required).

If the flasher cannot connect, hold **BOOT**, tap **RESET**, then release **BOOT**, and try again.

After install, keep USB connected and use **Configure Wi-Fi** in the installer dialog (Improv over serial).

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

### Suite (`home/suite`)

<esp-web-install-button manifest="firmware/manifest.json">
  <button slot="activate">Install Suite panel</button>
  <span slot="unsupported">Your browser does not support WebSerial. Use Chrome or Edge on desktop.</span>
  <span slot="not-allowed">HTTPS is required (or use localhost).</span>
</esp-web-install-button>

## After flashing

**Preferred:** use **Configure Wi-Fi** in the browser installer while USB is still connected.

**Fallback (SoftAP):**

1. The device starts a Wi-Fi access point (password: `touchpanel-setup`).
2. Connect with your phone — the captive portal opens automatically (or go to http://192.168.4.1/).
3. Enter your home Wi-Fi credentials.

Then add the device in Home Assistant via the ESPHome integration.

**Important:** open the ESPHome device in HA → **Configure** → enable **Allow the device to perform Home Assistant actions**. Without that, taps on the panel cannot control entities.

Firmware updates are offered automatically in Home Assistant when a new release is published (HTTP OTA from this site).

## Releases

Built binaries and manifests are attached to [GitHub Releases](https://github.com/dflourusso/ha-touch-panel/releases).
