Here’s a detailed plan to extract and integrate **Heidelberg’s Epigraphic Database (EDH)** data into your system for local use. **EDH** provides epigraphic data, primarily focusing on inscriptions from the Roman Empire. The following steps cover downloading the data, setting up a local database, automating the extraction process, and syncing it regularly.

---

# **Heidelberg’s Epigraphic Database (EDH) Data Extraction and Integration Plan**

### **Overview**
This plan outlines the process of extracting and downloading **EDH** data, storing it locally, and integrating it into your system for querying, analysis, and visualization. EDH provides epigraphic datasets including inscriptions, geographical data, and metadata related to Roman epigraphy.

### **Objectives**
- Download EDH data for local use.
- Set up a local database to store and manage EDH inscriptions data.
- Automate syncing to ensure the local database stays updated with new data.
- Integrate EDH data into your system for research, visualization, and analysis.
- Provide proper attribution when using EDH data.

---

## **Step 1: Accessing Heidelberg’s Epigraphic Database (EDH) Data**

EDH offers a variety of ways to access data, including manual downloads, their **API**, and SPARQL endpoints. In this plan, we focus on accessing the API and setting up local storage for the data.

### **1.1: Accessing EDH Data via API**

EDH provides an API for programmatic access to inscriptions data. You can query specific inscriptions or download full datasets.

1. **EDH API Documentation**:
   - Review the EDH API documentation for full details on the available endpoints and query parameters: [EDH API Documentation](https://edh-www.adw.uni-heidelberg.de/documentation).

2. **API Endpoint Example**:
   You can use the API to fetch epigraphic data in **JSON** format.
   
   Example API query to fetch inscriptions by Roman province:
   ```bash
   https://edh-www.adw.uni-heidelberg.de/inscriptions/search?province=Gallia
   ```

3. **Automating the API Requests**:
   Write a script to automate data extraction from the EDH API using Python. You can fetch inscriptions for specific regions, time periods, or inscription types.

   **Example Python Script**:
   ```python
   import requests
   import json

   url = "https://edh-www.adw.uni-heidelberg.de/inscriptions/search?province=Gallia"
   response = requests.get(url)
   
   if response.status_code == 200:
       data = response.json()
       with open('edh-gallia-inscriptions.json', 'w') as f:
           json.dump(data, f)
   else:
       print(f"Error fetching data: {response.status_code}")
   ```

4. **Download Data for Specific Parameters**:
   Use different parameters such as province, period, or type to extract the datasets you need from the API. These datasets can be stored locally for further analysis and querying.

---

## **Step 2: Set Up a Local Database for EDH Data**

Once you have downloaded the data from the EDH API, the next step is to store it in a local database for querying and integration into your system.

### **2.1: Create a PostgreSQL Database (with PostGIS for Geospatial Data)**

EDH inscriptions often include geospatial data such as coordinates for where the inscriptions were found. For efficient querying and analysis, use **PostgreSQL** with **PostGIS**.

1. **Create a PostgreSQL Database**:
   ```bash
   createdb edhdb
   ```

2. **Enable PostGIS Extension**:
   If you plan to store geographic data (e.g., coordinates of the inscriptions), enable the PostGIS extension:
   ```sql
   CREATE EXTENSION postgis;
   ```

3. **Create a Table for Storing EDH Inscriptions Data**:
   Define a schema based on the data structure provided by the EDH API, including inscription text, coordinates, and metadata.

   **Example SQL Schema**:
   ```sql
   CREATE TABLE edh_inscriptions (
       id SERIAL PRIMARY KEY,
       edh_id VARCHAR(255),
       text TEXT,
       province VARCHAR(255),
       findspot VARCHAR(255),
       latitude FLOAT,
       longitude FLOAT,
       period_start INT,
       period_end INT,
       metadata JSONB,
       modified TIMESTAMP
   );
   ```

---

### **2.2: Parsing and Inserting EDH Data into the Database**

You can write a Python script to parse the downloaded EDH JSON data and insert it into your PostgreSQL database.

**Example Parsing Script (Python for EDH API Data)**:

```python
import json
import psycopg2

# Connect to PostgreSQL
conn = psycopg2.connect("dbname=edhdb user=yourusername password=yourpassword")
cur = conn.cursor()

# Load the JSON dataset
with open('edh-gallia-inscriptions.json', 'r') as f:
    data = json.load(f)

    for inscription in data:
        edh_id = inscription.get('id')
        text = inscription.get('text')
        province = inscription.get('province')
        findspot = inscription.get('findspot')
        latitude = inscription.get('latitude')
        longitude = inscription.get('longitude')
        period_start = inscription.get('date', {}).get('start', None)
        period_end = inscription.get('date', {}).get('end', None)
        metadata = json.dumps(inscription)
        modified = inscription.get('modified')

        # Insert into the PostgreSQL table
        cur.execute("""
            INSERT INTO edh_inscriptions (edh_id, text, province, findspot, latitude, longitude, period_start, period_end, metadata, modified)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (edh_id) DO UPDATE
            SET text = EXCLUDED.text,
                province = EXCLUDED.province,
                findspot = EXCLUDED.findspot,
                latitude = EXCLUDED.latitude,
                longitude = EXCLUDED.longitude,
                period_start = EXCLUDED.period_start,
                period_end = EXCLUDED.period_end,
                metadata = EXCLUDED.metadata,
                modified = EXCLUDED.modified;
        """, (edh_id, text, province, findspot, latitude, longitude, period_start, period_end, metadata, modified))

# Commit and close connection
conn.commit()
cur.close()
conn.close()
```

---

## **Step 3: Sync and Update EDH Data**

To keep your local data in sync with updates from EDH, you’ll want to set up a mechanism that checks for new data and updates your local database regularly.

### **3.1: Automating Data Syncing**

1. **Use the API’s Last-Modified Header**:
   - The EDH API provides a `Last-Modified` header, which indicates the last time the data was updated. You can use this to check if new data is available before downloading it.

   **Example to Check Last-Modified Date**:
   ```python
   import requests

   url = "https://edh-www.adw.uni-heidelberg.de/inscriptions/search?province=Gallia"
   response = requests.head(url)
   last_modified = response.headers.get('Last-Modified')
   print(f"Last Modified: {last_modified}")
   ```

2. **Store and Compare Last Sync Date**:
   - Store the last sync date in a local file or database. Before downloading the dataset again, compare the stored sync date with the `Last-Modified` date.

   **Example Script for Syncing Data**:
   ```python
   last_sync_file = 'last_sync.txt'

   # Read the last sync date
   with open(last_sync_file, 'r') as f:
       last_sync = f.read().strip()

   # Check for updates
   response = requests.head("https://edh-www.adw.uni-heidelberg.de/inscriptions/search?province=Gallia")
   last_modified = response.headers.get('Last-Modified')

   if last_modified > last_sync:
       # Download new data
       response = requests.get("https://edh-www.adw.uni-heidelberg.de/inscriptions/search?province=Gallia")
       with open('edh-gallia-inscriptions.json', 'wb') as f:
           f.write(response.content)

       # Update the last sync file
       with open(last_sync_file, 'w') as f:
           f.write(last_modified)
   else:
       print("No updates available.")
   ```

---

### **3.2: Automating Syncing with Cron Jobs**

Set up a cron job to automatically check for updates and download the data at regular intervals (e.g., every week).

**Example Cron Job** (sync every Monday at 3 AM):
```bash
crontab -e
```

Add the following line to the crontab file:
```bash
0 3 * * 1 /path/to/sync_edh.py
```

---

## **Step 4: Integrating EDH Data into Your System**

Once the EDH data is stored in your local database, integrate it into your system for research, querying, and visualization.

### **4.1: Query EDH Data for Research**

You can use SQL queries to retrieve epigraphic data for analysis or integration with other historical datasets.

- **Example Query**: Retrieve all inscriptions found in a specific province.
   ```sql
   SELECT text, findspot, latitude, longitude
   FROM edh_inscriptions
   WHERE province = 'Gallia';
   ``



`

- **Example Query**: Find inscriptions from a specific time period.
   ```sql
   SELECT text, period_start, period_end
   FROM edh_inscriptions
   WHERE period_start >= 100 AND period_end <= 300;
   ```

### **4.2: Visualizing EDH Data on a Map**

You can use **Leaflet** or **OpenLayers** to visualize the inscription data on a map.

- **Leaflet Example**:
   ```javascript
   var map = L.map('map').setView([48.8566, 2.3522], 5); // Centered on Europe
   L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

   // Add markers for inscriptions
   var marker = L.marker([48.8566, 2.3522]).addTo(map);
   marker.bindPopup("<b>Inscription</b><br>Description of the inscription.").openPopup();
   ```

---

## **Step 5: Attribution and Maintenance**

### **5.1: Provide Proper Attribution**

When using EDH data, make sure to provide proper attribution as per their licensing guidelines.

- **Attribution Example**:
   ```html
   <p>Data from <a href="https://edh-www.adw.uni-heidelberg.de/">Heidelberg’s Epigraphic Database (EDH)</a>, courtesy of the Heidelberg Academy of Sciences and Humanities.</p>
   ```

### **5.2: Monitor and Maintain the System**

1. **Monitor Sync Logs**:
   Regularly check logs from the sync script to ensure that data updates are applied correctly, and no errors occur during the syncing process.

2. **Handle Data Updates**:
   When new data is downloaded, ensure that it is inserted into the database, with updates applied to existing records.

---

### **Conclusion**

This plan provides a comprehensive approach to extracting and integrating data from **Heidelberg’s Epigraphic Database (EDH)** into your system. By automating the download and syncing process, you can ensure that your local database stays up-to-date with the latest epigraphic data. You can then query, analyze, and visualize this data for research or integration into broader historical projects.