# Adding BLE Support for B0/MCUboot S0 and S1 Updates

## Introduction

This is a companion to **Adding B0 (NSIB) and MCUboot to an nRF54LM20A Project**. It assumes that project is already in place — B0 verifying MCUboot from S0/S1, MCUboot verifying the application, mcumgr already working over UART — and covers the extra step of updating MCUboot's S0/S1 slots wirelessly over BLE.

Quick recap of why this needs MCUboot at all: NSIB/B0 has no support for firmware updates over SMP transport, full stop. BLE DFU only becomes possible because MCUboot is sitting underneath B0 doing the verification and swap work; B0 itself stays out of the update path entirely, immutable and untouched.

---

## Enabling the BLE Transport

The application already has the common mcumgr/IMG_MANAGER dependencies enabled from the UART setup (`CONFIG_MCUMGR`, `CONFIG_MCUMGR_GRP_IMG`, `CONFIG_IMG_MANAGER`, and the rest — see the main article for the full list). Add the BLE-specific lines on top, in `prj.conf`:

```ini
# BLE transport
CONFIG_MCUMGR_TRANSPORT_BT=y
CONFIG_MCUMGR_TRANSPORT_BT_PERM_RW=y
CONFIG_MCUMGR_TRANSPORT_BT_REASSEMBLY=y
CONFIG_NCS_SAMPLE_MCUMGR_BT_OTA_DFU=y

CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr"
CONFIG_BT_SMP=y
CONFIG_BT_MAX_CONN=1
CONFIG_BT_BUF_ACL_RX_SIZE=251
CONFIG_BT_BUF_ACL_TX_SIZE=251
```

Unlike the UART transport, BLE doesn't expose the SMP server just by enabling Kconfig — the application has to actually advertise and accept a connection first, which is application code, not configuration.

---

## Advertising for SMP over BLE

A minimal advertising setup, adapted from Zephyr's `smp_svr` sample, makes the device discoverable to any SMP client (nRF Connect Device Manager, the `mcumgr` CLI):

```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/mgmt/mcumgr/transport/smp_bt.h>
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(smp_bt_app, LOG_LEVEL_INF);

static struct k_work advertise_work;

static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_UUID128_ALL, SMP_BT_SVC_UUID_VAL),
};

static const struct bt_data sd[] = {
    BT_DATA(BT_DATA_NAME_COMPLETE, CONFIG_BT_DEVICE_NAME, sizeof(CONFIG_BT_DEVICE_NAME) - 1),
};

static void advertise(struct k_work *work)
{
    int rc = bt_le_adv_start(BT_LE_ADV_CONN_FAST_1, ad, ARRAY_SIZE(ad), sd, ARRAY_SIZE(sd));
    if (rc) {
        LOG_ERR("Advertising failed to start (rc %d)", rc);
        return;
    }
    LOG_INF("Advertising successfully started");
}

static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        LOG_ERR("Connection failed, err 0x%02x %s", err, bt_hci_err_to_str(err));
        k_work_submit(&advertise_work);
    } else {
        LOG_INF("Connected");
    }
}

static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    LOG_INF("Disconnected, reason 0x%02x %s", reason, bt_hci_err_to_str(reason));
}

static void on_conn_recycled(void)
{
    k_work_submit(&advertise_work);
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
    .recycled = on_conn_recycled,
};

static void bt_ready(int err)
{
    if (err != 0) {
        LOG_ERR("Bluetooth failed to initialise: %d", err);
    } else {
        k_work_submit(&advertise_work);
    }
}

void start_smp_bluetooth_adverts(void)
{
    int rc;
    k_work_init(&advertise_work, advertise);
    rc = bt_enable(bt_ready);
    if (rc != 0) {
        LOG_ERR("Bluetooth enable failed: %d", rc);
    }
}
```

Called once from `main()`:

```c
int main(void)
{
    start_smp_bluetooth_adverts();
    /* ... rest of application ... */
}
```

---

## Actually Performing the Update Over BLE

The mechanics are identical to the UART flow described in the main article: same S0/S1 image files, same rule about uploading whichever slot isn't currently active. Only the connection string changes.

As a reminder, once the project is built there will be two files — `signed_by_mcuboot_and_b0_s1_image.bin` for S1 and `signed_by_mcuboot_and_b0_mcuboot.bin` for S0. Each carries metadata about where it's meant to live, so the slot that's currently *inactive* is always the upload target; the active one needs to be re-checked at update time, not assumed from the last update.

```bash
mcumgr --conntype ble --connstring peer_name='Zephyr' image upload build/<s0 or s1 signed image>
```

followed by:

```bash
mcumgr --conntype ble --connstring peer_name='Zephyr' image confirm
mcumgr --conntype ble --connstring peer_name='Zephyr' image reset
```

AuTerm (https://github.com/thedjnK/AuTerm/) supports BLE connections too, and is generally a less error-prone way to drive this than typing connstrings by hand on the bench.

As with the UART path, there's no automatic detection yet of which slot is currently active — that decision has to be made by whoever is initiating the update. See the main article's "Open item" section for where that work stands; it isn't transport-specific, so solving it once covers both UART and BLE.

---

## References

- Main article: **Adding B0 (NSIB) and MCUboot to an nRF54LM20A Project**
- AuTerm, BLE/UART SMP client: https://github.com/thedjnK/AuTerm/
- Nordic Developer Academy, "Bootloaders and DFU / FOTA" lesson, exercises on DFU over UART/BLE
