Here’s a full integration plan for syncing and implementing **Pleiades** data into your system, complete with steps for setting up regular updates, checking for new data before downloading, and properly managing and visualizing the data. You can save this document for reference or project documentation.

---

# **Pleiades Data Integration and Sync Plan**

### **Overview**
This document outlines the steps required to integrate and sync data from the **Pleiades Gazetteer** into your project. Pleiades provides a comprehensive dataset of ancient places and locations, particularly focusing on the Greco-Roman world. The goal is to automate the process of checking for updates and downloading new or updated data while maintaining a local database for efficient querying and analysis.

### **Objectives**
- Automate the process of syncing data from Pleiades.
- Periodically check if there is new data available.
- Download and store Pleiades data locally in a database.
- Update the local database only when new or updated data is available.
- Provide attribution according to Pleiades’ license.

---

## **Step 1: Download and Analyze Initial Data**
First, you need to download the latest version of the Pleiades dataset and set up your local database.

### **Download Pleiades Data**
Pleiades provides downloadable datasets in various formats, including JSON, RDF, and KML. For this integration, we’ll use the **JSON** dataset as it's easy to manipulate and parse.

1. Go to the [Pleiades downloads page](https://pleiades.stoa.org/downloads).
2. Download the **Pleiades Places JSON** file.

   ```bash
   wget https://atlantides.org/downloads/pleiades/json/pleiades-places-latest.json
   ```

### **Initial Data Import**
Set up a local **PostgreSQL** database (or any database of your choice) with **PostGIS** enabled for geospatial queries.

1. **Create a database and enable PostGIS**:
   ```sql
   CREATE DATABASE pleiadesdb;
   \c pleiadesdb;
   CREATE EXTENSION postgis;
   ```

2. **Create a table to store Pleiades data**:
   You will need to define a table schema to store the relevant fields from the Pleiades dataset.

   ```sql
   CREATE TABLE pleiades_places (
       id SERIAL PRIMARY KEY,
       pleiades_id VARCHAR(50),
       title VARCHAR(255),
       description TEXT,
       latitude FLOAT,
       longitude FLOAT,
       accuracy VARCHAR(50),
       location_type VARCHAR(255),
       modified TIMESTAMP,
       geojson JSONB
   );
   ```

3. **Import data**:
   Write a script to parse the JSON file and insert it into the database. Example using Python:

   ```python
   import json
   import psycopg2

   conn = psycopg2.connect("dbname=pleiadesdb user=yourusername password=yourpassword")
   cur = conn.cursor()

   with open('pleiades-places-latest.json') as f:
       data = json.load(f)
       for place in data:
           pleiades_id = place.get('id')
           title = place.get('title')
           description = place.get('description')
           geo = place.get('reprPoint')
           lat, lon = geo if geo else (None, None)
           modified = place.get('modified')
           location_type = place.get('placeTypes')
           accuracy = place.get('accuracy')
           geojson = json.dumps(place.get('geometry'))

           cur.execute("""
               INSERT INTO pleiades_places (pleiades_id, title, description, latitude, longitude, accuracy, location_type, modified, geojson)
               VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
               """, (pleiades_id, title, description, lat, lon, accuracy, location_type, modified, geojson))
           
   conn.commit()
   cur.close()
   conn.close()
   ```

### **Initial Setup Complete**
At this point, you have downloaded the Pleiades dataset and stored it in a local database for further querying and use.

---

## **Step 2: Set Up Data Syncing and Update Check**

### **Check for New Data**
Pleiades updates their dataset periodically. To avoid downloading the entire dataset every time, you can set up a system to check for the modification timestamp and download only if new data is available.

### **Create a Sync Script**
We’ll automate the process of checking for new data, downloading it, and updating the database.

1. **Check for File Modification**:  
   Pleiades’ downloadable datasets contain a header with metadata, including the modification timestamp. You can compare the file’s modification time against your last sync date.

2. **Fetch the Last Modified Date**:
   Use the HTTP `HEAD` request to check the file’s last modified date before downloading.

   **Example in Python**:
   ```python
   import requests

   pleiades_url = "https://atlantides.org/downloads/pleiades/json/pleiades-places-latest.json"
   response = requests.head(pleiades_url)
   last_modified = response.headers.get('Last-Modified')

   print("Last Modified:", last_modified)
   ```

3. **Store and Compare Last Sync Date**:
   Store the last sync date in a local file or database. Each time the script runs, compare the `Last-Modified` date from the HTTP header to the stored date. If the dataset has been updated, download the new data.

   ```python
   with open('last_sync.txt', 'r') as f:
       last_sync = f.read().strip()

   if last_modified > last_sync:
       # Download new data
       response = requests.get(pleiades_url)
       with open('pleiades-places-latest.json', 'wb') as f:
           f.write(response.content)

       # Update the last sync file
       with open('last_sync.txt', 'w') as f:
           f.write(last_modified)
   ```

### **Automating the Sync with Cron**
Set up a **cron job** (on Linux) to run the sync script periodically. For example, to check for updates every week:

```bash
crontab -e
```

Add the following line:

```bash
0 3 * * 1 /path/to/sync_pleiades.py
```

This cron job will run every Monday at 3 AM to check for new updates.

---

## **Step 3: Updating the Local Database**

Once new data is detected and downloaded, update the local database with the changes.

1. **Parse the Updated Data**:
   Re-run the parsing and insertion script from Step 1. Be sure to update existing entries or insert new ones if the place ID doesn't exist in the database.

2. **Update Script Example**:
   Modify your script to check if a place already exists, and if it does, update its data:

   ```python
   cur.execute("""
       INSERT INTO pleiades_places (pleiades_id, title, description, latitude, longitude, accuracy, location_type, modified, geojson)
       VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
       ON CONFLICT (pleiades_id) DO UPDATE
       SET title = EXCLUDED.title,
           description = EXCLUDED.description,
           latitude = EXCLUDED.latitude,
           longitude = EXCLUDED.longitude,
           accuracy = EXCLUDED.accuracy,
           location_type = EXCLUDED.location_type,
           modified = EXCLUDED.modified,
           geojson = EXCLUDED.geojson;
   """, (pleiades_id, title, description, lat, lon, accuracy, location_type, modified, geojson))
   ```

   This `ON CONFLICT` clause ensures that existing records are updated while new ones are inserted.

---

## **Step 4: Data Visualization and Use**

Once the data is stored locally, you can visualize and use it in your application.

1. **Display the Data on a Map**:
   Use **Leaflet** or **OpenLayers** to display Pleiades places on a map. For example, you can load the data from your PostgreSQL/PostGIS database and display ancient sites on an interactive map.

   Example using Leaflet (JavaScript):
   ```javascript
   var map = L.map('map').setView([41.89, 12.49], 5); // Rome coordinates
   L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

   // Add a marker for an ancient place
   var marker = L.marker([41.89, 12.49]).addTo(map);
   marker.bindPopup("<b>Rome</b>").openPopup();
   ```

2. **Provide Attribution**:
   As per the **Creative Commons Attribution 3.0 License**, you must give appropriate credit to Pleiades.

   Example Attribution:
   ```html
   <p>Data from <a href="https://pleiades.stoa.org/">Pleiades</a>, courtesy of The Institute for the Study of the Ancient World (ISAW).</p>
   ```

---

## **Step 5: Monitor and Maintain**

1. **Regular Updates**:
   Ensure your cron job is running correctly, and monitor the logs to confirm the database is updated with the latest data.

2. **Error Handling**:
   Implement error handling in your sync script to catch issues like network failures or database errors. Log these errors for debugging.

---

### **Conclusion**

By following this plan, you can automate the process of syncing Pleiades data with your

 local system, ensuring that you always have the latest information on ancient places. You can now use the synced data for visualization, research, or integration into your applications. Don't forget to provide proper attribution when using the data in public projects.