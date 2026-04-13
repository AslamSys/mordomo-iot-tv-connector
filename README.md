# 📺 IoT TV Connector

## 🔗 Navegação

**[🏠 AslamSys](https://github.com/AslamSys)** → **[📚 _system](https://github.com/AslamSys/_system)** → **[📂 IoT (RPi 3B+)](https://github.com/AslamSys/_system/blob/main/hardware/iot%20-%20(raspberry-pi-3b)/README.md)** → **iot-tv-connector**

### Containers Relacionados (iot)
- [iot-orchestrator](https://github.com/AslamSys/iot-orchestrator)
- [iot-mqtt-broker](https://github.com/AslamSys/iot-mqtt-broker)
- [iot-state-cache](https://github.com/AslamSys/iot-state-cache)

---

**Container:** `iot-tv-connector`  
**Ecossistema:** IoT  
**Hardware:** Raspberry Pi 3B+  
**Sem LLM:** Execução direta de comandos

---

## 📋 Propósito

Controlador genérico de TVs. Recebe comandos unificados via NATS (`iot.tv.*`) e traduz para o protocolo nativo de cada TV detectada na rede. O `iot-orchestrator` nunca sabe qual marca/protocolo está conectado.

---

## 🎯 Responsabilidades

- ✅ Auto-descoberta de TVs na rede local (mDNS/SSDP)
- ✅ Roteamento de comandos para o adaptador correto por device_id
- ✅ Interface unificada independente de marca/protocolo
- ✅ Reprodução de mídia (URL de stream ou arquivo Jellyfin)
- ✅ Controles básicos: ligar, desligar, volume, input, pause/play
- ✅ Reportar estado atual ao iot-state-cache

---

## 🔌 Adaptadores

| Marca | Protocolo | Status | Notas |
|---|---|---|---|
| **LG webOS** | WebSocket (porta 3000) | ✅ Implementado | Magic Remote, lança mídia diretamente |
| Samsung Tizen | WebSocket (porta 8001) | 🔜 Planejado | SmartThings API alternativa |
| Sony Android TV | REST (Android Debug Bridge) | 🔜 Planejado | |
| Chromecast | Cast SDK | 🔜 Planejado | Qualquer TV com HDMI |
| DLNA genérico | UPnP/DLNA | 🔜 Planejado | Fallback para TVs sem API |

---

## 🧠 Arquitetura de Adaptadores

```
NATS "iot.tv.play_media"
        ↓
  TV Connector
        ↓
  device_registry.yaml  →  device_id: "tv_sala" → type: "lg_webos", ip: "192.168.1.50"
        ↓
  ┌─────────────────┐
  │  AdapterFactory │
  └────────┬────────┘
           ├── LGWebOSAdapter   → ws://192.168.1.50:3000  (WebSocket)
           ├── SamsungAdapter   → ws://tv-ip:8001         (WebSocket)
           └── DLNAAdapter      → UPnP SOAP               (HTTP)
```

---

## 🔌 NATS Topics

### Subscribe
```javascript
// Reproduzir mídia (URL direta ou item Jellyfin)
Topic: "iot.tv.play_media"
Payload: {
  "device_id": "tv_sala",
  "media_url": "http://nas:8096/Videos/123/stream.mkv",  // Jellyfin stream URL
  "title": "Cast Away (2000)",
  "type": "video"  // "video" | "audio" | "image"
}

// Controles de playback
Topic: "iot.tv.control"
Payload: {
  "device_id": "tv_sala",
  "action": "pause"  // "pause" | "play" | "stop" | "seek"
  "position_seconds": 120  // apenas para "seek"
}

// Controles básicos
Topic: "iot.tv.power"
Payload: { "device_id": "tv_sala", "state": "on" }  // "on" | "off"

Topic: "iot.tv.volume"
Payload: { "device_id": "tv_sala", "level": 30, "mute": false }

Topic: "iot.tv.input"
Payload: { "device_id": "tv_sala", "source": "hdmi1" }
```

### Publish
```javascript
// Estado atual da TV (sincronizado com iot-state-cache)
Topic: "iot.tv.state"
Payload: {
  "device_id": "tv_sala",
  "power": "on",
  "volume": 30,
  "input": "app",
  "playing": {
    "title": "Cast Away",
    "media_url": "...",
    "position_seconds": 245,
    "duration_seconds": 7200
  }
}
```

---

## 🔧 LG webOS — Adaptador (v1)

A LG webOS expõe uma API completa via WebSocket na porta 3000. Não precisa de IR nem de MQTT.

```javascript
// Conexão inicial (requer pareamento no primeiro uso)
const lgtv = require('lgtv2');
const tv = lgtv({ url: 'ws://192.168.1.50:3000' });

// Lançar mídia diretamente
tv.request('ssap://media.viewer/open', {
  target: 'http://nas:8096/Videos/123/stream.mkv'
});

// Volume
tv.request('ssap://audio/setVolume', { volume: 30 });

// Desligar
tv.request('ssap://system/turnOff');
```

**Pareamento:** Na primeira conexão, a TV exibe um prompt de autorização. O `client-key` gerado é salvo em `/data/lg_client_key.json` e reutilizado nas conexões seguintes.

---

## ⚙️ Configuração

```yaml
# config/devices.yaml
devices:
  tv_sala:
    type: lg_webos
    ip: "192.168.1.50"
    name: "TV Sala"
    room: "sala"
    jellyfin_client_name: "LG webOS"  # nome registrado no Jellyfin
```

---

## 🚀 Docker

```yaml
iot-tv-connector:
  build: ./iot-tv-connector
  environment:
    - NATS_URL=nats://mordomo-nats:4222
    - DEVICES_CONFIG=/data/devices.yaml
    - LG_CLIENT_KEY_PATH=/data/lg_client_key.json
  volumes:
    - ./data:/data
  networks:
    - iot-net
    - shared-nats
  deploy:
    resources:
      limits:
        cpus: '0.3'
        memory: 128M
```

---

## 🔄 Changelog

### v1.0.0
- ✅ LG webOS adapter (WebSocket)
- ✅ Interface NATS unificada `iot.tv.*`
- ✅ Auto-descoberta via mDNS
- ✅ Reprodução de mídia via URL Jellyfin
