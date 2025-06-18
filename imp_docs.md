> ### _There's a whole  bunch of methods/ways I've tried, but I simply cannot get a TiTiler app in a Conda environment work completely offline on an air-gapped system, with COGs hosted locally on the system, and fetched via a WMS endpoint and rendered in a Leaflet/Jinja template on a browser. I later found out, after much pain, that the CDN links to various (almost all) static files/assets (css/js libs/modules) that I had downloaded and hosted locally on the same system (even gave the absolute paths to the offline app in the template) simply refused to work because they were hardcoded into the default TiTIler pip package (and other dependencies) site-packages location. Pretty fed up with this. Oh, and I've already tried most well-known alternatives already (Terracotta, GeoServer/MapServer, Dash-Plotly, Mapbox/Maplibre, etc.). I do not want an XYZ tile server._

#### To resolve your offline TiTiler deployment challenges on an air-gapped system, follow this structured approach that overrides CDN dependencies and ensures full offline functionality with local COGs and Leaflet rendering:

### Core Solution Strategy
1. **Bypass CDN Hardcoding**  
   Override TiTiler's built-in templates to replace CDN URLs with local paths.
2. **Localize Static Assets**  
   Download all required libraries and serve them from your app's static directory.
3. **Custom WMS Endpoint Integration**  
   Configure Leaflet to use TiTiler's WMS endpoint with local COG paths.

---

### Step-by-Step Implementation

#### 1. **Directory Structure Setup**
```bash
your_project/
├── static/
│   ├── leaflet/               # Downloaded Leaflet assets
│   ├── openlayers/            # Downloaded OpenLayers assets
│   └── titiler_overrides/     # Custom JS/CSS for template fixes
├── templates/
│   ├── titiler/               # Overridden TiTiler templates
│   │   ├── wms.html           # Modified WMS template
│   │   └── ...                # Other templates as needed
├── cogs/                      # Local COG storage
├── app.py                     # Main FastAPI app
└── config.py                  # Configuration settings
```

#### 2. **Download Static Assets (On Internet-Connected Machine)**
```bash
# Leaflet (v1.9.4 example)
wget https://unpkg.com/leaflet@1.9.4/dist/leaflet.zip
unzip leaflet.zip -d static/leaflet

# OpenLayers (v8.2.0 example)
wget https://github.com/openlayers/openlayers/releases/download/v8.2.0/dist.zip
unzip dist.zip -d static/openlayers

# TiTiler dependency overrides (if needed)
wget https://example.com/custom_titiler_fix.js -P static/titiler_overrides/
```

#### 3. **Override TiTiler Templates**
Create `templates/titiler/wms.html`:
```jinja
<!DOCTYPE html>
<html>
<head>
  <title>Local WMS Viewer</title>
  <!-- Load local Leaflet instead of CDN -->
  <link rel="stylesheet" href="{{ url_for('static', filename='leaflet/leaflet.css') }}">
  <script src="{{ url_for('static', filename='leaflet/leaflet.js') }}"></script>
  
  <!-- Add custom fixes for TiTiler -->
  <script src="{{ url_for('static', filename='titiler_overrides/custom_fix.js') }}"></script>
  
  <style>
    #map { height: 100vh; }
  </style>
</head>
<body>
  <div id="map"></div>
  <script>
    var map = L.map('map').setView([0, 0], 2);
    
    // Local WMS Endpoint (serving COGs from /cogs)
    L.tileLayer.wms("{{ url_for('wms_endpoint') }}", {
      layers: 'my_cog_layer',  // Your COG layer name
      format: 'image/png',
      transparent: true,
      attribution: 'Local COG'
    }).addTo(map);
  </script>
</body>
</html>
```

#### 4. **Configure FastAPI App (`app.py`)**
```python
from fastapi import FastAPI, Request
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from titiler.core.factory import TilerFactory
from titiler.core.errors import DEFAULT_STATUS_CODES, add_exception_handlers
import os

app = FastAPI(title="Air-Gapped TiTiler")

# Mount local assets
app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

# Override TiTiler's default template paths
app.state.template_directory = os.path.join("templates", "titiler")

# Setup COG Tiler with local filesystem
cog = TilerFactory(
    reader_options={"env": {"GDAL_DISABLE_READDIR_ON_OPEN": "EMPTY_DIR"}},
    router_prefix="/cogs"
)
app.include_router(cog.router)

# Custom WMS endpoint handler
@app.get("/wms/viewer", include_in_schema=False)
async def wms_viewer(request: Request):
    return templates.TemplateResponse(
        "wms.html",
        {"request": request, "wms_endpoint": app.url_path_for("wms")},
    )

# Critical: Disable external network calls
os.environ["TITILER_CACHE_DISABLE"] = "True"  # Disable tile caching
os.environ["NO_PROXY"] = "*"                  # Block all external HTTP

add_exception_handlers(app, DEFAULT_STATUS_CODES)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, loop="none")
```

#### 5. **Environment Configuration (`config.py`)**
```python
# Disable all external connections
import socket
socket.socket = None  # Block network sockets

# Force GDAL to use local paths
import os
os.environ["CPL_VSIL_CURL_ALLOWED_EXTENSIONS"] = ""  # Disable remote reads
os.environ["GDAL_DISABLE_READDIR_ON_OPEN"] = "EMPTY_DIR"
```

#### 6. **Run with Strict Offline Enforcement**
```bash
# Block network access at system level (Linux example)
sudo iptables -A OUTPUT -d 0.0.0.0/0 -j DROP

# Start the app (with network blocked)
python app.py
```

---

### Key Mitigation Strategies
1. **Template Overrides**  
   Explicitly bypass TiTiler's CDN references by overriding templates in `templates/titiler/`.

2. **Network Hardening**  
   - Block outbound traffic at OS level (`iptables`)
   - Disable GDAL's HTTP capabilities via env variables
   - Nullify Python's socket module

3. **COG Path Handling**  
   Use `file://` paths when registering COGs:
   ```python
   # When adding COGs to mosaic
   "files": ["file:///abs/path/to/cogs/your_cog.tif"]
   ```

4. **Dependency Audit**  
   Verify all wheels in Conda env are installed with `--no-deps` to prevent hidden CDN calls.

---

### Verification Steps
1. Check for CDN references in rendered HTML:
   ```bash
   curl http://localhost:8000/wms/viewer | grep '\.js'
   ```
2. Validate GDAL is only accessing local files:
   ```python
   # Add debug route in app.py
   @app.get("/debug")
   def debug():
       return {"env": dict(os.environ)}
   ```
3. Test with invalid URLs to confirm blocked external access.

This solution ensures all resources are constrained to the local filesystem while maintaining TiTiler's WMS functionality with Leaflet. The approach eliminates hidden external dependencies through aggressive environment hardening and explicit asset localization.

---

> ### _I should've told you - this is an air-gapped Windows 10 Pro system._

#### Got it. Deploying TiTiler offline on Windows 10 Pro requires a different approach for network hardening and path handling. Here's the revised solution tailored for Windows:

### Windows-Specific Solution for Air-Gapped TiTiler Deployment

#### 1. **Directory Structure (Windows Paths)**
```bash
C:\air_gapped_app\
├── static\
│   ├── leaflet\          # Contains leaflet.css, leaflet.js, etc.
│   ├── openlayers\       # OpenLayers assets
│   └── titiler_overrides\
├── templates\
│   ├── titiler\
│   │   └── wms.html      # Custom template
├── cogs\                 # Your COG files
├── app.py
└── config.py
```

#### 2. **Network Hardening (Windows Specific)**
```python
# config.py - Windows network lockdown
import os
import sys

# Disable all external network access
os.environ["NO_PROXY"] = "*"
os.environ["GDAL_DISABLE_READDIR_ON_OPEN"] = "EMPTY_DIR"
os.environ["CPL_CURL_ENABLED"] = "NO"

# Block socket connections
if sys.platform == 'win32':
    import socket
    def guard(*args, **kwargs):
        raise RuntimeError("Network access blocked in air-gapped environment")
    socket.socket = guard
```

#### 3. **Windows Path Handling in `app.py`**
```python
from pathlib import Path
from fastapi import FastAPI, Request
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from titiler.core.factory import TilerFactory

app = FastAPI(title="Offline TiTiler (Windows)")

# Windows path handling
BASE_DIR = Path(__file__).parent.resolve()

# Static files config
static_dir = BASE_DIR / "static"
app.mount("/static", StaticFiles(directory=static_dir), name="static")

# Templates
templates_dir = BASE_DIR / "templates"
templates = Jinja2Templates(directory=templates_dir)

# Override TiTiler templates
app.state.template_directory = str(templates_dir / "titiler")

# COG Tiler with local filesystem support
cog = TilerFactory(
    reader_options={
        "env": {
            "GDAL_DISABLE_READDIR_ON_OPEN": "EMPTY_DIR",
            "GDAL_HTTP_MERGE_CONSECUTIVE_RANGES": "YES",
            "GDAL_HTTP_MULTIPLEX": "YES",
            "GDAL_HTTP_VERSION": "2",
            "VSI_CACHE": "TRUE"
        }
    },
    router_prefix="/cogs"
)
app.include_router(cog.router)

# Custom WMS viewer
@app.get("/wms/viewer", response_class=HTMLResponse)
async def wms_viewer(request: Request):
    return templates.TemplateResponse(
        "titiler/wms.html",
        {
            "request": request,
            "wms_endpoint": request.url_for("wms"),
            "leaflet_css": "/static/leaflet/leaflet.css",
            "leaflet_js": "/static/leaflet/leaflet.js"
        }
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### 4. **Modified `wms.html` Template (Windows Path Compatible)**
```jinja
<!DOCTYPE html>
<html>
<head>
    <title>Local COG Viewer</title>
    <link rel="stylesheet" href="{{ leaflet_css }}">
    <script src="{{ leaflet_js }}"></script>
    <style>
        #map { height: 100vh; }
    </style>
</head>
<body>
    <div id="map"></div>
    <script>
        // Windows-compatible absolute path to COG
        const cogPath = "file:///C:/air_gapped_app/cogs/your_raster.tif";
        
        var map = L.map('map').setView([0, 0], 2);
        
        L.tileLayer.wms("{{ wms_endpoint }}", {
            layers: cogPath,
            format: 'image/png',
            transparent: true,
            version: '1.3.0',
            attribution: 'Local COG'
        }).addTo(map);
    </script>
</body>
</html>
```

#### 5. **Windows Firewall Configuration**
1. Press `Win + R` and type `wf.msc` to open Firewall
2. Create new **Outbound Rules**:
   - Block all connections for Python executable:
     - Program path: `C:\Python3X\python.exe` (your Python path)
     - Action: Block all connections
   - Repeat for `uvicorn.exe` if used separately

#### 6. **Offline Package Installation**
1. On connected machine:
```powershell
pip download titiler[full] -d .\offline_packages
pip download uvicorn fastapi jinja2 -d .\offline_packages
```
2. Copy packages to air-gapped system
3. Install offline:
```powershell
pip install --no-index --find-links=.\offline_packages titiler[full] uvicorn fastapi jinja2
```

#### 7. **GDAL Windows Configuration**
Create `gdal-data` directory in your app folder and set environment variable:
```python
# In app.py before creating TilerFactory
os.environ["GDAL_DATA"] = str(BASE_DIR / "gdal-data")
```

#### 8. **Running the Application**
```powershell
# In PowerShell
$env:PROJ_LIB = "C:\air_gapped_app\proj-data"
$env:GDAL_DATA = "C:\air_gapped_app\gdal-data"
python app.py
```

### Key Windows-Specific Adjustments:
1. **Path Handling**:
   - Use `pathlib` for Windows path compatibility
   - `file:///` URLs with drive letters (`file:///C:/path/to/cog.tif`)
   - Raw string paths for Windows backslashes

2. **Network Lockdown**:
   - Socket-level blocking in Python
   - Windows Firewall outbound rules
   - GDAL environment variables to disable HTTP

3. **Static Assets**:
   - Serve via FastAPI's static files at `/static` endpoint
   - Absolute paths in templates using `url_for()`

4. **GDAL Dependencies**:
   - Bundle `gdal-data` and `proj-data` directories
   - Set `GDAL_DATA` and `PROJ_LIB` environment variables

### Verification Commands (PowerShell):
```powershell
# Check listening ports
netstat -ano | Select-String ":8000"

# Test WMS endpoint (in another PowerShell):
(Invoke-WebRequest "http://localhost:8000/cogs/wms?service=WMS&request=GetCapabilities").Content > capabilities.xml

# Check for CDN leaks
Get-Content .\templates\titiler\wms.html | Select-String "http[s]?://"
```

This solution addresses Windows-specific path handling, network lockdown through both application-level and system firewall controls, and proper GDAL configuration for offline COG access. The key is the combination of Python socket blocking, Windows Firewall rules, and GDAL environment configuration to ensure complete offline operation.

* * *

* * *

* * *
