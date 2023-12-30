---
rootPaths:
  - support/assurance/synthetic-content
icDb: support/assurance/results-test-e2e/ingestion-center.duckdb
diagsMd: support/assurance/results-test-e2e/diagnostics.md
diagsXlsx: support/assurance/results-test-e2e/diagnostics.xlsx
resourceDb: support/assurance/results-test-e2e/resource.sqlite.db
sources:
  - uri: support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx
    nature: Excel Workbook Sheet
    tableName: ahc_hrsn_valid_01_admin_demographic
    assurable: false
  - uri: support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx
    nature: Excel Workbook Sheet
    tableName: ahc_hrsn_valid_01_screening
    assurable: false
  - uri: support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx
    nature: Excel Workbook Sheet
    tableName: ahc_hrsn_valid_01_q_e_admin_data
    assurable: false
  - uri: support/assurance/synthetic-content/synthetic-fail.csv
    nature: CSV
    tableName: synthetic_fail
    assurable: false
  - uri: support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx
    nature: ERROR
    tableName: ERROR
    assurable: false
  - uri: support/assurance/synthetic-content/ahc-hrsn-12-12-2023-valid.csv
    nature: CSV
    tableName: ahc_hrsn_12_12_2023_valid
    assurable: true
---
# Ingest Diagnostics

## init
```sql

-- no before-init SQL found
CREATE TABLE IF NOT EXISTS "ingest_session" (
    "ingest_session_id" TEXT PRIMARY KEY NOT NULL,
    "ingest_started_at" TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    "ingest_finished_at" TIMESTAMPTZ,
    "elaboration" TEXT
);
CREATE TABLE IF NOT EXISTS "ingest_session_entry" (
    "ingest_session_entry_id" TEXT PRIMARY KEY NOT NULL,
    "session_id" TEXT NOT NULL,
    "ingest_src" TEXT NOT NULL,
    "ingest_table_name" TEXT,
    "elaboration" TEXT,
    FOREIGN KEY("session_id") REFERENCES "ingest_session"("ingest_session_id")
);
CREATE TABLE IF NOT EXISTS "ingest_session_state" (
    "ingest_session_state_id" TEXT PRIMARY KEY NOT NULL,
    "session_id" TEXT NOT NULL,
    "session_entry_id" TEXT,
    "from_state" TEXT NOT NULL,
    "to_state" TEXT NOT NULL,
    "transition_result" TEXT,
    "transition_reason" TEXT,
    "transitioned_at" TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    "elaboration" TEXT,
    FOREIGN KEY("session_id") REFERENCES "ingest_session"("ingest_session_id"),
    FOREIGN KEY("session_entry_id") REFERENCES "ingest_session_entry"("ingest_session_entry_id"),
    UNIQUE("ingest_session_state_id", "from_state", "to_state")
);
CREATE TABLE IF NOT EXISTS "ingest_session_issue" (
    "ingest_session_issue_id" TEXT PRIMARY KEY NOT NULL,
    "session_id" TEXT NOT NULL,
    "session_entry_id" TEXT,
    "issue_type" TEXT NOT NULL,
    "issue_message" TEXT NOT NULL,
    "issue_row" INTEGER,
    "issue_column" TEXT,
    "invalid_value" TEXT,
    "remediation" TEXT,
    "elaboration" TEXT,
    FOREIGN KEY("session_id") REFERENCES "ingest_session"("ingest_session_id"),
    FOREIGN KEY("session_entry_id") REFERENCES "ingest_session_entry"("ingest_session_entry_id")
);


DROP VIEW IF EXISTS "ingest_session_diagnostic_text";
CREATE VIEW IF NOT EXISTS "ingest_session_diagnostic_text" AS
    SELECT
        -- Including all other columns from 'ingest_session'
        ises.* EXCLUDE (ingest_started_at, ingest_finished_at),
        -- TODO: Casting known timestamp columns to text so emit to Excel works with GDAL (spatial)
           -- strftime(timestamptz ingest_started_at, '%Y-%m-%d %H:%M:%S') AS ingest_started_at,
           -- strftime(timestamptz ingest_finished_at, '%Y-%m-%d %H:%M:%S') AS ingest_finished_at,
    
        -- Including all columns from 'ingest_session_entry'
        isee.* EXCLUDE (session_id),
    
        -- Including all other columns from 'ingest_session_issue'
        isi.* EXCLUDE (session_id, session_entry_id)
    FROM ingest_session AS ises
    JOIN ingest_session_entry AS isee ON ises.ingest_session_id = isee.session_id
    LEFT JOIN ingest_session_issue AS isi ON isee.ingest_session_entry_id = isi.session_entry_id;

-- register the current session and use the identifier for all logging
INSERT INTO "ingest_session" ("ingest_session_id", "ingest_started_at", "ingest_finished_at", "elaboration") VALUES ('05269d28-15ae-5bd6-bd88-f949ccfa52d7', NULL, NULL, NULL);

-- no after-init SQL found
```


## ingest-0
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('7bab389e-54af-5a13-a39f-079abdc73a48', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx', 'ahc_hrsn_valid_01_admin_demographic', NULL);
        
INSERT INTO "ingest_session_issue" ("ingest_session_issue_id", "session_id", "session_entry_id", "issue_type", "issue_message", "issue_row", "issue_column", "invalid_value", "remediation", "elaboration") VALUES ('168a34c7-d043-5ec4-a84a-c961f1a301ef', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', '7bab389e-54af-5a13-a39f-079abdc73a48', 'TODO', 'Excel workbook ''ahc-hrsn-valid-01.xlsx'' sheet ''Admin_Demographic'' has not been implemented yet.', NULL, NULL, 'support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx', NULL, NULL);;

-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = '7bab389e-54af-5a13-a39f-079abdc73a48';
```

### stdout
```sh
[{"ingest_session_issue_id":"168a34c7-d043-5ec4-a84a-c961f1a301ef","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"7bab389e-54af-5a13-a39f-079abdc73a48","issue_type":"TODO","issue_message":"Excel workbook 'ahc-hrsn-valid-01.xlsx' sheet 'Admin_Demographic' has not been implemented yet.","issue_row":null,"issue_column":null,"invalid_value":"support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx","remediation":null,"elaboration":null}]

```

## ingest-1
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('1931dfcc-e8fc-597d-b1bc-65b4287e6fdf', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx', 'ahc_hrsn_valid_01_screening', NULL);
     
-- ingest Excel workbook sheet 'Screening' into ahc_hrsn_valid_01_screening using spatial plugin
INSTALL spatial; LOAD spatial;

-- be sure to add src_file_row_number and session_id columns to each row
-- because assurance CTEs require them
CREATE TABLE ahc_hrsn_valid_01_screening AS
  SELECT *, row_number() OVER () as src_file_row_number, '05269d28-15ae-5bd6-bd88-f949ccfa52d7' as session_id, '1931dfcc-e8fc-597d-b1bc-65b4287e6fdf' as session_entry_id
    FROM st_read('support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx', layer='Screening', open_options=['HEADERS=FORCE', 'FIELD_TYPES=AUTO']);          

WITH required_column_names_in_src AS (
    SELECT column_name
      FROM (VALUES ('PAT_MRN_ID'), ('FACILITY'), ('FIRST_NAME'), ('LAST_NAME'), ('PAT_BIRTH_DATE'), ('MEDICAID_CIN'), ('ENCOUNTER_ID'), ('SURVEY'), ('SURVEY_ID'), ('RECORDED_TIME'), ('QUESTION'), ('MEAS_VALUE'), ('QUESTION_CODE'), ('QUESTION_CODE_SYSTEM_NAME'), ('ANSWER_CODE'), ('ANSWER_CODE_SYSTEM_NAME'), ('SDOH_DOMAIN'), ('NEED_INDICATED'), ('VISIT_PART_2_FLAG'), ('VISIT_OMH_FLAG'), ('VISIT_OPWDD_FLAG')) AS required(column_name)
     WHERE required.column_name NOT IN (
         SELECT upper(trim(column_name))
           FROM information_schema.columns
          WHERE table_name = 'ahc_hrsn_valid_01_screening')
)
INSERT INTO ingest_session_issue (ingest_session_issue_id, session_id, session_entry_id, issue_type, issue_message, remediation)
    SELECT uuid(),
           '05269d28-15ae-5bd6-bd88-f949ccfa52d7',
           '1931dfcc-e8fc-597d-b1bc-65b4287e6fdf',
           'Missing Column',
           'Required column ' || column_name || ' is missing in ahc_hrsn_valid_01_screening.',
           'Ensure ahc_hrsn_valid_01_screening contains the column "' || column_name || '"'
      FROM required_column_names_in_src;

-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = '1931dfcc-e8fc-597d-b1bc-65b4287e6fdf';
```

### stdout
```sh
[{"ingest_session_issue_id":"efc1e820-da65-4e18-8468-0e81abb125b2","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column PAT_MRN_ID is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"PAT_MRN_ID\"","elaboration":null},
{"ingest_session_issue_id":"9414797b-9423-4ca8-872f-7eb109c0aa76","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column FACILITY is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"FACILITY\"","elaboration":null},
{"ingest_session_issue_id":"5b1a083f-ac68-4121-aa42-390de29477f8","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column FIRST_NAME is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"FIRST_NAME\"","elaboration":null},
{"ingest_session_issue_id":"033851ed-1ab3-432f-b359-37af598fc4ca","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column LAST_NAME is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"LAST_NAME\"","elaboration":null},
{"ingest_session_issue_id":"2d549b3a-c0dc-418c-9dde-1d9ec283585d","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column PAT_BIRTH_DATE is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"PAT_BIRTH_DATE\"","elaboration":null},
{"ingest_session_issue_id":"3892eea6-0ccb-46c7-b6eb-b1104a764157","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column MEDICAID_CIN is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"MEDICAID_CIN\"","elaboration":null},
{"ingest_session_issue_id":"0928c6d3-0e58-432c-bc43-3f3df029de27","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column ENCOUNTER_ID is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"ENCOUNTER_ID\"","elaboration":null},
{"ingest_session_issue_id":"b0df8ac0-55cc-43b0-88ea-8dd6cab6608f","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column SURVEY is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"SURVEY\"","elaboration":null},
{"ingest_session_issue_id":"f0f6d0aa-5c74-47e3-9d38-9588a709666e","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column SURVEY_ID is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"SURVEY_ID\"","elaboration":null},
{"ingest_session_issue_id":"15f29e60-b2c6-426d-8c54-1ca2391f48c2","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column RECORDED_TIME is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"RECORDED_TIME\"","elaboration":null},
{"ingest_session_issue_id":"d2431531-532e-4e66-b67e-31a53187f673","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column QUESTION is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"QUESTION\"","elaboration":null},
{"ingest_session_issue_id":"9f0035a6-35f9-4807-b54e-dd2426a8f402","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column MEAS_VALUE is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"MEAS_VALUE\"","elaboration":null},
{"ingest_session_issue_id":"13671684-1927-449f-ac60-d8450c236269","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column QUESTION_CODE is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"QUESTION_CODE\"","elaboration":null},
{"ingest_session_issue_id":"81ac4542-5cc3-482f-b50d-234718148d00","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column QUESTION_CODE_SYSTEM_NAME is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"QUESTION_CODE_SYSTEM_NAME\"","elaboration":null},
{"ingest_session_issue_id":"b525ecf0-c916-4233-8ce5-3f7b9096eb98","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column ANSWER_CODE is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"ANSWER_CODE\"","elaboration":null},
{"ingest_session_issue_id":"32f88e3e-9280-4e11-bc2e-dc7687e04f9f","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column ANSWER_CODE_SYSTEM_NAME is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"ANSWER_CODE_SYSTEM_NAME\"","elaboration":null},
{"ingest_session_issue_id":"c3680798-6fe4-4e01-909f-f6369df4d89e","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column SDOH_DOMAIN is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"SDOH_DOMAIN\"","elaboration":null},
{"ingest_session_issue_id":"3775d60c-6e3f-4416-8ac3-950b486f31f3","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column NEED_INDICATED is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"NEED_INDICATED\"","elaboration":null},
{"ingest_session_issue_id":"aed7d34d-5f41-475b-9f81-f6ae25d21d57","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column VISIT_PART_2_FLAG is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"VISIT_PART_2_FLAG\"","elaboration":null},
{"ingest_session_issue_id":"9ca0bfbe-c6b8-4808-a60c-32bc89698fbd","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column VISIT_OMH_FLAG is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"VISIT_OMH_FLAG\"","elaboration":null},
{"ingest_session_issue_id":"0c1690e6-d1ba-48ec-8517-0e15ea9a1ba2","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"1931dfcc-e8fc-597d-b1bc-65b4287e6fdf","issue_type":"Missing Column","issue_message":"Required column VISIT_OPWDD_FLAG is missing in ahc_hrsn_valid_01_screening.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure ahc_hrsn_valid_01_screening contains the column \"VISIT_OPWDD_FLAG\"","elaboration":null}]

```

## ingest-2
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('8b7c669c-1795-5f6b-8f3a-3e502b74c628', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx', 'ahc_hrsn_valid_01_q_e_admin_data', NULL);
        
INSERT INTO "ingest_session_issue" ("ingest_session_issue_id", "session_id", "session_entry_id", "issue_type", "issue_message", "issue_row", "issue_column", "invalid_value", "remediation", "elaboration") VALUES ('7b979b68-7227-53fd-b689-e4fe153afb76', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', '8b7c669c-1795-5f6b-8f3a-3e502b74c628', 'TODO', 'Excel workbook ''ahc-hrsn-valid-01.xlsx'' sheet ''QE_Admin_Data'' has not been implemented yet.', NULL, NULL, 'support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx', NULL, NULL);;

-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = '8b7c669c-1795-5f6b-8f3a-3e502b74c628';
```

### stdout
```sh
[{"ingest_session_issue_id":"7b979b68-7227-53fd-b689-e4fe153afb76","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"8b7c669c-1795-5f6b-8f3a-3e502b74c628","issue_type":"TODO","issue_message":"Excel workbook 'ahc-hrsn-valid-01.xlsx' sheet 'QE_Admin_Data' has not been implemented yet.","issue_row":null,"issue_column":null,"invalid_value":"support/assurance/synthetic-content/ahc-hrsn-valid-01.xlsx","remediation":null,"elaboration":null}]

```

## ingest-3
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('abf5c680-a135-5d89-b871-fa5b9b99aed6', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/synthetic-fail.csv', 'synthetic_fail', NULL);

-- be sure to add src_file_row_number and session_id columns to each row
-- because assurance CTEs require them
CREATE TABLE synthetic_fail AS
  SELECT *, row_number() OVER () as src_file_row_number, '05269d28-15ae-5bd6-bd88-f949ccfa52d7' as session_id, 'abf5c680-a135-5d89-b871-fa5b9b99aed6' as session_entry_id
    FROM read_csv_auto('support/assurance/synthetic-content/synthetic-fail.csv');

WITH required_column_names_in_src AS (
    SELECT column_name
      FROM (VALUES ('PAT_MRN_ID'), ('FACILITY'), ('FIRST_NAME'), ('LAST_NAME'), ('PAT_BIRTH_DATE'), ('MEDICAID_CIN'), ('ENCOUNTER_ID'), ('SURVEY'), ('SURVEY_ID'), ('RECORDED_TIME'), ('QUESTION'), ('MEAS_VALUE'), ('QUESTION_CODE'), ('QUESTION_CODE_SYSTEM_NAME'), ('ANSWER_CODE'), ('ANSWER_CODE_SYSTEM_NAME'), ('SDOH_DOMAIN'), ('NEED_INDICATED'), ('VISIT_PART_2_FLAG'), ('VISIT_OMH_FLAG'), ('VISIT_OPWDD_FLAG')) AS required(column_name)
     WHERE required.column_name NOT IN (
         SELECT upper(trim(column_name))
           FROM information_schema.columns
          WHERE table_name = 'synthetic_fail')
)
INSERT INTO ingest_session_issue (ingest_session_issue_id, session_id, session_entry_id, issue_type, issue_message, remediation)
    SELECT uuid(),
           '05269d28-15ae-5bd6-bd88-f949ccfa52d7',
           'abf5c680-a135-5d89-b871-fa5b9b99aed6',
           'Missing Column',
           'Required column ' || column_name || ' is missing in synthetic_fail.',
           'Ensure synthetic_fail contains the column "' || column_name || '"'
      FROM required_column_names_in_src;

-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = 'abf5c680-a135-5d89-b871-fa5b9b99aed6';
```

### stdout
```sh
[{"ingest_session_issue_id":"e0ef5d95-0d2b-4066-8990-815529b835a3","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column PAT_MRN_ID is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"PAT_MRN_ID\"","elaboration":null},
{"ingest_session_issue_id":"7dabc078-629c-4621-b16a-5a639ccfe754","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column FACILITY is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"FACILITY\"","elaboration":null},
{"ingest_session_issue_id":"19f30cc8-0a8d-494d-bf41-247fbc0c1738","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column FIRST_NAME is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"FIRST_NAME\"","elaboration":null},
{"ingest_session_issue_id":"3da9a0fc-aa88-4dfc-b1a6-62cd053f70eb","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column LAST_NAME is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"LAST_NAME\"","elaboration":null},
{"ingest_session_issue_id":"1efeeef1-1f7e-4713-b2a1-9186b83213fe","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column PAT_BIRTH_DATE is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"PAT_BIRTH_DATE\"","elaboration":null},
{"ingest_session_issue_id":"0bc52139-5699-48a3-b5c8-97785c587fb6","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column MEDICAID_CIN is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"MEDICAID_CIN\"","elaboration":null},
{"ingest_session_issue_id":"da909980-d595-48ac-9c70-ac2c1d1369ad","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column ENCOUNTER_ID is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"ENCOUNTER_ID\"","elaboration":null},
{"ingest_session_issue_id":"1884edab-f544-4346-8b96-03c99dc0c256","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column SURVEY is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"SURVEY\"","elaboration":null},
{"ingest_session_issue_id":"18af58b9-98f6-4651-97a7-93061191fdeb","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column SURVEY_ID is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"SURVEY_ID\"","elaboration":null},
{"ingest_session_issue_id":"c9e4604b-2516-4094-a0b9-290626a02f1a","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column RECORDED_TIME is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"RECORDED_TIME\"","elaboration":null},
{"ingest_session_issue_id":"00c33614-4ed2-4258-acf5-725f7a586901","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column QUESTION is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"QUESTION\"","elaboration":null},
{"ingest_session_issue_id":"fc1975b7-80fc-4cc0-b82a-b5a2afc450e2","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column MEAS_VALUE is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"MEAS_VALUE\"","elaboration":null},
{"ingest_session_issue_id":"67600e7c-3091-4c25-b33d-2b7f96da6da9","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column QUESTION_CODE is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"QUESTION_CODE\"","elaboration":null},
{"ingest_session_issue_id":"3bf82264-b9a9-44bd-bf2d-8e9f022548d2","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column QUESTION_CODE_SYSTEM_NAME is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"QUESTION_CODE_SYSTEM_NAME\"","elaboration":null},
{"ingest_session_issue_id":"944d50c2-cbc1-45c5-a113-96553809d060","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column ANSWER_CODE is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"ANSWER_CODE\"","elaboration":null},
{"ingest_session_issue_id":"69ce2fee-0201-4480-998d-1713cb9ab3fe","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column ANSWER_CODE_SYSTEM_NAME is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"ANSWER_CODE_SYSTEM_NAME\"","elaboration":null},
{"ingest_session_issue_id":"a151c278-9a20-43be-9d63-72d7206b833f","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column SDOH_DOMAIN is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"SDOH_DOMAIN\"","elaboration":null},
{"ingest_session_issue_id":"aa1e709e-524e-4c8e-b4b1-c1910aba9478","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column NEED_INDICATED is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"NEED_INDICATED\"","elaboration":null},
{"ingest_session_issue_id":"a28b94f8-6e6e-40b2-b4fa-da926f929d9e","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column VISIT_PART_2_FLAG is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"VISIT_PART_2_FLAG\"","elaboration":null},
{"ingest_session_issue_id":"5bf45a6b-81de-4544-9dfa-9847dd150ab2","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column VISIT_OMH_FLAG is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"VISIT_OMH_FLAG\"","elaboration":null},
{"ingest_session_issue_id":"c4560caa-cb01-4228-b78c-e6b17c5ce141","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"abf5c680-a135-5d89-b871-fa5b9b99aed6","issue_type":"Missing Column","issue_message":"Required column VISIT_OPWDD_FLAG is missing in synthetic_fail.","issue_row":null,"issue_column":null,"invalid_value":null,"remediation":"Ensure synthetic_fail contains the column \"VISIT_OPWDD_FLAG\"","elaboration":null}]

```

## ingest-4
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('641dff51-97fd-56b3-8443-c1ed568a6d66', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx', 'ERROR', NULL);
INSERT INTO "ingest_session_issue" ("ingest_session_issue_id", "session_id", "session_entry_id", "issue_type", "issue_message", "issue_row", "issue_column", "invalid_value", "remediation", "elaboration") VALUES ('d70a4700-6b40-52fc-a7a2-69ef0d7f69ff', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', '641dff51-97fd-56b3-8443-c1ed568a6d66', 'Sheet Missing', 'Excel workbook sheet ''Admin_Demographic'' not found in ''synthetic-fail-excel-01.xlsx'' (available: Sheet1)', NULL, NULL, 'support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx', NULL, NULL);;
-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = '641dff51-97fd-56b3-8443-c1ed568a6d66';
```

### stdout
```sh
[{"ingest_session_issue_id":"d70a4700-6b40-52fc-a7a2-69ef0d7f69ff","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"641dff51-97fd-56b3-8443-c1ed568a6d66","issue_type":"Sheet Missing","issue_message":"Excel workbook sheet 'Admin_Demographic' not found in 'synthetic-fail-excel-01.xlsx' (available: Sheet1)","issue_row":null,"issue_column":null,"invalid_value":"support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx","remediation":null,"elaboration":null}]

```

## ingest-5
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('47277588-99e8-59f5-8384-b24344a86073', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx', 'ERROR', NULL);
INSERT INTO "ingest_session_issue" ("ingest_session_issue_id", "session_id", "session_entry_id", "issue_type", "issue_message", "issue_row", "issue_column", "invalid_value", "remediation", "elaboration") VALUES ('58b22e99-5854-53bf-adbe-08e67df99b85', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', '47277588-99e8-59f5-8384-b24344a86073', 'Sheet Missing', 'Excel workbook sheet ''Screening'' not found in ''synthetic-fail-excel-01.xlsx'' (available: Sheet1)', NULL, NULL, 'support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx', NULL, NULL);;
-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = '47277588-99e8-59f5-8384-b24344a86073';
```

### stdout
```sh
[{"ingest_session_issue_id":"58b22e99-5854-53bf-adbe-08e67df99b85","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"47277588-99e8-59f5-8384-b24344a86073","issue_type":"Sheet Missing","issue_message":"Excel workbook sheet 'Screening' not found in 'synthetic-fail-excel-01.xlsx' (available: Sheet1)","issue_row":null,"issue_column":null,"invalid_value":"support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx","remediation":null,"elaboration":null}]

```

## ingest-6
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('a26ce332-3ced-5623-861d-23a2ef78e4a9', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx', 'ERROR', NULL);
INSERT INTO "ingest_session_issue" ("ingest_session_issue_id", "session_id", "session_entry_id", "issue_type", "issue_message", "issue_row", "issue_column", "invalid_value", "remediation", "elaboration") VALUES ('bc0c03b5-d1ba-5301-850f-5e4c42c1bf09', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'a26ce332-3ced-5623-861d-23a2ef78e4a9', 'Sheet Missing', 'Excel workbook sheet ''QE_Admin_Data'' not found in ''synthetic-fail-excel-01.xlsx'' (available: Sheet1)', NULL, NULL, 'support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx', NULL, NULL);;
-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = 'a26ce332-3ced-5623-861d-23a2ef78e4a9';
```

### stdout
```sh
[{"ingest_session_issue_id":"bc0c03b5-d1ba-5301-850f-5e4c42c1bf09","session_id":"05269d28-15ae-5bd6-bd88-f949ccfa52d7","session_entry_id":"a26ce332-3ced-5623-861d-23a2ef78e4a9","issue_type":"Sheet Missing","issue_message":"Excel workbook sheet 'QE_Admin_Data' not found in 'synthetic-fail-excel-01.xlsx' (available: Sheet1)","issue_row":null,"issue_column":null,"invalid_value":"support/assurance/synthetic-content/synthetic-fail-excel-01.xlsx","remediation":null,"elaboration":null}]

```

## ingest-7
```sql
-- required by IngestEngine, setup the ingestion entry for logging
INSERT INTO "ingest_session_entry" ("ingest_session_entry_id", "session_id", "ingest_src", "ingest_table_name", "elaboration") VALUES ('ae477ba1-c7f1-5f34-847a-50bddb7130aa', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', 'support/assurance/synthetic-content/ahc-hrsn-12-12-2023-valid.csv', 'ahc_hrsn_12_12_2023_valid', NULL);

-- be sure to add src_file_row_number and session_id columns to each row
-- because assurance CTEs require them
CREATE TABLE ahc_hrsn_12_12_2023_valid AS
  SELECT *, row_number() OVER () as src_file_row_number, '05269d28-15ae-5bd6-bd88-f949ccfa52d7' as session_id, 'ae477ba1-c7f1-5f34-847a-50bddb7130aa' as session_entry_id
    FROM read_csv_auto('support/assurance/synthetic-content/ahc-hrsn-12-12-2023-valid.csv');

WITH required_column_names_in_src AS (
    SELECT column_name
      FROM (VALUES ('PAT_MRN_ID'), ('FACILITY'), ('FIRST_NAME'), ('LAST_NAME'), ('PAT_BIRTH_DATE'), ('MEDICAID_CIN'), ('ENCOUNTER_ID'), ('SURVEY'), ('SURVEY_ID'), ('RECORDED_TIME'), ('QUESTION'), ('MEAS_VALUE'), ('QUESTION_CODE'), ('QUESTION_CODE_SYSTEM_NAME'), ('ANSWER_CODE'), ('ANSWER_CODE_SYSTEM_NAME'), ('SDOH_DOMAIN'), ('NEED_INDICATED'), ('VISIT_PART_2_FLAG'), ('VISIT_OMH_FLAG'), ('VISIT_OPWDD_FLAG')) AS required(column_name)
     WHERE required.column_name NOT IN (
         SELECT upper(trim(column_name))
           FROM information_schema.columns
          WHERE table_name = 'ahc_hrsn_12_12_2023_valid')
)
INSERT INTO ingest_session_issue (ingest_session_issue_id, session_id, session_entry_id, issue_type, issue_message, remediation)
    SELECT uuid(),
           '05269d28-15ae-5bd6-bd88-f949ccfa52d7',
           'ae477ba1-c7f1-5f34-847a-50bddb7130aa',
           'Missing Column',
           'Required column ' || column_name || ' is missing in ahc_hrsn_12_12_2023_valid.',
           'Ensure ahc_hrsn_12_12_2023_valid contains the column "' || column_name || '"'
      FROM required_column_names_in_src;

-- required by IngestEngine, emit the errors for the given session (file) so it can be picked up
SELECT * FROM ingest_session_issue WHERE session_id = '05269d28-15ae-5bd6-bd88-f949ccfa52d7' and session_entry_id = 'ae477ba1-c7f1-5f34-847a-50bddb7130aa';
```


## ensureContent
```sql
WITH numeric_value_in_all_rows AS (
    SELECT 'SURVEY_ID' AS issue_column,
           SURVEY_ID AS invalid_value,
           src_file_row_number AS issue_row
      FROM ahc_hrsn_12_12_2023_valid
     WHERE SURVEY_ID IS NOT NULL
       AND SURVEY_ID NOT SIMILAR TO '^[+-]?[0-9]+$'
)
INSERT INTO ingest_session_issue (ingest_session_issue_id, session_id, session_entry_id, issue_type, issue_row, issue_column, invalid_value, issue_message, remediation)
    SELECT uuid(),
           '05269d28-15ae-5bd6-bd88-f949ccfa52d7',
           'ae477ba1-c7f1-5f34-847a-50bddb7130aa',
           'Data Type Mismatch',
           issue_row,
           issue_column,
           invalid_value,
           'Non-integer value "' || invalid_value || '" found in ' || issue_column,
           'Convert non-integer values to INTEGER'
      FROM numeric_value_in_all_rows;
```


## emitResources
```sql
INSERT INTO "ingest_session_state" ("ingest_session_state_id", "session_id", "session_entry_id", "from_state", "to_state", "transition_result", "transition_reason", "transitioned_at", "elaboration") VALUES ('05e8feaa-0bed-5909-a817-39812494b361', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', NULL, 'NONE', 'ENTER(init)', NULL, 'rsEE.beforeCell', (make_timestamp(2023, 11, 6, 17, 22, 33.879)), NULL);
INSERT INTO "ingest_session_state" ("ingest_session_state_id", "session_id", "session_entry_id", "from_state", "to_state", "transition_result", "transition_reason", "transitioned_at", "elaboration") VALUES ('8f460419-7b80-516d-8919-84520950f612', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', NULL, 'EXIT(init)', 'ENTER(ingest)', NULL, 'rsEE.afterCell', (make_timestamp(2023, 11, 6, 17, 22, 33.879)), NULL);
INSERT INTO "ingest_session_state" ("ingest_session_state_id", "session_id", "session_entry_id", "from_state", "to_state", "transition_result", "transition_reason", "transitioned_at", "elaboration") VALUES ('8aad9cfa-b1a2-5fb1-a6ab-613a79a7e839', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', NULL, 'EXIT(ingest)', 'ENTER(ensureContent)', NULL, 'rsEE.afterCell', (make_timestamp(2023, 11, 6, 17, 22, 33.879)), NULL);
INSERT INTO "ingest_session_state" ("ingest_session_state_id", "session_id", "session_entry_id", "from_state", "to_state", "transition_result", "transition_reason", "transitioned_at", "elaboration") VALUES ('b41ccd27-9a4f-5cc8-9c5d-b55242d90fb0', '05269d28-15ae-5bd6-bd88-f949ccfa52d7', NULL, 'EXIT(ensureContent)', 'ENTER(emitResources)', NULL, 'rsEE.afterCell', (make_timestamp(2023, 11, 6, 17, 22, 33.879)), NULL);

ATTACH 'support/assurance/results-test-e2e/resource.sqlite.db' AS resource_db (TYPE SQLITE);

CREATE TABLE resource_db.ingest_session AS SELECT * FROM ingest_session;
CREATE TABLE resource_db.ingest_session_entry AS SELECT * FROM ingest_session_entry;
CREATE TABLE resource_db.ingest_session_state AS SELECT * FROM ingest_session_state;
CREATE TABLE resource_db.ingest_session_issue AS SELECT * FROM ingest_session_issue;

-- {contentResult.map(cr => `CREATE TABLE resource_db.${cr.iaSqlSupplier.tableName} AS SELECT * FROM ${cr.tableName}`).join(";")};

DETACH DATABASE resource_db;
-- no after-finalize SQL provided
```


## emitDiagnostics
```sql
INSTALL spatial; LOAD spatial;
-- TODO: join with ingest_session table to give all the results in one sheet
COPY (SELECT * FROM ingest_session_diagnostic_text) TO 'support/assurance/results-test-e2e/diagnostics.xlsx' WITH (FORMAT GDAL, DRIVER 'xlsx');
```