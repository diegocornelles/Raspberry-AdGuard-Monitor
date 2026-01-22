# 🛡️ Torre de Vigilancia: AdGuard Home + Prometheus + Grafana

> **Convierte tu Raspberry Pi en un centro de ciberseguridad doméstico.**

Este proyecto despliega un stack completo de monitorización y bloqueo de publicidad. No solo limpia el tráfico de tu red (PCs, móviles, Smart TV) eliminando anuncios y rastreadores, sino que te ofrece un panel de control profesional para visualizar todo lo que ocurre en tu casa en tiempo real.

![Vista Previa del Dashboard](images/dashboard-preview.png)

## 📋 ¿Qué consigue este proyecto?

1.  **Navegación Limpia:** Elimina banners, pop-ups y vídeos de publicidad en toda la red WiFi/Cable.
2.  **Privacidad:** Bloquea la telemetría de Windows, rastreadores de móviles y espionaje de Smart TVs.
3.  **Velocidad:** Las webs cargan un **30-40% más rápido** al no descargar basura publicitaria.
4.  **Control Total:** Un dashboard (Grafana) que muestra qué dispositivos consumen más y qué dominios son bloqueados.

---

## 🏗️ La Arquitectura (El Stack)

* **Hardware:** Raspberry Pi 3 B+ (o superior) con Raspberry Pi OS Lite.
* **DNS & Bloqueo:** [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome).
* **Base de Datos:** Prometheus (Series temporales).
* **Exportador:** AdGuard-Exporter (Versión HenryWhitaker3).
* **Visualización:** Grafana.

---

## 🚀 Guía de Instalación Paso a Paso

### 1. Preparativos Previos
* Tener la Raspberry Pi conectada por cable (recomendado) al router.
* Tener una **IP Estática** configurada (ej: `192.168.1.213`).
* Acceso por terminal (SSH).

### 2. Instalar y Configurar AdGuard Home

**A. Instalación:**
Ejecuta el script oficial en tu terminal:
```bash
curl -s -S -L [https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh](https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh) | sh -s -- -v
```

**B. Configuración Inicial:**

1.  Abre tu navegador y entra en `http://TU_IP_RASPBERRY:3000`.
2.  Sigue el asistente. **Importante:** Cuando pregunte por el puerto del panel de administración, elige el puerto **80** (o 8080 si tienes otro servidor web). El puerto DNS debe ser el **53**.
3.  Crea tu usuario y contraseña.

---

**C. Activar la "Lista Maestra" (OISD Big):**

Para bloquear anuncios difíciles, necesitamos una lista potente.

1.  Entra al panel (`http://TU_IP_RASPBERRY`).
2.  En el menú superior, ve a **Filtros** > **Listas de bloqueo DNS**.
3.  Haz clic en el botón verde **"Añadir lista de bloqueo"**.
4.  Elige la pestaña/botón **"Añadir una lista personalizada"**.
5.  Rellena los datos:
    * **Nombre:** `OISD Big`
    * **URL:** `https://big.oisd.nl`
6.  Haz clic en **Guardar**. (Espera unos segundos a que descargue las reglas).

---

## 3. Instalar el Sistema de Monitorización

### A. Instalar Prometheus (Base de datos)

```bash
sudo apt update
sudo apt install prometheus prometheus-node-exporter -y
```
**B. Instalar AdGuard Exporter (El traductor):**

Descargamos la versión compatible con Raspberry Pi (ARM):

```bash
# Descargar
wget https://github.com/henrywhitaker3/adguard-exporter/releases/download/v1.2.1/adguard-exporter_1.2.1_linux_armv6.tar.gz

# Descomprimir
tar -xvf adguard-exporter_1.2.1_linux_armv6.tar.gz

# Mover a la carpeta de usuario y dar permisos
mv adguard-exporter /home/pi/
chmod +x /home/pi/adguard-exporter
```

---

**C. Crear el servicio automático:**

Para que el exportador arranque siempre, creamos un servicio.

Ejecuta:

```bash
sudo nano /etc/systemd/system/adguard-exporter.service
```

Pega el siguiente contenido (⚠️ Cambia `TU_USUARIO` y `TU_CONTRASEÑA` por los de AdGuard):

```ini
[Unit]
Description=AdGuard Home Exporter
After=network.target

[Service]
User=root
Environment="ADGUARD_SERVERS=http://127.0.0.1"
Environment="ADGUARD_USERNAMES=TU_USUARIO"
Environment="ADGUARD_PASSWORDS=TU_CONTRASEÑA"
Environment="INTERVAL=10s"
Environment="LOG_LIMIT=10000"
ExecStart=/home/pi/adguard-exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

Guarda (Ctrl+O, Enter) y sal (Ctrl+X).

Activa el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl start adguard-exporter
sudo systemctl enable adguard-exporter
```

---

**D. Conectar Prometheus al Exportador:**

Edita la config de Prometheus:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Al final del archivo, añade esto (cuidado con la indentación):

```yaml
  - job_name: 'adguard'
    static_configs:
      - targets: ['localhost:9618']
```

Reinicia:

```bash
sudo systemctl restart prometheus
```

---

## 4. Instalar y Configurar Grafana (El Panel)

**A. Instalación:**

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install grafana -y

# Arrancar Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

---

**B. Configuración Visual:**

Entra en `http://TU_IP_RASPBERRY:3000` (Usuario: `admin` / Pass: `admin`).

**Añadir Fuente de Datos:**

1. Ve al menú lateral **Connections** > **Data sources**.
2. Click en **"Add new data source"**.
3. Selecciona **Prometheus**.
4. En **"Prometheus server URL"** escribe: `http://localhost:9090`.
5. Baja al final y dale a **"Save & test"**.

**Importar el Dashboard:**

1. Ve al menú **Dashboards** > **New** > **Import**.
2. Escribe el ID: `20799` y dale a **Load**.
3. Selecciona tu fuente **"Prometheus"** abajo y dale a **Import**.

---

**C. (Opcional) Truco para Iconos:**

Si quieres que aparezcan iconos (🎮 Steam, 🟥 Youtube) en los gráficos:

1. Edita el gráfico **"Top Queries"**.
2. Ve a la pestaña **Transformations**.
3. Añade **"Rename by regex"**.
4. Ejemplo:
   - **Regex:** `.*steam.*`
   - **Replacement:** `🎮 Steam`
