# Bridge Arcson Security

A web-based security monitoring prototype for an ESP32 setup. A Node.js server connects MQTT sensor messages to a browser dashboard through Socket.IO, stores the latest sensor states in MySQL, and forwards uploaded JPEG frames as an MJPEG camera stream.

The dashboard is in Spanish and displays PIR motion status, lock status, server connectivity, and a camera feed. This repository contains the backend and web client. ESP32 firmware, wiring instructions, and broker configuration are not included.

## How it works

```text
ESP32 / MQTT publisher -> MQTT broker -> Node.js server -> Socket.IO -> Dashboard
                                             |
                                             v
                                           MySQL

Camera -> HTTP JPEG uploads -> Node.js server -> HTTP MJPEG stream -> Dashboard
```

MySQL stores the latest motion and lock states, not an event history. The server reads these states when a browser connects. Camera frames are held in memory, with only the latest frame retained for new stream viewers.

## Requirements

- Node.js 18 or newer and npm, as required by the installed Express version.
- A running MySQL server.
- An MQTT broker listening at `mqtt://localhost:1883` on the backend machine.
- Python 3 or another static file server for the dashboard.
- Internet access for the dashboard's Bootstrap stylesheet, loaded from jsDelivr.

Hardware is optional for the manual checks below. A physical setup needs a device that publishes the MQTT messages and a camera device that uploads JPEG frames.

## Setup

Run commands from the repository root unless otherwise noted.

### 1. Install dependencies

```sh
npm ci
```

### 2. Initialize MySQL

The repository does not include migrations. Run the following minimal schema in your MySQL client, using an account allowed to create the database and table:

```sql
CREATE DATABASE IF NOT EXISTS bridgearcson;
USE bridgearcson;

CREATE TABLE IF NOT EXISTS sensores (
    nombre VARCHAR(50) NOT NULL PRIMARY KEY,
    estado BOOLEAN NOT NULL DEFAULT FALSE
);

INSERT IGNORE INTO sensores (nombre, estado)
VALUES ('movimiento', FALSE), ('cerrojo', FALSE);
```

Both rows must exist because incoming MQTT messages update existing records. The application database account needs `SELECT` and `UPDATE` permissions on this table.

### 3. Configure and start the backend

The backend reads these environment variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `DB_HOST` | `localhost` | MySQL host |
| `DB_USER` | `root` | MySQL user |
| `DB_PASS` | Empty string | MySQL password |
| `DB_NAME` | `bridgearcson` | MySQL database |

For example, set credentials in your shell before starting:

```sh
export DB_HOST=localhost
export DB_USER=bridgearcson_app
export DB_PASS='replace-with-your-password'
export DB_NAME=bridgearcson
npm start
```

Use an existing MySQL account with the required permissions. The example does not create that account. The application does not automatically load a `.env` file.

Start the MQTT broker before the backend. The HTTP and Socket.IO server listens on port `3000`. The port and MQTT broker URL are hardcoded in `server/server.js`; there are no `PORT` or MQTT environment settings.

### 4. Point the dashboard at the backend

In `client/index.html`, replace all three occurrences of `192.168.0.23` with the backend host:

- Camera image URL: `http://<backend-host>:3000/camera/stream`
- Socket.IO script URL: `http://<backend-host>:3000/socket.io/socket.io.js`
- Socket.IO connection URL: `ws://<backend-host>:3000`

Use `localhost` when the browser and backend run on the same machine. For another computer or an ESP32, use the backend machine's reachable LAN address.

### 5. Serve the dashboard

In a separate terminal:

```sh
python3 -m http.server 8080 --directory client
```

Open <http://localhost:8080>. Express does not serve the `client` directory, so opening port `3000` directly does not show the dashboard. See the known limitations below for current client-side errors.

## MQTT and Socket.IO messages

Sensor payloads are case-sensitive strings. `ON` means active or closed; `OFF` means inactive or open. The backend treats any payload other than `ON` as false when storing a state.

| MQTT topic | Behavior |
| --- | --- |
| `sensor/movimiento` | Emits the Socket.IO event `movimiento` and updates the `movimiento` database row. |
| `sensor/cerrojo` | Emits the Socket.IO event `cerrojo` and updates the `cerrojo` database row. |
| `actuador/cerrojo` | Subscribed to, but currently has no message handler. |

The backend also accepts Socket.IO events `cerrar` and `abrir`, publishing `ON` and `OFF` respectively to `sensor/cerrojo`. These commands use the sensor topic, not `actuador/cerrojo`. Firmware must match this behavior; a displayed state is not independent confirmation that a physical lock moved.

## Camera endpoints

| Method | Path | Behavior |
| --- | --- | --- |
| `POST` | `/camera/upload` | Accepts a raw JPEG body with `Content-Type: image/jpeg`, limited to `2mb`. Returns `200` for an accepted frame and `400` for an empty body. |
| `GET` | `/camera/stream` | Keeps an MJPEG response open and sends frames as they arrive. Sends the latest frame immediately if one is available. |

Configure the camera device to repeatedly upload JPEG bytes to `http://<backend-host>:3000/camera/upload`. The endpoint expects raw bytes, not JSON or a multipart form upload. Video has no audio and is not recorded to disk.

## Manual checks

With MySQL, the MQTT broker, and the backend running, use the Mosquitto command-line client, if installed, to simulate sensor messages:

```sh
mosquitto_pub -h localhost -p 1883 -t sensor/movimiento -m ON
mosquitto_pub -h localhost -p 1883 -t sensor/movimiento -m OFF
mosquitto_pub -h localhost -p 1883 -t sensor/cerrojo -m ON
mosquitto_pub -h localhost -p 1883 -t sensor/cerrojo -m OFF
```

These messages update the database and broadcast to connected browsers. The lock messages may also affect connected hardware that consumes that topic.

Inspect the stored states in MySQL:

```sql
SELECT nombre, estado FROM bridgearcson.sensores;
```

Upload an existing JPEG file:

```sh
curl -i -X POST http://localhost:3000/camera/upload \
  -H 'Content-Type: image/jpeg' \
  --data-binary @/path/to/frame.jpg
```

Open <http://localhost:3000/camera/stream> to see the uploaded frame. Repeated uploads provide the live feed.

There is no automated test script in `package.json`.

## Known limitations and troubleshooting

- **Missing dashboard elements:** JavaScript references `toggle-sensor`, `toggle-cerrojo`, `label-sensor`, and `label-cerrojo`, but those elements are absent from the HTML. This causes null-reference errors during initialization and sensor updates. The current page does not expose working lock controls. Restore the elements or guard the references before relying on those interactions.
- **Dashboard stays offline:** Check all three backend URLs in `client/index.html` and ensure the browser can reach port `3000`. The Online indicator reflects the Socket.IO connection, not overall MQTT, database, or hardware health.
- **Database errors or missing states:** Verify credentials, the `sensores` table, and both seed rows. Database calls in asynchronous event handlers do not currently have error handling.
- **No sensor updates:** Verify the broker is reachable from the backend at `localhost:1883`, and publishers use the exact topics and payloads above.
- **Blank camera feed:** No image is available until a device or manual request uploads a frame. Check the upload content type and the dashboard's stream URL.
- **Deployment scope:** The prototype has no application authentication or authorization, uses unencrypted HTTP/MQTT connections, and allows all Socket.IO origins. It needs access controls and transport security before exposure beyond a trusted development network.

## Project structure

```text
client/
  index.html          Dashboard markup and browser-side Socket.IO logic
  css/style.css       Dashboard styles
  src/Logos/          Brand images and favicon
  src/Productos/      Product image
server/
  server.js           MQTT bridge, MySQL access, Socket.IO, camera endpoints
package.json          Dependencies and start command
package-lock.json     Locked dependency versions
```

## License

`package.json` declares ISC. A standalone license file is not included.
