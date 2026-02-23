# avionmesh-cpp

A C++20 library implementing the Avi-on mesh lighting protocol, built on top of [recsrmesh-cpp](https://github.com/oyvindkinsey/recsrmesh-cpp).

Platform independence is inherited from recsrmesh-cpp — no BLE stack or hardware dependencies are introduced at this layer.

The equivalent Python library is [avionmesh](https://github.com/oyvindkinsey/avionmesh).

## features

- Build and send commands to Avi-on mesh devices: brightness, color temperature, group management, ping, date/time sync
- Parse MCP responses into device state (brightness, color temperature, on/off)
- Targets individual devices, groups, or the entire mesh (broadcast)
- Color temperature can be set without changing on/off or brightness state

## usage

```cpp
#include <avionmesh/avionmesh.h>

// Initialise the underlying CSRMesh context
csrmesh::MeshContext mesh;
csrmesh::init(mesh, ble_write_fn, "your-base64-passphrase");

// Set brightness on a device
avionmesh::Command cmd;
avionmesh::cmd_brightness(device_id, 200, cmd);
avionmesh::send_cmd(mesh, cmd);

// Set color temperature (kelvin) without turning on
avionmesh::cmd_color_temp(device_id, 2700, cmd);
avionmesh::send_cmd(mesh, cmd);

// Add device to a group
avionmesh::cmd_insert_group(device_id, group_id, cmd);
avionmesh::send_cmd(mesh, cmd);

// Parse an incoming response
avionmesh::Status status;
csrmesh::set_rx_callback(mesh, [&](uint16_t mcp_src, uint16_t crypto_src,
                                    uint8_t opcode, const uint8_t *payload, size_t len) {
    avionmesh::parse_response(mcp_src, crypto_src, opcode, payload, len, status);
});
```

## dependencies

- [recsrmesh-cpp](https://github.com/oyvindkinsey/recsrmesh-cpp) — CSRMesh BLE transport and crypto layer (included as a submodule)

## used by

- [esphome-avionmesh](https://github.com/oyvindkinsey/esphome-avionmesh) — ESPHome component for Home Assistant integration via MQTT

## license

LGPL-3.0-or-later — See the `LICENSE` file for details.
