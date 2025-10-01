## REST → Azure SQL with Azure Data Factory (Step-by-Step)

This guide shows how to build an end-to-end pipeline that pulls JSON from a REST API and loads it into Azure SQL Database using Azure Data Factory (ADF). It includes a mock API (Beeceptor), clean SQL schema, mapping, scheduling, verification, and troubleshooting.

### Architecture
[REST API (JSON)] ──GET──> [ADF Copy Activity] ──INSERT──> [Azure SQL Database]
              (Beeceptor)                     (AutoResolve IR)           (dbo.ApiReadings)
              
### Prerequisites
Azure subscription and permission to create:
- Azure Data Factory
+ Azure SQL Database (single DB on a SQL Server)
* SQL server networking: Allow Azure services and resources to access this server = On
- (Optional) Git repo for ADF definitions

### Mock your REST API (Beeceptor)
Create a Beeceptor endpoint (e.g. https://<subdomain>.free.beeceptor.com).
1. Add a GET rule for path /data with header:
2. Content-Type: application/json
3. Response body (sample):
```
[
  {"deviceId":"mx-001","ts":"2025-09-20T08:00:00Z","tempC":21.7,"humidity":44},
  {"deviceId":"mx-002","ts":"2025-09-20T08:05:00Z","tempC":22.4,"humidity":47}
]
```
4. Test
```
curl -i https://<subdomain>.free.beeceptor.com/data
```

### Create the SQL Table
In Azure Portal → SQL Database → Query editor, run:
```
CREATE TABLE dbo.ApiReadings (
  DeviceId     NVARCHAR(50)      NOT NULL,
  ReadingTime  DATETIMEOFFSET(0) NOT NULL, -- accepts ISO timestamps with trailing 'Z'
  Temperature  FLOAT             NULL,
  Humidity     FLOAT             NULL,
  CONSTRAINT PK_ApiReadings PRIMARY KEY (DeviceId, ReadingTime)
);
```
Using DATETIMEOFFSET avoids parsing errors on ...Z timestamps.

### Create Linked Services (ADF → Manage → Linked services)
1. REST
   - Type: REST
   - Base URL: https://<subdomain>.free.beeceptor.com/ (note the trailing slash)
   - Authentication: Anonymous
   - Test connection → Save
2. Azure SQL Database
   - Select your server/database (same DB where you created dbo.ApiReadings)
   - SQL auth or Entra ID
   - Test connection → Save
     
### Create Datasets (ADF → Author → + → Dataset)
1. Source dataset (REST + JSON)
   - Type: REST
   - Format: JSON
   - Linked service: the REST LS from step 4
   - Do not set a path here (we’ll pass it in the activity)
   - Schema: None (we’ll import on mapping)
2. Sink dataset (Azure SQL Table)
   - Type: Azure SQL Database
   - Linked service: your SQL LS
   - Table: dbo.ApiReadings
     
### Build the Pipeline (Copy Activity)
1. Author → + Pipeline → drag Copy data onto the canvas.
2. Source tab
   - Dataset: the REST JSON dataset
   - Request method: GET
   - Relative URL: data
     - If your Base URL ends with /, use data
     - If your Base URL has no trailing slash, use /data
   - (Advanced) JSON path: $ (your response is a root array)
   - Click Preview data → you should see rows.
3. Sink tab
   - Dataset: the Azure SQL dataset (dbo.ApiReadings)
   - (Default insert is fine for a POC)
4. Mapping tab
   - Click Clear → Import schemas
   - Ensure the mapping shows exactly:
     - deviceId → DeviceId
     - ts → ReadingTime
     - tempC → Temperature
     - humidity → Humidity
5. Click on Trigger now to trigger the pipeline. Confirm it succeeds.

### Images
## Architecture
![Screenshot 2025-09-30 142859](https://github.com/user-attachments/assets/f94bdacf-aaae-4146-8ebb-e8273a0b9488)

## ADF Platform
![Screenshot 2025-09-30 142455](https://github.com/user-attachments/assets/36d3cecc-e882-476a-9b9e-80103468786d)

## REST API
![Screenshot 2025-09-30 142408](https://github.com/user-attachments/assets/3b62a73b-7bf4-485d-97e7-3fe7754f38e0)

## SQL Database with Written Data
![Screenshot 2025-09-30 135851](https://github.com/user-attachments/assets/12c43ce0-01ec-4bf1-8135-eb189758314d)




