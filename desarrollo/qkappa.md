-- Create source table for raw inputs
CREATE TABLE raw_inputs (
  patient_id STRING,
  measurement_type STRING,
  source_type STRING,
  source_id STRING,
  timestamp TIMESTAMP(3),
  value ROW<...>,  -- Type-specific fields
  quality ROW
    device_quality DOUBLE,
    measurement_conditions DOUBLE,
    signal_quality DOUBLE
  >,
  PRIMARY KEY (patient_id) NOT ENFORCED
) WITH (
  'connector' = 'kafka',
  'topic' = 'raw.inputs',
  'properties.bootstrap.servers' = '...',
  'format' = 'json'
);

-- Create target tables for each measurement type
CREATE TABLE measurements_respiratory_rate (
  patient_id STRING,
  source_type STRING,
  source_id STRING,
  timestamp TIMESTAMP(3),
  value DOUBLE,  -- breaths per minute
  quality ROW<...>,
  PRIMARY KEY (patient_id) NOT ENFORCED
) WITH (
  'connector' = 'kafka',
  'topic' = 'measurements.respiratory_rate',
  'format' = 'json'
);

-- Similar CREATE TABLE statements for other vitals...

-- Distribution queries
INSERT INTO measurements_respiratory_rate
SELECT 
  patient_id,
  source_type,
  source_id,
  timestamp,
  CAST(value.raw_value AS DOUBLE) as value,
  quality
FROM raw_inputs 
WHERE measurement_type = 'RESPIRATORY_RATE';

-- Similar INSERT statements for other vitals...

-- For respiratory rate as example
CREATE TABLE parameters_respiratory_rate (
  patient_id STRING,
  timestamp TIMESTAMP(3),
  value DOUBLE,
  raw_news2_score INT,
  freshness ROW
    weight DOUBLE,
    age_minutes INT,
    max_valid_hours INT
  >,
  quality ROW
    weight DOUBLE,
    components ROW
      device_quality DOUBLE,
      measurement_conditions DOUBLE,
      signal_quality DOUBLE
    >
  >,
  adjusted_score DOUBLE,
  status STRING,
  PRIMARY KEY (patient_id) NOT ENFORCED
) WITH (
  'connector' = 'kafka',
  'topic' = 'parameters.respiratory_rate',
  'format' = 'json'
);

-- Scoring logic for respiratory rate
INSERT INTO parameters_respiratory_rate
SELECT
  patient_id,
  timestamp,
  value,
  CASE  -- NEWS2 scoring logic
    WHEN value <= 8 THEN 3
    WHEN value <= 11 THEN 1
    WHEN value <= 20 THEN 0
    WHEN value <= 24 THEN 2
    ELSE 3
  END as raw_news2_score,
  ROW(
    -- Freshness calculation (8 hour window for respiratory rate)
    CAST(GREATEST(0, 1 - TIMESTAMPDIFF(HOUR, timestamp, CURRENT_TIMESTAMP) / 8.0) AS DOUBLE),
    TIMESTAMPDIFF(MINUTE, timestamp, CURRENT_TIMESTAMP),
    8
  ) as freshness,
  ROW(
    (quality.device_quality * 0.4 + 
     quality.measurement_conditions * 0.3 + 
     quality.signal_quality * 0.3),
    ROW(
      quality.device_quality,
      quality.measurement_conditions,
      quality.signal_quality
    )
  ) as quality,
  -- Adjusted score calculation
  CAST(raw_news2_score * 
       GREATEST(0, 1 - TIMESTAMPDIFF(HOUR, timestamp, CURRENT_TIMESTAMP) / 8.0) *
       (quality.device_quality * 0.4 + 
        quality.measurement_conditions * 0.3 + 
        quality.signal_quality * 0.3) AS DOUBLE) as adjusted_score,
  CASE
    WHEN quality.device_quality * 0.4 + 
         quality.measurement_conditions * 0.3 + 
         quality.signal_quality * 0.3 >= 0.8 THEN 'VALID'
    WHEN quality.device_quality * 0.4 + 
         quality.measurement_conditions * 0.3 + 
         quality.signal_quality * 0.3 >= 0.5 THEN 'DEGRADED'
    ELSE 'INVALID'
  END as status
FROM measurements_respiratory_rate;

CREATE TABLE scores_computed (
  patient_id STRING,
  timestamp TIMESTAMP(3),
  score DOUBLE,
  confidence DOUBLE,
  summary_stats ROW
    valid_parameters INT,
    degraded_parameters INT,
    invalid_parameters INT,
    total_adjusted_score DOUBLE,
    average_quality DOUBLE,
    average_freshness DOUBLE,
    oldest_measurement_age INT
  >,
  PRIMARY KEY (patient_id) NOT ENFORCED
) WITH (
  'connector' = 'kafka',
  'topic' = 'scores.computed',
  'format' = 'json'
);

-- We'll need to join all parameter topics within their validity windows
INSERT INTO scores_computed
SELECT
  r.patient_id,
  CURRENT_TIMESTAMP as timestamp,
  -- Sum of all adjusted scores
  (COALESCE(r.adjusted_score, 0) + 
   COALESCE(o.adjusted_score, 0) + 
   COALESCE(b.adjusted_score, 0) +
   COALESCE(h.adjusted_score, 0) +
   COALESCE(t.adjusted_score, 0) +
   COALESCE(c.adjusted_score, 0)) as score,
  -- Confidence calculation based on quality and freshness
  (
    SELECT AVG(quality.weight) 
    FROM (VALUES 
      (r.quality.weight),
      (o.quality.weight),
      (b.quality.weight),
      (h.quality.weight),
      (t.quality.weight),
      (c.quality.weight)
    ) as q(quality)
  ) * 100 as confidence,
  ROW(
    -- Count of valid parameters
    (CASE WHEN r.status = 'VALID' THEN 1 ELSE 0 END +
     CASE WHEN o.status = 'VALID' THEN 1 ELSE 0 END +
     CASE WHEN b.status = 'VALID' THEN 1 ELSE 0 END +
     CASE WHEN h.status = 'VALID' THEN 1 ELSE 0 END +
     CASE WHEN t.status = 'VALID' THEN 1 ELSE 0 END +
     CASE WHEN c.status = 'VALID' THEN 1 ELSE 0 END),
    -- Similar counts for degraded and invalid
    -- other summary stats...
  ) as summary_stats
FROM parameters_respiratory_rate r
FULL OUTER JOIN parameters_oxygen_saturation o ON r.patient_id = o.patient_id
FULL OUTER JOIN parameters_blood_pressure b ON r.patient_id = b.patient_id
FULL OUTER JOIN parameters_heart_rate h ON r.patient_id = h.patient_id
FULL OUTER JOIN parameters_temperature t ON r.patient_id = t.patient_id
FULL OUTER JOIN parameters_consciousness c ON r.patient_id = c.patient_id
-- Window specification for temporal correlation
WHERE -- Add time window conditions;

CREATE TABLE alerts_triggered (
  alert_id STRING,
  patient_id STRING,
  type STRING,
  severity INT,
  timestamp TIMESTAMP(3),
  details ROW<...>,  -- Alert-specific details
  confidence DOUBLE,
  PRIMARY KEY (alert_id) NOT ENFORCED
) WITH (
  'connector' = 'kafka',
  'topic' = 'alerts.triggered',
  'format' = 'json'
);

-- Alert generation logic
INSERT INTO alerts_triggered
SELECT
  -- Alert generation based on scores and thresholds
  MD5(CONCAT(patient_id, CAST(CURRENT_TIMESTAMP AS STRING))) as alert_id,
  patient_id,
  CASE
    WHEN score >= 7 THEN 'HIGH_SCORE'
    WHEN confidence < 60 THEN 'LOW_CONFIDENCE'
    -- other conditions...
  END as type,
  -- severity calculation...
FROM scores_computed
WHERE score >= 7 OR confidence < 60  -- Alert conditions;