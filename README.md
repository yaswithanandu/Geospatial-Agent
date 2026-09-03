# Geospatial Agent 

An AI agent that turns natural-language requests into real geospatial analyses. Ask it something like *"Find suitable areas for affordable housing in Palo Alto"* and it plans a workflow, pulls live elevation/weather/OSM/land-use data, runs GIS operations (buffering, clipping, reclassification, weighted overlay, multi-criteria analysis), and publishes the result as a map layer — all with a full audit trail of what it did and why.

The project has two parts:
- **Backend** (`gra_project/backend`) — a Django + LangChain agent that plans and executes GIS tool calls
- **Frontend** (`gra_project/geospatial-analysis-platform`) — a Next.js/React map UI (Leaflet) for uploading data, drawing regions of interest, and viewing results

An earlier, simpler prototype lives in `gra-backend/` (a standalone FastAPI-style agent without the Django app, threads, or streaming execution).

## ✨ Key Features

- **Natural-language planning**: an LLM (via Groq) turns a query into a step-by-step tool-call plan with reasoning for each step
- **Live data acquisition**:
  - Copernicus DEM elevation data (AWS Earth Search STAC catalog)
  - Open-Meteo weather data (temperature, precipitation)
  - ISRO Bhuvan vector layers (WFS)
  - OpenStreetMap features via OSMnx
- **GIS analysis tools**: buffering, clipping, reclassification, raster math, weighted overlay / multi-criteria analysis, area calculation, and more, built on WhiteboxTools, GeoPandas, and Rasterio
- **Region of interest (ROI) support**: clip any acquired dataset to a user-drawn ROI
- **Threaded workflows**: conversations/analyses are grouped into threads, each with its own messages and output layers
- **Streaming execution**: results and progress stream back to the client via Server-Sent Events
- **Interactive map frontend**: draw ROIs, upload data, and visualize raster/vector outputs on a Leaflet map
- **GeoServer integration** (optional): publish final rasters as WMS layers

## 🏗️ Architecture

```
User query
   │
   ▼
Planner (LangChain + Groq LLM)  ──►  WorkflowPlan (ordered ToolCall steps + reasoning)
   │
   ▼
Executor (Django views, streamed via SSE)
   │
   ▼
Tool Registry (agent_app/tools.py)
   ├─ acquire_* tools  → live data sources (STAC, Open-Meteo, Bhuvan, OSM)
   ├─ analysis tools   → buffer, clip, reclassify, overlay, MCA, area calc
   └─ publish tool      → GeoServer WMS publishing
   │
   ▼
Frontend map UI (Next.js + Leaflet) renders outputs and lets users draw ROIs
```

## 📁 Repository Structure

```
gra_project/
├── backend/                         # Django backend
│   ├── manage.py
│   ├── requirements.txt
│   ├── IMPLEMENTATION_GUIDE.md      # Deep-dive on tools, API, and workflow
│   ├── config/                      # Django project settings/urls
│   └── agent_app/
│       ├── agent.py                 # LangChain planner (LLM + tool schemas)
│       ├── tools.py                 # Tool registry: data acquisition + GIS analysis
│       ├── views.py                 # Upload / Plan / Execute / Stream / Threads endpoints
│       ├── urls.py
│       ├── models.py                # ROI and thread/message models
│       └── geoserver_utils.py       # GeoServer publishing + WMS proxy
│
└── geospatial-analysis-platform/    # Next.js frontend
    ├── app/                         # App router pages
    ├── components/                  # Map, ROI drawing, UI components
    └── package.json

gra-backend/                         # Earlier standalone prototype (agent.py, tools.py, main.py)
```

## 🔧 Prerequisites

- Python 3.10+
- Node.js 18+ and npm/pnpm
- A [Groq API key](https://console.groq.com/) (used to run the planning LLM)
- (Optional) A running [GeoServer](https://geoserver.org/) instance for publishing final maps

## 🚀 Getting Started

### 1. Backend setup

```bash
cd gra_project/backend
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file in `gra_project/backend/` with:

```
GROQ_API_KEY=your_groq_api_key_here
SECRET_KEY=your_django_secret_key      # optional, falls back to a dev key
DEBUG=True
```

Run migrations and start the server:

```bash
python manage.py migrate
python manage.py runserver
```

The API will be available at `http://localhost:8000/api/`.

### 2. Frontend setup

```bash
cd gra_project/geospatial-analysis-platform
npm install       # or pnpm install
npm run dev
```

The app will be available at `http://localhost:3000`.

### 3. GeoServer

To enable publishing of final analysis rasters as WMS layers, run GeoServer locally on `localhost:8080` with the default `admin`/`geoserver` credentials. If GeoServer isn't running, publishing calls fail gracefully rather than blocking the workflow.

## 📡 API Overview

All endpoints are served under `/api/` (see `agent_app/urls.py`):

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/upload/` | POST | Upload a data file for use in an analysis |
| `/api/plan/` | POST | Generate a step-by-step workflow plan from a natural-language query |
| `/api/execute/` | POST | Execute a previously generated plan |
| `/api/execute/stream/` | GET | Execute a plan and stream progress via Server-Sent Events |
| `/api/threads/` | GET/POST | List or create analysis threads |
| `/api/threads/<id>/messages/` | GET | Get messages for a thread |
| `/api/threads/<id>/layers/` | GET | Get output layers for a thread |
| `/api/wms-proxy/` | GET | Proxy requests to a GeoServer WMS endpoint |

For a detailed breakdown of the underlying tools (data acquisition + analysis functions, their parameters, and example calls), see [`gra_project/backend/IMPLEMENTATION_GUIDE.md`](gra_project/backend/IMPLEMENTATION_GUIDE.md).

### Example workflow

For the query *"Find suitable housing areas in Palo Alto"*, the agent might plan:

1. Acquire schools (positive suitability factor)
2. Acquire noisy venues like bars (negative factor)
3. Acquire elevation data (flat areas preferred)
4. Run a weighted multi-criteria analysis combining the layers
5. Publish the resulting suitability raster to GeoServer

## 🧪 Testing

```bash
cd gra_project/backend
python test_implementation.py       # Test individual tool components
python test_django_views.py         # Test Django endpoints (server must be running)
python test_agent_fixes.py
python validate_fixes.py
```

## 🐛 Troubleshooting

- **Import errors**: double-check `pip install -r requirements.txt` completed, especially geospatial packages (`geopandas`, `rasterio`, `pystac-client`, `stackstac`, `rioxarray`, `whitebox`)
- **DEM/STAC issues**: verify internet access and that the requested place can be geocoded; some areas lack DEM coverage
- **Open-Meteo issues**: the API is free but rate-limited
- **Bhuvan issues**: the service may be temporarily down, or the place may be outside India
- **GeoServer issues**: confirm it's running on `localhost:8080` with default credentials; the app will continue without map publishing if it's unavailable

## 🛠️ Tech Stack

**Backend**: Django, Django REST Framework, LangChain, langchain-groq, GeoPandas, Rasterio, WhiteboxTools, OSMnx, pystac-client, stackstac, rioxarray, PySAL

**Frontend**: Next.js 14, React 18, TypeScript, Tailwind CSS, Leaflet / react-leaflet, Radix UI

## 📄 License

No license file is currently included in this repository. Add one (e.g. MIT, Apache-2.0) if you intend to share or open-source this project.
