# 🎬 Episodio 4 — Tu propio Netflix casero

> Continuando desde donde lo dejamos — ya tienes Ubuntu Server y CasaOS instalado. En este episodio montamos un servidor multimedia completo para ver películas y series desde cualquier lugar.

📺 **[Ver el video en YouTube](#)** ← _(link próximamente)_

---

## 🧩 ¿Qué vamos a instalar?

| Servicio | Para qué sirve |
|----------|---------------|
| **Jellyfin** | Tu reproductor multimedia, el "Netflix" |
| **Jellyseerr** | Para pedir películas/series fácilmente |
| **Sonarr** | Descarga y organiza series automáticamente |
| **Radarr** | Descarga y organiza películas automáticamente |
| **Prowlarr** | Conecta los buscadores con Sonarr y Radarr |
| **qBittorrent** | El que hace las descargas |

Al terminar podrás pedir una película desde tu celular y aparecerá sola en Jellyfin lista para ver. 🍿

---

## ✅ Requisitos

- Haber visto los episodios anteriores ([Ep.1](/Episodio_1) · [Ep.2](/Episodio_2) · [Ep.3](/Episodio_3))
- Ubuntu Server instalado y funcionando
- CasaOS instalado
- Acceso por SSH a tu servidor

---

## 📁 Parte 1 — Las carpetas de CasaOS

CasaOS ya crea automáticamente la estructura de carpetas que vamos a usar. No necesitas crear nada. Así luce en tu servidor:

```
/DATA/
├── AppData/        ← configuraciones de cada app (CasaOS las gestiona)
├── Media/
│   ├── Movies/     ← aquí van las películas
│   └── TV Shows/   ← aquí van las series
└── Downloads/      ← carpeta de descargas temporales
```

Solo necesitamos crear la carpeta de descargas si no existe aún:

```bash
mkdir -p /DATA/Downloads
```

---

## 🐳 Parte 2 — Levantar el stack con Docker Compose

Creamos un solo archivo que levanta todos los servicios de una vez:

```bash
nano /DATA/docker-compose.yml
```

Pega este contenido:

```yaml
version: "3.8"

services:

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - /DATA/AppData/jellyfin:/config
      - /DATA/Media/Movies:/data/peliculas
      - /DATA/Media/TV Shows:/data/series

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    restart: unless-stopped
    ports:
      - "5055:5055"
    volumes:
      - /DATA/AppData/jellyseerr:/app/config
    environment:
      - TZ=America/Lima

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    ports:
      - "8989:8989"
    volumes:
      - /DATA/AppData/sonarr:/config
      - /DATA/Media/TV Shows:/series
      - /DATA/Downloads:/downloads
    environment:
      - TZ=America/Lima

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    ports:
      - "7878:7878"
    volumes:
      - /DATA/AppData/radarr:/config
      - /DATA/Media/Movies:/movies
      - /DATA/Downloads:/downloads
    environment:
      - TZ=America/Lima

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    ports:
      - "9696:9696"
    volumes:
      - /DATA/AppData/prowlarr:/config
    environment:
      - TZ=America/Lima

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    volumes:
      - /DATA/AppData/qbittorrent:/config
      - /DATA/Downloads:/downloads
    environment:
      - TZ=America/Lima
      - WEBUI_PORT=8080
```

> ⚠️ Si tu zona horaria no es Lima, cámbiala. Puedes buscar la tuya [aquí](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

Ahora levantamos todo:

```bash
cd /DATA
docker compose up -d
```

Verifica que todos estén corriendo:

```bash
docker compose ps
```

Deberías ver 6 servicios con estado `running`. 🟢

> **¿Los ves en CasaOS?** Ve al panel — debería detectar los contenedores automáticamente y mostrarlos en la interfaz.

---

## ⚙️ Parte 3 — Configurar los servicios

> Reemplaza `TU-IP` con la IP de tu servidor en tu red (ej: `192.168.1.114`).  
> Si no la sabes: `ip addr show | grep inet`

### qBittorrent — `http://TU-IP:8080`

1. Obtén la contraseña temporal de los logs:
   ```bash
   docker logs qbittorrent | grep password
   ```
2. Inicia sesión y ve a **Herramientas → Opciones → Carpeta de descarga**
3. Establece `/downloads` como carpeta de descarga
4. Cambia la contraseña en **Opciones → Interfaz Web**

---

### Prowlarr — `http://TU-IP:9696`

1. Completa el setup inicial y crea tu usuario
2. Ve a **Indexers → Add Indexer** y agrega tus buscadores (ej: YTS, 1337x)
3. Deja abierto — volveremos aquí para conectar con Sonarr y Radarr

---

### Radarr (películas) — `http://TU-IP:7878`

1. Ve a **Settings → General** y copia tu **API Key**
2. Ve a **Settings → Media Management → Add Root Folder** → agrega `/movies`
3. Ve a **Settings → Download Clients → Add → qBittorrent**:
   - Host: `qbittorrent` · Puerto: `8080`
   - Usuario y contraseña de qBittorrent
4. En Prowlarr → **Settings → Apps → Add → Radarr** → pega la API Key

---

### Sonarr (series) — `http://TU-IP:8989`

Igual que Radarr pero:
- Carpeta raíz: `/series`
- En Prowlarr → Apps → Add → **Sonarr**

---

### Jellyfin — `http://TU-IP:8096`

1. Sigue el asistente y crea tu usuario administrador
2. Agrega tus bibliotecas:
   - **Películas** → `/data/peliculas` (apunta a `/DATA/Media/Movies`)
   - **Series** → `/data/series` (apunta a `/DATA/Media/TV Shows`)
3. Inicia el escaneo — Jellyfin descargará portadas y metadata solo

---

### Jellyseerr — `http://TU-IP:5055`

1. En el setup selecciona **Iniciar sesión con Jellyfin**
2. URL de Jellyfin: `http://jellyfin:8096`
3. Inicia sesión con tu cuenta de Jellyfin
4. Conecta con Sonarr y Radarr usando sus API Keys

¡Listo! Desde Jellyseerr puedes pedir contenido y llegará solo a Jellyfin. 🎉

---

## 📱 Parte 4 — Acceder desde tu celular con Tailscale

Tailscale te permite conectarte a tu servidor desde cualquier lugar — con datos móviles o un WiFi externo — de forma segura y sin configurar nada en tu router.

### En el servidor

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Aparecerá una URL — ábrela en tu navegador para autenticar con tu cuenta de Tailscale. Luego obtén tu IP:

```bash
tailscale ip
```

Guarda ese número (algo como `100.x.x.x`).

### En tu celular

1. Descarga **Tailscale** desde Play Store o App Store
2. Inicia sesión con la misma cuenta
3. Activa la VPN

Ya puedes acceder desde cualquier lugar:

| Servicio | URL con Tailscale |
|----------|-------------------|
| Jellyfin | `http://100.x.x.x:8096` |
| Jellyseerr | `http://100.x.x.x:5055` |

---

## 🔁 Comandos útiles

```bash
# Ver estado de todos los contenedores
docker compose ps

# Ver logs de un servicio específico
docker logs nombre-del-servicio

# Reiniciar un servicio
docker restart nombre-del-servicio

# Actualizar todos los contenedores a la última versión
cd /DATA
docker compose pull
docker compose up -d
```

---

## 📺 Episodios anteriores

| Episodio | Tema |
|----------|------|
| [Episodio 1](/Episodio_1) | Preparar la laptop e instalar Ubuntu Server |
| [Episodio 2](/Episodio_2) | Primeros pasos y configuración base |
| [Episodio 3](/Episodio_3) | Instalar CasaOS |

---

> ⭐ Si te fue útil, dale una estrella al repo — ayuda mucho al canal.  
> 💬 ¿Dudas? Deja un comentario en el video o abre un issue aquí en GitHub.
