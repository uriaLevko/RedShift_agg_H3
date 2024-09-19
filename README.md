
 <h1>Project: Aggregating Satellite Data with H3 Hexagons in Redshift</h1>
    
<h2>Overview</h2>
    <p>
        This project involved working with satellite imagery data stored in a Redshift table. 
        The goal was to aggregate pixel-level data into H3 hexagons at a specific resolution (13), 
        calculate various metrics (such as bloom index and carbon levels), and identify the majority values 
        for certain fields (such as classification and bgi). The results were stored in an output table for future analysis.
    </p>

<h2>Steps in the Process</h2>

  <h3>1. Creating the Input Table</h3>
  <p>
      We first created an empty table named <strong>demo_table</strong>, which stores the pixel-level data. 
      This table contains latitude/longitude information, various satellite metrics (bloom index, carbon, classification, etc.), 
      and related metadata.
  </p>

    <pre><code>CREATE TABLE demo_table (
    _id VARCHAR(255),
    date TIMESTAMP,
    geometry_type VARCHAR(50),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    planet_h312 VARCHAR(255),
    planet_h313 VARCHAR(255),
    planet_h314 VARCHAR(255),
    planet_h315 VARCHAR(255),
    planet_classification INT,
    planet_bgi INT,
    planet_bloom DOUBLE PRECISION,
    planet_carbon DOUBLE PRECISION,
    relatedAOIs VARCHAR(255),
    lastUpdatedAt TIMESTAMP);


  <h3>2. Inserting Dummy Data for Testing</h3>
  <p>We inserted a few rows of <strong>dummy data</strong> into the table to simulate actual satellite data.</p>

    <pre><code>INSERT INTO demo_table (
    _id, date, geometry_type, latitude, longitude, planet_h312, planet_h313, planet_h314, planet_h315,
    planet_classification, planet_bgi, planet_bloom, planet_carbon, relatedAOIs, lastUpdatedAt
    ) VALUES
    ('6662b1e9259982ea9578af27', '2024-06-06T00:00:00', 'Point', -81.71524, 30.15538, '8c44f01862a85ff', '8d44f01862a84ff', '8e44f01862a84ef', '8f44f01862a84ee', 11, 5, 53.3, 609.2, '050de9a77469b480f89ba57e6787b1d9', '2024-06-07T07:08:25.834Z'),
    ('6662b1e9259982ea9578af28', '2024-06-06T00:00:00', 'Point', -81.71521, 30.15538, '8c44f01862a85ff', '8d44f01862a84ff', '8e44f01862a84c7', '8f44f01862a84c6', 11, 5, 52.5, 600.1, '050de9a77469b480f89ba57e6787b1d9', '2024-06-07T07:08:25.835Z'),
    ('6662b1e9259982ea9578af29', '2024-06-06T00:00:00', 'Point', -81.71518, 30.15538, '8c44f01862a85ff', '8d44f01862aab3f', '8e44f01862aab27', '8f44f01862aab25', 11, 5, 55.9, 638.9, '050de9a77469b480f89ba57e6787b1d9', '2024-06-07T07:08:25.835Z');

  <h3>3. Aggregating Data by H3 Hexagons</h3>
  <p>Next, we performed two versions of the <strong>aggregation</strong>:</p>
  <ul>
      <li>One based on pre-computed H3 hexagons (<code>planet_h313</code>).</li>
      <li>One based on dynamically calculating the H3 hexagon ID using <code>H3_FromLongLat</code>.</li>
  </ul>

  <h4>Query 1: Aggregation by Pre-Computed H3 Hexagons</h4>
    <pre><code>SELECT
    planet_h313 AS h3_id,
    MIN(lastUpdatedAt) AS first_lastUpdatedAt,
    MIN(date) AS first_date,
    AVG(latitude) AS center_latitude,
    ANY_VALUE(relatedAOIs) as relatedAOIs,
    AVG(planet_bloom) AS avg_bloomIndex,
    AVG(planet_carbon) AS avg_carbon,
    FIRST_VALUE(relatedAOIs) OVER (PARTITION BY planet_h313 ORDER BY lastUpdatedAt ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS first_relatedAOIs,
    COUNT(*) AS pixel_count
    FROM demo_table
    GROUP BY planet_h313, relatedAOIs, lastUpdatedAt;

    <h4>Query 2: Aggregation by Calculated H3 Hexagons</h4>
    <pre><code>SELECT
    H3_FromLongLat(longitude, latitude, 13) AS h3_id,
    MIN(lastUpdatedAt) AS first_lastUpdatedAt,
    MIN(date) AS first_date,
    AVG(planet_bloom) AS avg_bloomIndex,
    AVG(planet_bgi) AS avg_bgi,
    AVG(planet_carbon) AS avg_carbon,
    COUNT(*) AS pixel_count
FROM demo_table
GROUP BY h3_id;</code></pre>

<h3>4. Calculating the Most Common Values</h3>
<p>We used subqueries to calculate the <strong>most common values</strong> for <code>planet_classification</code> and <code>planet_bgi</code>.</p>

    <pre><code>WITH classification_counts AS (
    SELECT
        H3_FromLongLat(longitude, latitude, 13) AS h3_id,
        planet_classification,
        COUNT(*) AS class_count
    FROM demo_table
    GROUP BY h3_id, planet_classification
    ),
    bgi_counts AS (
    SELECT
        H3_FromLongLat(longitude, latitude, 13) AS h3_id,
        planet_bgi,
        COUNT(*) AS bgi_count
    FROM demo_table
    GROUP BY h3_id, planet_bgi
    ),
    most_common_classification AS (
    SELECT
        h3_id AS classification_h3_id,
        planet_classification AS most_common_classification,
        ROW_NUMBER() OVER (PARTITION BY h3_id ORDER BY class_count DESC) AS rank_classification
    FROM classification_counts
    ),
    most_common_bgi AS (
    SELECT
        h3_id AS bgi_h3_id,
        planet_bgi AS most_common_bgi,
        ROW_NUMBER() OVER (PARTITION BY h3_id ORDER BY bgi_count DESC) AS rank_bgi
    FROM bgi_counts
    )
    SELECT
    H3_FromLongLat(longitude, latitude, 13) AS h3_id,
    MIN(lastUpdatedAt) AS first_lastUpdatedAt,
    MIN(date) AS first_date,
    ANY_VALUE(relatedAOIs) as relatedAOIs,
    AVG(planet_bloom) AS avg_bloomIndex,
    AVG(planet_carbon) AS avg_carbon,
    mcc.most_common_classification,
    mcb.most_common_bgi,
    COUNT(*) AS pixel_count
    FROM demo_table base
    LEFT JOIN most_common_classification mcc 
    ON H3_FromLongLat(base.longitude, base.latitude, 13) = mcc.classification_h3_id AND mcc.rank_classification = 1
    LEFT JOIN most_common_bgi mcb 
    ON H3_FromLongLat(base.longitude, base.latitude, 13) = mcb.bgi_h3_id AND mcb.rank_bgi = 1
    GROUP BY h3_id, mcc.most_common_classification, mcb.most_common_bgi;

<h3>5. Creating an Output Table to Store Results</h3>
<p>We created an empty output table named <strong>h3_results_table</strong> to store the aggregated results. The structure matches the output of the aggregation queries.</p>

    <pre><code>CREATE TABLE h3_results_table (
    h3_id BIGINT,
    first_lastUpdatedAt TIMESTAMP,
    first_date TIMESTAMP,
    relatedAOIs VARCHAR(255),
    avg_bloomIndex DOUBLE PRECISION,
    avg_carbon DOUBLE PRECISION,
    most_common_classification INT,
    most_common_bgi INT,
    pixel_count INT
    );


<h3>6. Inserting Final Results into the Output Table</h3>
<p>
    Once the output table <strong>h3_results_table</strong> was created, we inserted the results from the previous queries. This step ensured that the aggregated results, including the most common values for classification and bgi, were stored for further analysis.
</p>

    <pre><code>INSERT INTO h3_results_table
    WITH classification_counts AS (
    SELECT
        H3_FromLongLat(longitude, latitude, 13) AS h3_id,
        planet_classification,
        COUNT(*) AS class_count
    FROM demo_table
    GROUP BY h3_id, planet_classification
    ),
    bgi_counts AS (
    SELECT
        H3_FromLongLat(longitude, latitude, 13) AS h3_id,
        planet_bgi,
        COUNT(*) AS bgi_count
    FROM demo_table
    GROUP BY h3_id, planet_bgi
    ),
    most_common_classification AS (
    SELECT
        h3_id AS classification_h3_id,
        planet_classification AS most_common_classification,
        ROW_NUMBER() OVER (PARTITION BY h3_id ORDER BY class_count DESC) AS rank_classification
    FROM classification_counts
    ),
    most_common_bgi AS (
        SELECT
        h3_id AS bgi_h3_id,
        planet_bgi AS most_common_bgi,
        ROW_NUMBER() OVER (PARTITION BY h3_id ORDER BY bgi_count DESC) AS rank_bgi
    FROM bgi_counts
    )
    SELECT
    H3_FromLongLat(longitude, latitude, 13) AS h3_id,  -- Calculate H3 ID
    MIN(lastUpdatedAt) AS first_lastUpdatedAt,  -- Earliest lastUpdatedAt timestamp
    MIN(date) AS first_date,  -- Earliest event date
    ANY_VALUE(relatedAOIs) as relatedAOIs,  -- Related AOI
    AVG(planet_bloom) AS avg_bloomIndex,  -- Average bloomIndex
    AVG(planet_carbon) AS avg_carbon,  -- Average carbon
    mcc.most_common_classification,  -- Most common classification
    mcb.most_common_bgi,  -- Most common bgi
    COUNT(*) AS pixel_count  -- Number of pixels aggregated into this H3 cell
    FROM
    demo_table base
    LEFT JOIN most_common_classification mcc 
    ON H3_FromLongLat(base.longitude, base.latitude, 13) = mcc.classification_h3_id AND mcc.rank_classification = 1
    LEFT JOIN most_common_bgi mcb 
    ON H3_FromLongLat(base.longitude, base.latitude, 13) = mcb.bgi_h3_id AND mcb.rank_bgi = 1
    GROUP BY
    h3_id, mcc.most_common_classification, mcb.most_common_bgi;</code></pre>

<h2>Conclusion</h2>
<p>
In this project, we successfully created an efficient pipeline to aggregate pixel-level satellite data into H3 hexagons, calculate the most common values for key metrics, and store the results in an output table. This method can now be used to efficiently analyze satellite imagery data at various H3 resolutions and provide insights into different environmental factors such as bloom index, carbon content, and classifications across different locations.
</p>


