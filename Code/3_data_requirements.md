# Data Requirement
The ETL process relies on a well-defined set of data. The data comes in a raw form and is processed by the data preparation stage, after which it is saved into input data in a form that is ready to be used by the subsequent steps within the overall ETL process. Additionally, there are reference tables that are defined for the ETL process to function properly. This section outlines the role of each of these data sources and how they are organised within the project folder structure in more detail. Below is a high-level folder structure that reflects how the data is stored under the project folder.

```shell
+-- Transit_Analytics_Tools_2024
|   +-- 1_raw_data
|   |   +-- 1_hastus
|   |   +-- 2_ gtfs
|   |   +-- 3_ticketing
|   +-- 2_input_data
|   |   +-- 1_reference_tables
|   |   +-- 2_corridor_definition
|   |   +-- 3_hastus
|   |   +-- 4_gtfs
|   |   +-- 5_transactions
|   |   +-- 6_trip_stop_timing
```
## Raw Data Sources

The raw data comes mainly from three different sources:

1. HASTUS data extracts including the itineraries, network and stop locations. These data are used to transfer the
   tarnsit metrics estimated at the stop to stop level to the underlying road network.
2. GTFS data. It is used to capture scheduled services within the analysis period.
3. Ticketing (transactions and trip stop timings report). This captures the details on the actual trips and is the main
   source for estimated load and estimated travel time that is reported at different level of aggregations in the final
   visulaisaitons.

The ETL process is designed to validate, read and transform the raw data into the input data. This process is heavily
relying on the data schema that is defined for each data item above. If any change to the content of the raw data is
detected, the schema for that data item needs to be updated accordingly. Below section provide more information on each
data item.

## Reference tables
Data from different sources may use varying conventions to refer to the same service details. For example, the direction names used in the Transactions and Trip Stop Timing Report may differ. Transactions might use Northbound, Southbound, Eastbound, or Westbound, while the Trip Stop Timing data might use North, South, East, and West. Similarly, operators name in the GTFS data may differ from those used in the ticketing data.

To combine scheduling data (GTFS) with ticketing data (transactions and trip stop timing data), reference tables are employed within the ETL process. These tables act as look-up mechanisms that align data from different sources. They also facilitate the exclusion of certain services or invalid ticket types efficiently and transparently. It is important to maintain the reference tables in line with the input ticketing data. This section details the role of the reference tables and highlights key checks to perform when new data sources become available.

Three main reference tables are used in the system, as outlined below:

1. Transaction to GTFS: This table is used to join the transaction data and GTFS data for the same route, direction, and operators, even if different names have been used in the two sources to indicate the same direction of travel or operator. Additionally, this table includes two columns that specify whether a service is a school service and whether it should be excluded from analysis. Temporary services, such as rail replacements or services introduced for specific events, are excluded from processing. This exclusion can be adjusted by updating the reference table. Table below shows the required fields in the transaction to gtfs reference file.

    | Attribute        | Description                                                                                   |
    |------------------|-----------------------------------------------------------------------------------------------|
    | operator         | The operator name as it's shown in transaction files                                          |
    | route            | This is the route short name as it appears in the transaction files                           |
    | direction        | This is the route direction as it appears in the transation files                             |
    | angency_id       | A numeric field corresponding the the agency id in the GTFS                                   |
    | route_short_name | A route short name as it appears in the GTFS                                                  |
    | direction_id     | direction id as it appears in GTFS corresponding the route direction                          |
    | type             | Indicating if a service is a school service or not                                            |
    | include          | A binary field indicating if the service should be included or excluded from further analysis |

2. Trip Stop Timing to GTFS: Similar to the Transaction to GTFS table, this table is used to join the trip stop timing data with the GTFS data for the same route, direction, and operators, despite differing nomenclature. It also includes columns to specify school services and whether a service should be excluded from analysis. Temporary services such as rail replacements or event-specific services are excluded from processing, and these exclusions can be modified by updating the reference table. The trip stop timing to gtfs reference file follows an identical structure as the transaction to gtfs reference table with the difference that the content of this file corresponds to the trip stop timing report instead of transactions. 
3. Ticket Status: This table is used to exclude transactions where the boardings or alightings are invalid based on the ticket types. Although invalid boardings or alightings are rare in a typical day, including them in the process could result in outliers in travel time estimation, which would adversely impact the aggregated metrics used in output visualisations. Therefore, it is crucial to exclude these records from the analysis. Table below shows the structure of this reference table: 

    | Attribute       | Description                                                                                             |
    |-----------------|---------------------------------------------------------------------------------------------------------|
    | ticket_status   | ticket_status as shown in the transactions                                                              |
    | boarding_valid  | A binary field indicating if the corresponding ticket status is valid to be used as a boarding record.  |
    | alighting_valid | A binary field indicating if the corresponding ticket status is valid to be used as a alighting record. |

## Input Data

The Data Preparation module is responsible for transforming raw data into input data, ensuring that the necessary transformations and data cleaning processes are applied. The module contains specific classes for each type of data, and the following outlines the high-level steps involved in preparing the input data from raw sources.

1. Full Network Data Preparation

    The raw network data is provided at three different resolution levels (low, medium, and high) in shape file format. Each layer represents underlying spatial road network (at different resolution level) across the entire state. But in its raw form, one spatial object is defined for both directions, and the directionality of that single spatial object is based on how the spatial object was drawn, not on the actual direction of travel. 
    The data preparation stage combines these three different resolutions networks into a single dataset, and corrects the directionality by creating a separate link per direction for the entire network. Additionally, the `LENGTH` attributes of the links are updated to reflect the correct spatial objects' length.
    The final processed data is stored in Parquet format, which is significantly more efficient for both processing and storage compared to raw spatial  file formats like shapefiles.
    
    The `PrepareFullNetwork` class within the data preparation module is responsible for preforming the abovementioned preparation step. The directional corrections of the links are performed as follows: 
    - In the raw network files, the direction of the link either To Destination (that is, the link is drawn in the direction of travel for a one way link), To Origin (that is, the link is drawn against the direction of travel for a one way link. This is less common), or in Both directions (for two-way links) is indicated in `FLOW` attributes as shown in table below.
    
       | FLOW | Direction      | Description                                        |
       |------|----------------|----------------------------------------------------|
       | OD   | To Destination | from origin node to destination                    |
       | ODB  | To Destination | from origin node to destination (except for bikes) |
       | DO   | To Origin      | from destination node to origin (except for bikes) |
       | DOB  | To Origin      | from destination node to origin (except for bikes) |
       | Null | Both           | both directions                                    |
    
    The data preparation stage corrected the direction of spatial objects as following:
    - it reverses the direction of links with 'SEG_FLOW' values equal to 'DO', 'DOB' to correct the link directions and show the direction of travel. 
    - It also generates a new spatial object for two-way links, so that the end result contains one link per direction. 
    - Links with 'SEG_FLOW' values equal to 'OD', 'ODB' remains unchanged as the direction in which they are drawn is the same the direction of travel. 
   Table blow shows attributes of the raw network files that is used with the Transit Analytics Tools' workflow.

| Attribute | Description                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ID        | Identifies a unique link within the HASTUS system. These links IDs comes from HASTUS network source (Street Pro Nav) which over time has been modified. New links created within HASTUS has a prefix of "GIRO" and a same link ID in two different versions of the HASTUS extract may differ, especially when an old link is split.In such cases, part of the link retains the original ID, while the other part(s) receive a new ID prefixed with “GIRO. |
| Flow      | The direction of the link that is, the direction in which the link is drawn.                                                                                                                                                                                                                                                                                                                                                                              |
| NODE_O    | This is the origin node number. Similar to the ID field above, there may be inconsistencies for new links between in HASTUS. New nodes do not appear to be created when links are split and there are several records with null Node IDs. New or split links with a GIRO prefix generally have 0 as the Node identifier.                                                                                                                                  |
| NODE_D    | This is the destination node number. Similar to the ID field above, there may be inconsistences for new links between in HASTUS. New nodes do not appear to be created when links are split and there are several records with null Node IDs. New or split links with a GIRO prefix generally have 0 as the Node identifier.                                                                                                                              |
| SPEED2    | This appears to be the posted speed of the link.                                                                                                                                                                                                                                                                                                                                                                                                          | 
| LENGTH    | A SEG_LENGTH field was included in the HASTUS export. This appears to be based on the Street Pro Nav length and does not appear to have been updated for new or split links. A new LENGTH field was calculated based on the spatial object length.                                                                                                                                                                                                        |  
                                                                                                                                                                                                                                                                                                                                                                                                         | 

2. Itinerary Data Preparation:
    Itineraries provide the ordered sequence of links that a bus must travel between stops for every route variant (shape_id in GTFS). The itineraries' segments corresponds with the relevant stop sequence as follows:
    - the first link in the fist segment corresponds to the first stop.
      - the last link in the first segment corresponds to the second stop.
      - the last link of the segment number (n-1) corresponds with the nth stop. 
      With the above description, the data preparation corrects segment orders so that they correspond to the stop sequence. This is done by the `PrepareItinerary` class within the data preparation module. The processed itinerary data is then saved in a standardised format for further use in the ETL process. Table below shows the required attributes in the raw itinerary:
    
    | Attribute | Description                                                                                                                                                   |
    |-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
    | shape_id  | This uniquely identifies a route variant by direction and matches shape_id in GTFS                                                                            |
    | segment   | This identifies stop to stop movements. The number of segments is one less than the number of stops                                                           |
    | order     | This identifies the order of links in a Segment                                                                                                               |
    | seg_id    | This identifies the link. This is the same as the ID field in the network.                                                                                    |
    | direction | This identifies the direction of a link and is the equivalent of SEG_FLOW in the HASTUS Links table. Which correspond with the direction field in the network |
    

3. Spatial Itinerary Data Preparation
Once the itinerary data has been processed, the spatial itinerary data is prepared by joining the spatial objects (from the full network) with the corresponding itinerary data. This ensures that the geospatial details are integrated into the itineraries, making the process of reading the spatial network much easier, as only the part of the network relevant to the region of interest is loaded into memory, rather than the full network.
4. Ticketing Data Preparation
The ticketing data, which includes transaction and trip stop timing data, is processed in a structured way. The preparation stage reads the raw ticketing data and organises it by partitioning the data based on the date. This enables the ETL process to efficiently handle ticketing data on a day-by-day basis, optimising performance and storage.
