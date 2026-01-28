# BluFi Provisioning (integrated esp‑wifi‑connect) - blufi.md

## Prerequisites

- A chip/firmware that supports BLE is required.  
- In `idf.py menuconfig` enable **Wi‑Fi Configuration Method → Esp Blufi** (`CONFIG_USE_ESP_BLUFI_WIFI_PROVISIONING=y`).  
  If you only want to use BluFi, you can disable the **Hotspot/Acoustic** options in the same menu.

- Keep the default NVS and event‑loop initialization (already handled in the project's `app_main`).  
- The macros `CONFIG_BT_BLUEDROID_ENABLED` and `CONFIG_BT_NIMBLE_ENABLED` should be set to **one** of them; they cannot be enabled at the same time.

## Workflow

1) The mobile app connects to the device via BluFi (e.g., the official **EspBlufi** app or a custom client) and sends the Wi‑Fi SSID/password.  

2) On the device side, the credentials are written to `SsidManager` (stored in NVS, part of the `esp-wifi-connect` component) in the `ESP_BLUFI_EVENT_REQ_CONNECT_TO_AP` event.  

3) Afterwards, `WifiStation` is started to scan and connect; the status is reported back through BluFi.  

4) After a successful provisioning, the device automatically connects to the new Wi‑Fi; if it fails, a failure status is returned.

## Usage Steps

1. **Configuration**: Enable `Esp Blufi` in `menuconfig`, compile, and flash the firmware.  

2. **Trigger provisioning**: The device automatically enters provisioning mode on first boot if no saved Wi‑Fi credentials exist.  

3. **Mobile operation**: Open the **EspBlufi** app (or another BluFi client), scan for and connect to the device, optionally enable encryption, enter the Wi‑Fi SSID/password as prompted, and send.  

4. **Observe results**:  
   - **Success** – BluFi reports a successful connection and the device automatically connects to Wi‑Fi.  
   - **Failure** – BluFi returns a failure status; you can resend or check the router.

## Notes

- BluFi can be compiled together with Hotspot/Acoustic provisioning, but both will start up simultaneously, increasing memory usage. It is recommended to keep only one method enabled in `menuconfig`.  

- If you perform multiple tests, consider clearing or overwriting the stored SSID (the `wifi` namespace) to avoid interference from old configurations.  

- When using a custom BluFi client, you must follow the official protocol frame format; refer to the official documentation link above.  

- The official documentation provides a download link for the **EspBlufi** app: https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/ble/blufi.html.