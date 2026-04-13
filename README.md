# 📺 mordomo-iot-tv-connector

## 🔗 Navegação

**[🏠 AslamSys](https://github.com/AslamSys)** → **[📚 _system](https://github.com/AslamSys/_system)** → **[📂 IoT (RPi 3B+)](https://github.com/AslamSys/_system/blob/main/hardware/iot%20-%20(raspberry-pi-3b)/README.md)** → **mordomo-iot-tv-connector**

### Containers Relacionados (iot)
- [mordomo-iot-orchestrator](https://github.com/AslamSys/mordomo-iot-orchestrator)
- [mordomo-iot-mqtt-broker](https://github.com/AslamSys/mordomo-iot-mqtt-broker)
- [mordomo-iot-state-cache](https://github.com/AslamSys/mordomo-iot-state-cache)

---

**Container:** `mordomo-iot-tv-connector`  
**Ecossistema:** Mordomo / IoT  
**Hardware:** Raspberry Pi 3B+  
**Sem LLM:** Execução direta de comandos

---

## 📋 Propósito

Controlador genérico de TVs. Recebe comandos unificados via NATS (`iot.tv.*`) e traduz para o protocolo nativo de cada TV. O `mordomo-iot-orchestrator` nunca sabe qual marca ou protocolo está conectado — só conhece `device_id`.

---

## 🔌 Adaptadores Suportados

| Marca | Protocolo | Porta | Status |
|---|---|---|---|
| **LG webOS** | WebSocket / WSS | 3000 (ws) · 3001 (wss) | ✅ Suportado |
| Samsung Tizen | WebSocket | 8001 | 🔜 Planejado |
| Sony Android TV | ADB REST | varies | 🔜 Planejado |
| Chromecast | Cast SDK | — | 🔜 Planejado |
| DLNA genérico | UPnP/SOAP | 1900 | 🔜 Planejado (fallback) |

---

## 🧠 Arquitetura

```
NATS "iot.tv.*"
        ↓
  mordomo-iot-tv-connector
        ↓
  devices.yaml  →  tv_sala: { type: lg_webos, ip: 192.168.1.50 }
        ↓
  AdapterFactory
        ├── LGWebOSAdapter   → wss://tv-ip:3001  (WebSocket SSL)
        ├── SamsungAdapter   → ws://tv-ip:8001
        └── DLNAAdapter      → HTTP UPnP SOAP
```

---

## 🔌 NATS Interface Unificada

### Subscribe (recebe do mordomo-brain / orchestrator)
```yaml
iot.tv.power:      { device_id, state: "on|off|screen_off|screen_on" }
iot.tv.volume:     { device_id, level: 0-100, mute: bool }
iot.tv.input:      { device_id, source: "hdmi1|hdmi2|app|..." }
iot.tv.control:    { device_id, action: "play|pause|stop|rewind|fast_forward" }
iot.tv.app:        { device_id, app_id: "com.webos.app.browser", url: "..." }
iot.tv.notify:     { device_id, message: "...", icon_url: "..." }
iot.tv.button:     { device_id, key: "home|back|ok|up|down|left|right|..." }
```

### Publish (envia ao iot-state-cache)
```yaml
iot.tv.state:
  device_id: tv_sala
  power: "on"
  volume: 30
  muted: false
  input: "app"
  foreground_app: "com.webos.app.browser"
```

---

## ⚙️ Configuração

```yaml
# config/devices.yaml
devices:
  tv_sala:
    type: lg_webos
    ip: "192.168.1.50"     # IP fixo no roteador (reserva DHCP por MAC)
    name: "TV Sala"
    room: "sala"
```

---

## 🚀 Docker

```yaml
mordomo-iot-tv-connector:
  build: ./mordomo-iot-tv-connector
  environment:
    - NATS_URL=nats://mordomo-nats:4222
    - DEVICES_CONFIG=/data/devices.yaml
    - LG_CLIENT_KEY_PATH=/data/lg_client_key.json
  volumes:
    - tv-connector-data:/data   # persiste client-key do pairing
  networks:
    - iot-net
    - shared-nats
  deploy:
    resources:
      limits:
        cpus: '0.3'
        memory: 128M

volumes:
  tv-connector-data:
```

---

---

# 📺 TV #1 — LG webOS (tv_sala)

## ✅ Vai funcionar?

**Sim** — com um detalhe: LG abandonou WebSocket simples (`ws://`, porta 3000) em firmwares a partir de **janeiro 2023**. Modelos atuais exigem **WSS** (`wss://`, porta 3001) com SSL.

O SSL da LG é **auto-assinado** — ou seja, não é um certificado de CA pública. Isso não impede a conexão: basta configurar o cliente para **não verificar o certificado** (`ssl_verify=False`). Não é um problema de segurança no contexto de rede local.

```python
# aiowebostv (biblioteca async Python — recomendada para o container)
from aiowebostv import WebOsClient

client = WebOsClient("192.168.1.50", client_key="abc123")
await client.connect()   # conecta via wss://192.168.1.50:3001 automaticamente
```

A biblioteca `aiowebostv` já lida com:
- Troca automática para WSS quando `ws` falha
- Aceitação do certificado auto-assinado
- Persistência do `client_key` após o primeiro pairing

---

## 🔧 Protocolo

| Item | Valor |
|---|---|
| Protocolo | WebSocket (SSL obrigatório em firmware ≥ 2023) |
| Porta | `3001` (wss) — porta `3000` (ws) apenas firmwares antigos |
| SSL | Auto-assinado pela LG — aceitar sem verificação |
| Autenticação | `client_key` gerado no primeiro pairing (prompt na TV) |
| Biblioteca Python | `aiowebostv` (async) ou `LGWebOSRemote` (CLI/sync) |

---

## 🔑 Pairing (primeiro uso)

1. TV e RPi na mesma rede Wi-Fi / LAN
2. Container sobe e tenta conectar → TV exibe prompt de autorização
3. Usuário aceita na TV → `client_key` gerado e salvo em `/data/lg_client_key.json`
4. Nas próximas conexões: reconecta automaticamente sem prompt

```json
// /data/lg_client_key.json (gerado automaticamente)
{
  "tv_sala": "a1b2c3d4e5f6..."
}
```

---

## 🎮 Comandos Disponíveis

### Power
```yaml
iot.tv.power: { device_id: tv_sala, state: "on" }        # Wake-on-LAN (requer MAC)
iot.tv.power: { device_id: tv_sala, state: "off" }       # ssap://system/turnOff
iot.tv.power: { device_id: tv_sala, state: "screen_off" } # tela apaga, som continua
iot.tv.power: { device_id: tv_sala, state: "screen_on" }  # reativa a tela
```

> ⚠️ `"on"` usa **Wake-on-LAN** — funciona apenas se a TV estiver em standby na mesma rede. Requer MAC address configurado.

### Volume & Áudio
```yaml
iot.tv.volume: { device_id: tv_sala, level: 30 }
iot.tv.volume: { device_id: tv_sala, mute: true }
# saídas disponíveis: tv_speaker | external_optical | external_arc | headphone | bt_soundbar
```

### Inputs
```yaml
iot.tv.input: { device_id: tv_sala, source: "HDMI_1" }
iot.tv.input: { device_id: tv_sala, source: "HDMI_2" }
# listagem dinâmica via: ssap://tv/getExternalInputList
```

### Apps
```yaml
iot.tv.app: { device_id: tv_sala, app_id: "com.webos.app.browser", url: "http://nas:8096" }
iot.tv.app: { device_id: tv_sala, app_id: "youtube.leanback.v4" }
iot.tv.app: { device_id: tv_sala, app_id: "netflix" }
# lista completa: ssap://com.webos.applicationManager/listApps
```

### Mídia (playback via browser embutido)
```yaml
# Opção 1: abre o Jellyfin no browser da TV
iot.tv.app:
  device_id: tv_sala
  app_id: com.webos.app.browser
  url: "http://192.168.1.X:8096"

# Opção 2: tenta abrir mídia direta (funciona em alguns modelos)
# ssap://media.viewer/open { target: "http://nas/video.mkv" }
# ⚠️ suporte variável por modelo/firmware — não confiável para MKV
```

> **Nota sobre cast de vídeo:** A LG não tem API pública para "cast" direto de arquivo de vídeo de forma confiável. A abordagem mais estável é abrir o **Jellyfin no browser embutido** ou usar **DLNA** (se habilitado na TV).

### Controles de Playback
```yaml
iot.tv.control: { device_id: tv_sala, action: "pause" }
iot.tv.control: { device_id: tv_sala, action: "play" }
iot.tv.control: { device_id: tv_sala, action: "stop" }
iot.tv.control: { device_id: tv_sala, action: "rewind" }
iot.tv.control: { device_id: tv_sala, action: "fast_forward" }
```

### Botões do Controle Remoto
```yaml
iot.tv.button: { device_id: tv_sala, key: "home" }
iot.tv.button: { device_id: tv_sala, key: "back" }
iot.tv.button: { device_id: tv_sala, key: "ok" }
# disponíveis: home | back | exit | ok | up | down | left | right
#              play | pause | stop | rewind | fast_forward
#              red | green | yellow | blue
#              volume_up | volume_down | channel_up | channel_down
```

### Notificações na Tela
```yaml
iot.tv.notify: { device_id: tv_sala, message: "Renan chegou em casa" }
iot.tv.notify: { device_id: tv_sala, message: "Porta aberta", icon_url: "http://..." }
# aparece como toast/overlay sem interromper o que está passando
```

### Informações / Estado
```yaml
# Obtidos via polling a cada 30s → publicados em iot.tv.state
ssap://com.webos.applicationManager/getForegroundAppInfo  # app ativo
ssap://audio/getStatus                                     # volume atual
ssap://tv/getExternalInputList                            # entradas disponíveis
ssap://system/getSystemInfo                               # modelo, firmware
ssap://com.webos.service.update/getCurrentSWInformation   # versão SW
```

---

## 🛠️ Setup na Rede

```yaml
# Recomendado: IP fixo por reserva DHCP no roteador (via MAC da TV)
# Settings → Network → Wired/Wi-Fi → Advanced → ver MAC address
# No roteador: reservar ex: 192.168.1.50 para esse MAC

# Para Wake-on-LAN funcionar:
# Settings → General → Additional Settings → Instant Game Response (OFF)
# Settings → General → Turn off TV → habilitar WoL (em alguns modelos: "Rede em Standby")
```

---

## 📦 Dependências Python

```txt
aiowebostv>=0.5.0      # cliente WebOS async (WSS nativo)
wakeonlan>=2.1.0       # para ligar via Wake-on-LAN
nats-py>=2.6.0         # client NATS
pyyaml>=6.0            # leitura do devices.yaml
```

---

---

# 📺 TV #2 — Samsung Tizen *(planejado)*

> Seção reservada. Protocolo: WebSocket porta 8001. Biblioteca: `samsungtvws`.

---

# 📺 TV #3 — DLNA Genérico *(planejado)*

> Seção reservada. Fallback UPnP/SOAP para qualquer TV com DLNA habilitado.

---

## 🔄 Changelog

### v1.1.0
- ✅ Documentação completa LG webOS (SSL, pairing, todos os comandos)
- ✅ Estrutura por seção de TV para futuras marcas

### v1.0.0
- ✅ Interface NATS unificada `iot.tv.*`
- ✅ Arquitetura adapter pattern

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
