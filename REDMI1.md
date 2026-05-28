# Full Professional OpenStreetMap (OSM) Setup with Docker and Renderd

This guide walks through a **production-ready setup** of OpenStreetMap using Docker, PostGIS, Mapnik, and renderd, including data download, tile caching, and troubleshooting.

---

## 1️⃣ Prerequisites

* A Linux server (Ubuntu 22.04 or similar)
* Docker & Docker Compose:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker --now

```

* Disk space:
* **Iran region:** ~200–300GB
* **Global:** Multiple TB


* Optional: `wget` and `curl` for downloading OSM data.

---

## 2️⃣ Project Directory Structure

```bash
mkdir -p ~/osm-docker/data
cd ~/osm-docker

```

```
osm-docker/
├─ data/
│  ├─ database/
│  ├─ tiles/
│  └─ style/
├─ cache/
├─ docker-compose.yml
└─ postgres-data/

```

---

## 3️⃣ Download OSM Data

### Iran Extract (example)

Use **Geofabrik** extracts:

```bash
cd ~/osm-docker/data
wget [https://download.geofabrik.de/asia/iran-latest.osm.pbf](https://download.geofabrik.de/asia/iran-latest.osm.pbf)

```

> Other regions or global extracts: [https://download.geofabrik.de/](https://download.geofabrik.de/)

---

## 4️⃣ Prepare Map Style

Download **OSM-Carto** style:

```bash
cd ~/osm-docker/data/style
git clone [https://github.com/gravitystorm/openstreetmap-carto.git](https://github.com/gravitystorm/openstreetmap-carto.git) .

```

* Ensure `mapnik.xml` is generated (via `carto project.mml` or Docker):

```bash
ls data/style/
# Output: mapnik.xml  openstreetmap-carto.style ...

```

> **Troubleshooting:** If `mapnik.xml` missing, run `carto project.mml > mapnik.xml`.

---

## 5️⃣ Docker Compose Setup

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:

  db:
    image: postgis/postgis:15-3.3
    container_name: osm_postgis
    restart: always
    environment:
      POSTGRES_USER: renderer
      POSTGRES_PASSWORD: rendererpass
      POSTGRES_DB: osm
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    ports:
      - "5433:5432"
    shm_size: '2gb'

  renderd:
    image: overv/openstreetmap-tile-server:latest
    container_name: osm_renderd
    depends_on:
      - db
    environment:
      THREADS: 8
      UPDATES: enabled
      NAME_LUA: openstreetmap-carto.lua
      NAME_STYLE: openstreetmap-carto.style
      NAME_MML: project.mml
      NAME_SQL: indexes.sql
      ALLOW_CORS: 1  # Enables built-in CORS configuration conditionally
    volumes:
      - ./data:/data
      - ./postgres-data:/var/lib/postgresql/data
      - ./cache/tiles:/var/cache/renderd/tiles
      - ./tiles:/var/lib/mod_tile
    ports:
      - "80:80"
      - "443:443"
    command: run
    restart: always

  importer:
    image: overv/openstreetmap-tile-server:latest
    container_name: osm_importer
    depends_on:
      - db
    environment:
      THREADS: 8
      UPDATES: enabled
    volumes:
      - ./data:/data
      - ./postgres-data:/var/lib/postgresql/data
    command: import
    restart: "no"

```

---

## 6️⃣ Fix Tile Cache Permissions

```bash
sudo mkdir -p ./cache/tiles
sudo chown -R 1000:1000 ./cache/tiles
sudo chmod -R 755 ./cache/tiles

```

> Docker containers must have **write access** to `/cache/tiles` to avoid `Permission denied`.

---

## 7️⃣ Import OSM Data & Run Services

```bash
docker-compose run --rm importer
docker-compose up -d

```

* Check logs for errors:

```bash
docker logs -f osm_renderd
docker logs -f osm_postgis

```

---

## 8️⃣ Pre-generate Tiles

* If the map appears gray, pre-generate tiles:

```bash
docker exec -it osm_renderd bash
render_list --bbox=44.0,24.0,63.0,40.0 --zoom=0-18

```

* Adjust bounding box and zoom as needed.

> **Tip:** Tiles are cached automatically; this ensures smooth navigation and reduces server load.

---

## 9️⃣ Testing Tile Access

```bash
curl -I http://YOUR_SERVER_IP/tile/14/8500/5350.png
# Expected: HTTP 200 OK

```

> **Troubleshooting:** Missing tiles in `/cache/tiles/default/...` are normal until renderd generates them.

---

## 🔧 Troubleshooting Notes

* **Permission Denied** → Fix ownership of cache folder.
* **Gray map** → Tiles not yet rendered.
* **SVG PARSING ERROR** → Safe to ignore.
* Tiles may not appear immediately on first request; pre-generation is recommended for popular regions.

### 🌐 Hot-Fixing CORS Issues (Zero-Downtime)

If frontend applications face CORS blocks (missing `Access-Control-Allow-Origin` headers) while requesting tiles, you can permanently and unconditionally inject the header **directly inside the running container** without restarting the service or causing downtime:

1. **Enter the running container's shell:**
```bash
docker exec -it osm_renderd /bin/bash

```


2. **Install `nano` text editor internally:**
```bash
apt-get update && apt-get install -y nano

```


3. **Modify the Apache VirtualHost configuration:**
```bash
nano /etc/apache2/sites-enabled/000-default.conf

```


4. **Replace the file content with the full updated configuration below:**
```apache
<VirtualHost *:80>
    Header set Server ""
    ServerAdmin webmaster@localhost

    AddTileConfig /tile/ default
    LoadTileConfigFile /etc/renderd.conf
    ModTileRenderdSocketName /run/renderd/renderd.sock
    ModTileRequestTimeout 0
    ModTileMissingRequestTimeout 30

    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # CORS Header injected directly for seamless frontend integration
    Header set Access-Control-Allow-Origin "*"
</VirtualHost>

```


*Save and exit (`Ctrl+O` -> `Enter` -> `Ctrl+X`).*
5. **Reload Apache configuration smoothly (Zero-Downtime):**
```bash
apachectl -k graceful

```


*This will apply changes instantly without dropping active connections.*
6. **Exit the container:**
```bash
exit

```



---

## 🔄 Updates & Maintenance

* Tile cache can be updated incrementally via **OSM replication**.
* For full updates, consider re-importing every 6 months.
* Pre-generate tiles for heavily accessed areas to improve performance.

---

## 10️⃣ Optional: Clear & Rebuild Cache

```bash
docker-compose stop renderd
sudo rm -rf ./cache/tiles/*
docker-compose start renderd

```

---

## ✅ Conclusion

With this setup, you now have a **production-ready OSM stack**:

* Dockerized PostGIS and renderd
* Tile caching for high-speed zoom
* Ability to update map data regularly
* Full control over style and region

This setup is ready for **GitHub documentation** or deployment in production environments.

---

**References & Downloads:**

* OSM Data (Geofabrik): [https://download.geofabrik.de/](https://download.geofabrik.de/)
* OSM-Carto Style: [https://github.com/gravitystorm/openstreetmap-carto](https://github.com/gravitystorm/openstreetmap-carto)
* Docker OSM Tile Server: [https://hub.docker.com/r/overv/openstreetmap-tile-server](https://hub.docker.com/r/overv/openstreetmap-tile-server)

```

```
