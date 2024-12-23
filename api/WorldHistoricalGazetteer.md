To integrate the **World Historical Gazetteer (WHG)** datasets into your system for local use, you can follow a process to download and store all available data, set up a local database, and develop a method for querying and visualizing the data. Here’s a detailed step-by-step plan:

---

# **World Historical Gazetteer Data Extraction and Integration Plan**

### **Overview**
The **World Historical Gazetteer (WHG)** provides rich datasets of historical places and their associated metadata, including geographical coordinates, names, descriptions, and historical periods. This plan outlines how to extract the entire dataset from WHG, store it locally, and integrate it into your system for querying, visualization, or analysis.

### **Objectives**
- Extract and download the complete dataset from the World Historical Gazetteer.
- Set up a local database to store and query the historical geospatial data.
- Sync and update the data regularly if new datasets are available.
- Integrate the data into your system for querying, analysis, and visualization.
- Provide proper attribution when using the WHG data.

---

## **Step 1: Downloading World Historical Gazetteer Datasets**

### **1.1: Obtain the Dataset**
WHG offers its data as downloadable datasets, typically in **GeoJSON**, **CSV**, or **JSON-LD** formats. These can be accessed from the WHG site.

1. **Go to the WHG Download Page**:
   - Visit the official [World Historical Gazetteer website](https://whgazetteer.org/).
   - Navigate to their dataset download page, where you will find links to various dataset formats.

2. **Download the Dataset**:
   - If available, use `wget` or `curl` to automate the download of the dataset. For example:
     ```bash
     wget -O whg-dataset-latest.json https://whg.org/path-to-dataset.json
     ```
   - Choose **GeoJSON** if the data includes geospatial coordinates, or **CSV**/**JSON-LD** for non-geospatial metadata.

---

## **Step 2: Set Up a Local Database for WHG Data**

Once the dataset is downloaded, you need to store the data locally for querying and integration into your system.

### **2.1: Create a PostgreSQL Database with PostGIS**

WHG data is primarily geospatial, so using **PostgreSQL** with **PostGIS** extension for geospatial queries is recommended.

1. **Create a New PostgreSQL Database**:
   - Create a new database to store the WHG data.
   ```bash
   createdb whgdb
   ```

2. **Enable PostGIS Extension**:
   - Install and enable the PostGIS extension to handle geospatial queries.
   ```sql
   CREATE EXTENSION postgis;
   ```

3. **Create a Table to Store WHG Data**:
   - Define a table schema to store the relevant fields from the WHG dataset, such as place names, coordinates, descriptions, historical periods, and other metadata.

   **Example SQL Schema**:
   ```sql
   CREATE TABLE whg_places (
       id SERIAL PRIMARY KEY,
       whg_id VARCHAR(50),
       name VARCHAR(255),
       description TEXT,
       start_date VARCHAR(50),
       end_date VARCHAR(50),
       latitude FLOAT,
       longitude FLOAT,
       geojson_data JSONB,
       modified TIMESTAMP
   );
   ```

---

### **2.2: Parse and Insert WHG Data into the Database**

You can write a script to parse the downloaded dataset (GeoJSON, CSV, or JSON-LD) and insert the relevant data into your local PostgreSQL database.

**Example Parsing Script (Python for GeoJSON):**

```python
import json
import psycopg2

# Connect to PostgreSQL
conn = psycopg2.connect("dbname=whgdb user=yourusername password=yourpassword")
cur = conn.cursor()

# Load the dataset (GeoJSON format)
with open('whg-dataset-latest.json', 'r') as f:
    data = json.load(f)
    for feature in data['features']:
        whg_id = feature['properties'].get('whg_id')
        name = feature['properties'].get('name')
        description = feature['properties'].get('description', '')
        start_date = feature['properties'].get('start_date')
        end_date = feature['properties'].get('end_date')
        coordinates = feature['geometry']['coordinates']
        lat, lon = coordinates if coordinates else (None, None)
        geojson_data = json.dumps(feature['geometry'])
        modified = feature['properties'].get('modified')

        # Insert into the database
        cur.execute("""
            INSERT INTO whg_places (whg_id, name, description, start_date, end_date, latitude, longitude, geojson_data, modified)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (whg_id) DO UPDATE
            SET name = EXCLUDED.name,
                description = EXCLUDED.description,
                start_date = EXCLUDED.start_date,
                end_date = EXCLUDED.end_date,
                latitude = EXCLUDED.latitude,
                longitude = EXCLUDED.longitude,
                geojson_data = EXCLUDED.geojson_data,
                modified = EXCLUDED.modified;
        """, (whg_id, name, description, start_date, end_date, lat, lon, geojson_data, modified))

# Commit the transaction and close the connection
conn.commit()
cur.close()
conn.close()
```

**Example Parsing Script (Python for CSV):**
```python
import csv
import psycopg2

# Connect to PostgreSQL
conn = psycopg2.connect("dbname=whgdb user=yourusername password=yourpassword")
cur = conn.cursor()

# Load the dataset (CSV format)
with open('whg-dataset-latest.csv', newline='') as f:
    reader = csv.DictReader(f)
    for row in reader:
        whg_id = row['whg_id']
        name = row['name']
        description = row['description']
        start_date = row['start_date']
        end_date = row['end_date']
        latitude = float(row['latitude']) if row['latitude'] else None
        longitude = float(row['longitude']) if row['longitude'] else None
        modified = row['modified']

        # Insert into the database
        cur.execute("""
            INSERT INTO whg_places (whg_id, name, description, start_date, end_date, latitude, longitude, modified)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (whg_id) DO UPDATE
            SET name = EXCLUDED.name,
                description = EXCLUDED.description,
                start_date = EXCLUDED.start_date,
                end_date = EXCLUDED.end_date,
                latitude = EXCLUDED.latitude,
                longitude = EXCLUDED.longitude,
                modified = EXCLUDED.modified;
        """, (whg_id, name, description, start_date, end_date, latitude, longitude, modified))

# Commit the transaction and close the connection
conn.commit()
cur.close()
conn.close()
```

---

## **Step 3: Sync and Update WHG Data**

To ensure your system always has the latest data, you need to periodically check for updates from the WHG dataset and sync your local copy accordingly.

### **3.1: Check for Updates**

1. **HTTP HEAD Request**:
   - Use `HEAD` requests to check the **Last-Modified** header of the dataset file before downloading it again. This will let you know if there is any new data available.

   **Example (Python)**:
   ```python
   import requests

   url = "https://whg.org/path-to-dataset.json"
   response = requests.head(url)
   last_modified = response.headers.get('Last-Modified')
   print("Last Modified:", last_modified)
   ```

2. **Compare the Last Sync Date**:
   - Store the date of the last successful sync in a local file or database.
   - Compare this date with the `Last-Modified` header before downloading the new dataset.

   **Example for Storing Last Sync Date**:
   ```python
   last_sync_file = 'last_sync.txt'

   # Read the last sync date
   with open(last_sync_file, 'r') as f:
       last_sync = f.read().strip()

   # Check the dataset for updates
   response = requests.head("https://whg.org/path-to-dataset.json")
   last_modified = response.headers.get('Last-Modified')

   if last_modified > last_sync:
       # Download the new dataset and update the local database
       response = requests.get("https://whg.org/path-to-dataset.json")
       with open('whg-dataset-latest.json', 'wb') as f:
           f.write(response.content)

       # Update the last sync file
       with open(last_sync_file, 'w') as f:
           f.write(last_modified)
   else:
       print("No updates available.")
   ```

### **3.2: Automate the Sync Process**

1. **Set Up a Cron Job**:
   - Automate the sync process using a cron job that checks for updates and downloads the dataset regularly. For example, to check for updates every week:
   ```bash
   crontab -e
   ```

   Add the following line to run the sync script every Monday at 2 AM:
   ```bash
   0 2 * * 1 /path/to/sync_whg.py
   ```

2. **Sync and Update Local Data**:
   - When new data is available, parse the updated dataset and update the local PostgreSQL database, either by inserting



 new records or updating existing ones.

---

## **Step 4: Integrating WHG Data into Your System**

Once the WHG data is stored locally, you can integrate it into your system for querying and visualization.

### **4.1: Query WHG Data for Historical Places**

You can use SQL queries to retrieve information about historical places and their geospatial data.

- **Example Query**: Retrieve all places associated with a specific historical period.
   ```sql
   SELECT name, description, start