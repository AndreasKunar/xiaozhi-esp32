# MCP Protocol IoT Control Usage Guide - mcp-usage.md

> This document explains how to implement IoT control for ESP32 devices based on the MCP protocol. Detailed protocol procedures are referenced in [`mcp-protocol.md`](./mcp-protocol.md).

## Introduction

MCP (**Model Context Protocol**) is a next‑generation protocol recommended for IoT control. It discovers and invokes **"tools"** on the backend through a standard JSON‑RPC 2.0 format, enabling flexible device control.

## Typical Usage Flow

1. After the device starts up, it establishes a connection with the backend via an underlying protocol (e.g., WebSocket/MQTT).
2. The backend initializes the session using the MCP protocol's `initialize` method.
3. The backend retrieves all supported tools (features) and their parameter descriptions from the device via `tools/list`.
4. The backend calls specific tools via `tools/call` to control the device.

For detailed protocol formats and interactions, see [`mcp-protocol.md`](./mcp-protocol.md).

## Device‑Side Tool Registration Method Explanation

A device registers callable **"tools"** on the backend by invoking `McpServer::AddTool`. The typical function signature is:

```cpp
void AddTool(
    const std::string& name,           // Tool name, recommended to be unique and hierarchical, e.g., self.dog.forward
    const std::string& description,    // Tool description, a concise explanation of its function, helpful for AI/user understanding
    const PropertyList& properties,    // List of input parameters (can be empty), supporting types: boolean, integer, string
    std::function<ReturnValue(const PropertyList&)> callback // Callback implementation executed when the tool is invoked
);
```

- **name**: Tool identifier; it is recommended to use a `"module.feature"` naming style.
- **description**: Natural‑language description; helps AI or users understand the tool's purpose.
- **properties**: Parameter list; supported types include boolean, integer, and string, with optional range and default values.
- **callback**: The actual execution logic when the tool is called; the callback may return `bool`, `int`, or `string`.

### Typical Registration Example (using ESP‑Hi)

```cpp
void InitializeTools() {
    auto& mcp_server = McpServer::GetInstance();
    // Example 1: No parameters, command the robot to move forward
    mcp_server.AddTool("self.dog.forward", "Robot moves forward", PropertyList(), [this](const PropertyList&) -> ReturnValue {
        servo_dog_ctrl_send(DOG_STATE_FORWARD, NULL);
        return true;
    });
    // Example 2: With parameters, set RGB color for a light
    mcp_server.AddTool("self.light.set_rgb", "Set RGB color", PropertyList({
        Property("r", kPropertyTypeInteger, 0, 255),
        Property("g", kPropertyTypeInteger, 0, 255),
        Property("b", kPropertyTypeInteger, 0, 255)
    }), [this](const PropertyList& properties) -> ReturnValue {
        int r = properties["r"].value<int>();
        int g = properties["g"].value<int>();
        int b = properties["b"].value<int>();
        led_on_ = true;
        SetLedColor(r, g, b);
        return true;
    });
}
```

## Common JSON‑RPC Tool Call Examples

### 1. List Available Tools
```json
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": { "cursor": "" },
  "id": 1
}
```

### 2. Control Chassis to Move Forward
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.chassis.go_forward",
    "arguments": {}
  },
  "id": 2
}
```

### 3. Switch Light Mode
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.chassis.switch_light_mode",
    "arguments": { "light_mode": 3 }
  },
  "id": 3
}
```

### 4. Flip the Camera
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.camera.set_camera_flipped",
    "arguments": {}
  },
  "id": 4
}
```

## Notes

- Tool names, parameters, and return values should conform to those registered on the device side via `AddTool`.
- It is recommended that all new projects adopt the MCP protocol for IoT control.
- For detailed protocol specifications and advanced usage, refer to [`mcp-protocol.md`](./mcp-protocol.md).
