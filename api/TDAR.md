Here is a detailed plan to extract and integrate **TDAR (The Digital Archaeological Record)** data into your system for local use. TDAR provides access to a large collection of archaeological datasets, including excavation records, artifact data, and more. This guide helps automate the download process, store the data locally, and integrate it into your system.

---

# **TDAR Data Extraction and Integration Plan**

### **Overview**
The purpose of this plan is to outline the steps for extracting, downloading, and storing TDAR data locally and integrating it into your system for analysis, visualization, and research. TDAR offers archaeological datasets in various formats, and this plan will help you automate the extraction, set up a local database for storage, and create a syncing mechanism for updating the data.

### **Objectives**
- Download TDAR datasets for local use.
- Set up a local database to store and manage TDAR data.
- Automate syncing to ensure the local database is updated with new data.
- Integrate TDAR data into your system for analysis and visualization.
- Provide proper attribution when using TDAR data.

---

## **Step 1: Accessing TDAR Data**

TDAR provides datasets that can be accessed via their **web interface** or **API** (if available), allowing you to download individual datasets for local use.

### **1.1: Browsing and Downloading Datasets**

1. **Browse Datasets**:
   - Go to the **TDAR website**: [https://www.tdar.org/](https://www.tdar.org/).
   - Use the search functionality to find relevant datasets. You can search by archaeological project, site, excavation, or object.

2. **Download the Dataset**:
   - TDAR may provide datasets in **CSV**, **JSON**, or **RDF** formats. Each dataset page will have a download button to get the data in a format that suits your needs.
   - If the dataset is available via direct download links, you can automate the download using tools like `wget` or `curl`.
   
   **Manual Download Example**:
   - For individual dataset downloads, navigate to a dataset page and click on the provided download link for the preferred format.

   **Automated Download Example** (if direct link is available):
   ```bash
   wget -O tdar-dataset-latest.csv https://www.tdar.org/path-to-dataset.csv
   ```

3. **Request Access (If Required)**:
   - Some TDAR datasets may require access requests or permissions. Ensure that you have the proper credentials or permissions to access and download the data.

---

## **Step 2: Setting Up a Local Database for TDAR Data**

Once you have downloaded the data, the next step is to store it in a local database for querying and integration into your system.

### **2.1: Create a PostgreSQL Database (with PostGIS for Geospatial Data)**

1. **Create a New PostgreSQL Database**:
   - You will need a PostgreSQL database to store and manage the TDAR data.
   ```bash
   createdb tdardb
   ```

2. **Enable PostGIS Extension (if the data contains geospatial information)**:
   - If TDAR datasets contain geographic information (e.g., site coordinates), enable the PostGIS extension to handle spatial queries.
   ```sql
   CREATE EXTENSION postgis;
   ```

3. **Create a Table to Store TDAR Data**:
   - Define a schema to match the structure of the TDAR datasets you’re downloading. This will depend on the type of data (e.g., excavation records, artifact descriptions, etc.).

   **Example SQL Schema** (for excavation data):
   ```sql
   CREATE TABLE tdar_excavations (
       id SERIAL PRIMARY KEY,
       tdar_id VARCHAR(255),
       site_name VARCHAR(255),
       description TEXT,
       excavation_date DATE,
       latitude FLOAT,
       longitude FLOAT,
       metadata JSONB,
       modified TIMESTAMP
   );
   ```

---

### **2.2: Parse and Insert TDAR Data into the Database**

Once the dataset is downloaded (e.g., as CSV, JSON, or RDF), you can write a script to parse and insert the data into your database.

**Example Parsing Script (Python for CSV Data)**:

```python
import csv
import psycopg2

# Connect to PostgreSQL
conn = psycopg2.connect("dbname=tdardb user=yourusername password=yourpassword")
cur = conn.cursor()

# Load the CSV dataset
with open('tdar-dataset-latest.csv', newline='') as f:
    reader = csv.DictReader(f)
    for row in reader:
        tdar_id = row['tdar_id']
        site_name = row['site_name']
        description = row['description']
        excavation_date = row['excavation_date']
        latitude = float(row['latitude']) if row['latitude'] else None
        longitude = float(row['longitude']) if row['longitude'] else None
        modified = row['modified']

        # Insert into the PostgreSQL table
        cur.execute("""
            INSERT INTO tdar_excavations (tdar_id, site_name, description, excavation_date, latitude, longitude, modified)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (tdar_id) DO UPDATE
            SET site_name = EXCLUDED.site_name,
                description = EXCLUDED.description,
                excavation_date = EXCLUDED.excavation_date,
                latitude = EXCLUDED.latitude,
                longitude = EXCLUDED.longitude,
                modified = EXCLUDED.modified;
        """, (tdar_id, site_name, description, excavation_date, latitude, longitude, modified))

# Commit and close connection
conn.commit()
cur.close()
conn.close()
```

**Example Parsing Script (JSON Data)**:

```python
import json
import psycopg2

# Connect to PostgreSQL
conn = psycopg2.connect("dbname=tdardb user=yourusername password=yourpassword")
cur = conn.cursor()

# Load the JSON dataset
with open('tdar-dataset-latest.json', 'r') as f:
    data = json.load(f)
    
    for entry in data:
        tdar_id = entry.get('tdar_id')
        site_name = entry.get('site_name')
        description = entry.get('description')
        excavation_date = entry.get('excavation_date')
        latitude = entry.get('latitude')
        longitude = entry.get('longitude')
        metadata = json.dumps(entry)
        modified = entry.get('modified')

        # Insert into the PostgreSQL table
        cur.execute("""
            INSERT INTO tdar_excavations (tdar_id, site_name, description, excavation_date, latitude, longitude, metadata, modified)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (tdar_id) DO UPDATE
            SET site_name = EXCLUDED.site_name,
                description = EXCLUDED.description,
                excavation_date = EXCLUDED.excavation_date,
                latitude = EXCLUDED.latitude,
                longitude = EXCLUDED.longitude,
                metadata = EXCLUDED.metadata,
                modified = EXCLUDED.modified;
        """, (tdar_id, site_name, description, excavation_date, latitude, longitude, metadata, modified))

# Commit and close connection
conn.commit()
cur.close()
conn.close()
```

---

## **Step 3: Sync and Update TDAR Data**

To ensure that you always have the latest TDAR data, you need to implement a mechanism that periodically checks for updates and syncs new data to your local database.

### **3.1: Checking for Updates**

1. **HTTP HEAD Request to Check Last Modified Date**:
   - Use `HEAD` requests to check the **Last-Modified** header of the dataset before downloading it again. This helps you avoid downloading the same data multiple times.

   **Example (Python)**:
   ```python
   import requests

   url = "https://www.tdar.org/path-to-dataset.csv"
   response = requests.head(url)
   last_modified = response.headers.get('Last-Modified')
   print("Last Modified:", last_modified)
   ```

2. **Compare Last Modified Date**:
   - Store the last modified date of the dataset in a local file or database. Before downloading the dataset again, compare the stored last sync date with the `Last-Modified` header to decide whether to download the new version.

   **Python Example**:
   ```python
   last_sync_file = 'last_sync.txt'

   # Read the last sync date
   with open(last_sync_file, 'r') as f:
       last_sync = f.read().strip()

   # Check the dataset for updates
   response = requests.head("https://www.tdar.org/path-to-dataset.csv")
   last_modified = response.headers.get('Last-Modified')

   if last_modified > last_sync:
       # Download new dataset
       response = requests.get("https://www.tdar.org/path-to-dataset.csv")
       with open('tdar-dataset-latest.csv', 'wb') as f:
           f.write(response.content)

       # Update last_sync.txt
       with open(last_sync_file, 'w') as f:
           f.write(last_modified)
   else:
       print("No updates available.")
   ```

---

### **3.2: Automating the Sync Process**

1. **Set Up a Cron Job**:
   - Automate the sync process using a cron job that runs periodically to check for updates and download the new dataset.

   **Example Cron Job** (check for updates every week):
   ```bash
   crontab -e
   ```

   Add the following line to check for updates every Monday at 3 AM:
   ```bash


   0 3 * * 1 /path/to/sync_tdar.py
   ```

2. **Update the Local Database**:
   - When new data is detected, parse the dataset and update the local database with new or modified records. Use `INSERT ... ON CONFLICT` to update existing records or insert new ones.

---

## **Step 4: Integrating TDAR Data into Your System**

Once the TDAR data is stored locally, you can integrate it into your system for querying, analysis, and visualization.

### **4.1: Query TDAR Data for Archaeological Research**

You can use SQL queries to retrieve excavation data, artifact records, or site metadata from your local database.

- **Example Query**: Retrieve all excavation sites from a specific region.
   ```sql
   SELECT site_name, description, latitude, longitude
   FROM tdar_excavations
   WHERE latitude BETWEEN 30 AND 35
   AND longitude BETWEEN -90 AND -85;
   ```

- **Example Query**: Find all excavations conducted during a specific period.
   ```sql
   SELECT site_name, description, excavation_date
   FROM tdar_excavations
   WHERE excavation_date BETWEEN '1900-01-01' AND '1950-12-31';
   ```

### **4.2: Visualizing TDAR Data on a Map**

Use **Leaflet** or **OpenLayers** to visualize TDAR data on an interactive map.

- **Leaflet Example**:
   ```javascript
   var map = L.map('map').setView([35.0, -90.0], 5); // Example coordinates for the region
   L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

   // Add markers for excavation sites
   var marker = L.marker([35.0, -90.0]).addTo(map);
   marker.bindPopup("<b>Excavation Site</b><br>Description of the site.").openPopup();
   ```

---

## **Step 5: Attribution and Maintenance**

### **5.1: Provide Proper Attribution**

When using TDAR data, ensure that you follow TDAR’s licensing and attribution guidelines.

- **Attribution Example**:
   ```html
   <p>Data from <a href="https://www.tdar.org/">The Digital Archaeological Record (TDAR)</a>, courtesy of TDAR contributors.</p>
   ```

### **5.2: Monitor and Maintain the System**

1. **Monitor Sync Logs**:
   - Regularly monitor the cron job and script logs to ensure the sync process runs smoothly and catches any errors (e.g., network issues or database errors).

2. **Handle Data Updates**:
   - When updates are detected and new data is downloaded, ensure that records are either inserted as new entries or updated in the local database.

---

### **Conclusion**

This plan outlines the steps to extract, store, and integrate **TDAR** data into your system. By automating the download and syncing process, you can maintain an up-to-date local copy of the archaeological datasets, which you can use for research, analysis, or visualization. Always remember to provide proper attribution for the data in compliance with TDAR’s licensing terms.