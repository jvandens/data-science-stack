# Quick GIS Demo Stack - Portainer Deployment

**Purpose:** 1-hour demo, then tear down. Minimal config, all defaults.

## Deploy in Portainer

1. **Portainer** → **Stacks** → **Add Stack**
2. Name: `gis-demo`
3. **Web editor** → Paste `demo-stack-portainer.yml`
4. **Deploy the stack**
5. Wait 2-3 minutes for images to pull

## Access Services

| Service | URL | Login |
|---------|-----|-------|
| **JupyterLab** | http://your-ip:8888 | No password |
| **VS Code** | http://your-ip:8444 | Password: `demo` |
| **Superset** | http://your-ip:8088 | admin / admin |
| **Tile Server** | http://your-ip:7800 | N/A |
| **Feature Server** | http://your-ip:9001 | N/A |
| **PostGIS** | your-ip:5432 | demo / demo123 |

## Quick Demo Script (15 minutes)

### 1. Load Sample Data (JupyterLab - 3 min)

Open http://your-ip:8888, create new notebook:

```python
!pip install geoalchemy2

import geopandas as gpd
from shapely.geometry import Point
import psycopg2

# Create test data
data = {
    'name': ['San Francisco', 'New York', 'London'],
    'population': [874961, 8336817, 8982000],
    'geometry': [
        Point(-122.4194, 37.7749),
        Point(-74.0060, 40.7128),
        Point(-0.1278, 51.5074)
    ]
}

gdf = gpd.GeoDataFrame(data, crs='EPSG:4326')

# Connect with psycopg2 directly
conn = psycopg2.connect(
    host='postgis',
    database='demo',
    user='demo',
    password='demo123'
)
cur = conn.cursor()

# Create table
cur.execute("""
    DROP TABLE IF EXISTS demo_points;
    CREATE TABLE demo_points (
        name VARCHAR(100),
        population INTEGER,
        geometry GEOMETRY(POINT, 4326)
    );
""")

# Insert data
for idx, row in gdf.iterrows():
    cur.execute("""
        INSERT INTO demo_points (name, population, geometry)
        VALUES (%s, %s, ST_GeomFromText(%s, 4326))
    """, (row['name'], row['population'], row.geometry.wkt))

# Create spatial index
cur.execute("CREATE INDEX demo_points_geom_idx ON demo_points USING GIST (geometry);")

conn.commit()
cur.close()
conn.close()

print(f"✓ Created {len(gdf)} cities in PostGIS")
print(f"✓ View at: http://your-ip:7800")
```

### 2. View Vector Tiles (Browser - 2 min)

Open: http://your-ip:7800

You'll see `public.countries` listed. Click to preview tiles!

Tile URL: `http://your-ip:7800/public.countries/{z}/{x}/{y}.pbf`

### 3. Query Features API (Browser - 2 min)

Open: http://your-ip:9001

Click "collections" → "public.countries"

Get GeoJSON: http://your-ip:9001/collections/public.countries/items?limit=10

### 4. Build Dashboard (Superset - 5 min)

1. Open http://your-ip:8088 (admin/admin)
2. **Settings** → **Database Connections** → **+ Database**
3. Select PostgreSQL
4. SQLAlchemy URI: `postgresql://demo:demo123@postgis:5432/demo`
5. **Connect**
6. Create dataset from `countries` table
7. Create chart (map visualization)

### 5. DuckDB Analytics (JupyterLab - 3 min)

New cell in notebook:

```python
import duckdb

# Initialize DuckDB with spatial
con = duckdb.connect()
con.execute("INSTALL spatial; LOAD spatial;")

# Register GeoPandas dataframe
con.execute("CREATE TABLE countries AS SELECT * FROM world")

# Fast analytics
result = con.execute("""
    SELECT continent, 
           COUNT(*) as country_count,
           SUM(pop_est) as total_population
    FROM countries
    GROUP BY continent
    ORDER BY total_population DESC
""").df()

print(result)
```

## Demo Talking Points

**PostGIS**: "Enterprise spatial database, handles billions of features"

**JupyterLab**: "Data scientists love this - Python + GeoPandas for analysis"

**pg_tileserv**: "Modern vector tiles - automatic from PostGIS tables, no config"

**pg_featureserv**: "RESTful GeoJSON API - also automatic from PostGIS"

**Superset**: "BI dashboards for executives who don't code"

**DuckDB**: "10-100x faster than PostGIS for analytics - reads Parquet directly"

**VS Code**: "For developers writing ETL scripts"

## Advanced Demo (if time)

### Show ArcGIS Pro Connection

1. ArcGIS Pro → **Insert** → **Connections** → **New Database Connection**
2. PostgreSQL, host: your-ip, database: demo, user: demo, password: demo123
3. Browse `public.countries` → Add to map
4. "Works seamlessly with your existing tools"

### Show Web Map

Create `map.html` in VS Code:

```html
<!DOCTYPE html>
<html>
<head>
    <link href='https://unpkg.com/maplibre-gl@3.6.2/dist/maplibre-gl.css' rel='stylesheet' />
    <script src='https://unpkg.com/maplibre-gl@3.6.2/dist/maplibre-gl.js'></script>
    <style>body { margin: 0; } #map { height: 100vh; }</style>
</head>
<body>
    <div id="map"></div>
    <script>
        const map = new maplibregl.Map({
            container: 'map',
            style: {
                version: 8,
                sources: {
                    countries: {
                        type: 'vector',
                        tiles: ['http://YOUR-IP:7800/public.countries/{z}/{x}/{y}.pbf'],
                        minzoom: 0,
                        maxzoom: 14
                    }
                },
                layers: [{
                    id: 'countries',
                    type: 'fill',
                    source: 'countries',
                    'source-layer': 'countries',
                    paint: {
                        'fill-color': '#088',
                        'fill-opacity': 0.6
                    }
                }]
            },
            center: [0, 20],
            zoom: 2
        });
    </script>
</body>
</html>
```

Save and open in browser: "Vector tiles rendering in real-time"

## Tear Down

In Portainer:
1. **Stacks** → **gis-demo** → **Stop**
2. **Delete** (removes containers)
3. **Volumes** → Delete demo volumes (removes data)

Done! Everything cleaned up.

## Questions to Expect

**Q: "Does this scale?"**
A: "This is a single VM demo. Production uses Kubernetes, managed databases, load balancers. Same containers, different infrastructure."

**Q: "What about security?"**
A: "Demo has no auth. Production adds: OAuth/SSO, SSL/TLS, firewall rules, encrypted storage, audit logging."

**Q: "Can we use our existing GIS tools?"**
A: "Yes! ArcGIS Pro, QGIS, FME, R, Python - all connect to PostGIS. Modern APIs work with web apps."

**Q: "How much does it cost?"**
A: "Open source stack = $0 software. Cloud hosting ~$500-2000/month depending on scale. Replaces expensive per-seat licenses."

**Q: "How long to implement?"**
A: "Pilot like this: 1 week. Production with proper security/HA: 4-8 weeks. Migration from legacy: 3-6 months."
