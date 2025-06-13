# SQL used in the blog
This page describes the SQL used in the blog, to be applied to the results.db files produced by the executions of SAP AI Launchpad, to verify the completion of the executions and as well to draw additional insights.
For learning purposes, can be used already on the results.db file provided in this repository.

## Find Number of submitted runs
```
SELECT count(*) FROM submission
```

## Find Number of Obtained Results
```
SELECT count(*) FROM submission_result
```

## Find Dropped samples
```
SELECT
    json_extract(s.template_variables, '$.Phrase') AS phrase,
    json_extract(s.template_variables, '$.goldenTruth') AS goldenTruth
FROM submission s
LEFT JOIN submission_result sr ON s.id = sr.submission_id
WHERE sr.submission_id IS NULL;
```

## Visualising Phrases and BERTScore Precision and Recall
```
SELECT
	r.name AS run_name,
    json_extract(s.template_variables, '$.Phrase') AS phrase,
    json_extract(s.template_variables, '$.goldenTruth') AS goldenTruth,
    json_extract(sr.completion_result, '$.orchestration_result.choices[0].message.content') AS content,
	json_extract(er.metric_result, '$.precision') AS precision,
	json_extract(er.metric_result, '$.recall') AS recall
FROM submission s
LEFT JOIN submission_result sr ON s.id = sr.submission_id 
LEFT JOIN evaluation_result er ON s.id = er.submission_id 
LEFT JOIN run r ON s.run_id = r.id;

## Create an evaluation table

CREATE TABLE "myResult" (
"run_name"	TEXT,
	"goldenTruth" TEXT,
	"content"	TEXT ,
	"precision"	,
	"recall"
)
```
## Store the Phrase and the goldenTruth components in separate columns

```
UPDATE myResult
SET
  gt_Drug = substr(
    goldenTruth,
    instr(goldenTruth, '|') + 1,
    instr(substr(goldenTruth, instr(goldenTruth, '|') + 1), '|') - 1
  ),
  
  gt_Dose = substr(
    goldenTruth,
    instr(goldenTruth, '|') + 1 + 
      instr(substr(goldenTruth, instr(goldenTruth, '|') + 1), '|'),
    instr(substr(substr(goldenTruth, instr(goldenTruth, '|') + 1 + 
      instr(substr(goldenTruth, instr(goldenTruth, '|') + 1), '|')),
      1), '|') - 1
  ),

  gt_AdverseEvent = substr(
    goldenTruth,
    instr(goldenTruth, '|') + 1 + 
      instr(substr(goldenTruth, instr(goldenTruth, '|') + 1), '|') + 
      instr(substr(substr(goldenTruth, instr(goldenTruth, '|') + 1 + 
        instr(substr(goldenTruth, instr(goldenTruth, '|') + 1), '|')), 1), '|')
  );

UPDATE myResult
SET
  ai_Drug = substr(
    content,
    instr(content, '|') + 1,
    instr(substr(content, instr(content, '|') + 1), '|') - 1
  ),
  
  ai_Dose = substr(
    content,
    instr(content, '|') + 1 + 
      instr(substr(content, instr(content, '|') + 1), '|'),
    instr(substr(substr(content, instr(content, '|') + 1 + 
      instr(substr(content, instr(content, '|') + 1), '|')),
      1), '|') - 1
  ),

  ai_AdverseEvent = substr(
    content,
    instr(content, '|') + 1 + 
      instr(substr(content, instr(content, '|') + 1), '|') + 
      instr(substr(substr(content, instr(content, '|') + 1 + 
        instr(substr(content, instr(content, '|') + 1), '|')), 1), '|')
  );
```
## Generate the confusion matrix 

```
UPDATE myResult
SET
  -- Drug
  TP_Drug = CASE WHEN ai_Drug = gt_Drug THEN 1 ELSE 0 END,
  TN_Drug = CASE WHEN ai_Drug IS NULL AND gt_Drug IS NULL THEN 1 ELSE 0 END,
  FP_Drug = CASE WHEN ai_Drug IS NOT NULL AND ai_Drug != gt_Drug THEN 1 ELSE 0 END,
  FN_Drug = CASE WHEN ai_Drug IS NULL AND gt_Drug IS NOT NULL THEN 1 
                 WHEN ai_Drug != gt_Drug THEN 1 
                 ELSE 0 END,

  -- Dose
  TP_Dose = CASE WHEN ai_Dose = gt_Dose THEN 1 ELSE 0 END,
  TN_Dose = CASE WHEN ai_Dose IS NULL AND gt_Dose IS NULL THEN 1 ELSE 0 END,
  FP_Dose = CASE WHEN ai_Dose IS NOT NULL AND ai_Dose != gt_Dose THEN 1 ELSE 0 END,
  FN_Dose = CASE WHEN ai_Dose IS NULL AND gt_Dose IS NOT NULL THEN 1 
                 WHEN ai_Dose != gt_Dose THEN 1 
                 ELSE 0 END,

  -- Adverse Event
  TP_AdverseEvent = CASE WHEN ai_AdverseEvent = gt_AdverseEvent THEN 1 ELSE 0 END,
  TN_AdverseEvent = CASE WHEN ai_AdverseEvent IS NULL AND gt_AdverseEvent IS NULL THEN 1 ELSE 0 END,
  FP_AdverseEvent = CASE WHEN ai_AdverseEvent IS NOT NULL AND ai_AdverseEvent != gt_AdverseEvent THEN 1 ELSE 0 END,
  FN_AdverseEvent = CASE WHEN ai_AdverseEvent IS NULL AND gt_AdverseEvent IS NOT NULL THEN 1 
                          WHEN ai_AdverseEvent != gt_AdverseEvent THEN 1 
                          ELSE 0 END;

```
## Calculate the Precision / Recall and F1 scores
```
SELECT
  run_name,
  'Drug' AS field,
  SUM(TP_Drug) AS TP,
  SUM(FP_Drug) AS FP,
  SUM(FN_Drug) AS FN,
  SUM(TN_Drug) AS TN,
  ROUND(1.0 * SUM(TP_Drug) / NULLIF(SUM(TP_Drug) + SUM(FP_Drug), 0), 3) AS precision,
  ROUND(1.0 * SUM(TP_Drug) / NULLIF(SUM(TP_Drug) + SUM(FN_Drug), 0), 3) AS recall,
  ROUND(
    2.0 * (1.0 * SUM(TP_Drug)) /
    NULLIF(2.0 * SUM(TP_Drug) + SUM(FP_Drug) + SUM(FN_Drug), 0),
    3
  ) AS f1_score
FROM myResult
GROUP BY run_name

UNION ALL

SELECT
  run_name,
  'Dose' AS field,
  SUM(TP_Dose) AS TP,
  SUM(FP_Dose) AS FP,
  SUM(FN_Dose) AS FN,
  SUM(TN_Dose) AS TN,
  ROUND(1.0 * SUM(TP_Dose) / NULLIF(SUM(TP_Dose) + SUM(FP_Dose), 0), 3) AS precision,
  ROUND(1.0 * SUM(TP_Dose) / NULLIF(SUM(TP_Dose) + SUM(FN_Dose), 0), 3) AS recall,
  ROUND(
    2.0 * (1.0 * SUM(TP_Dose)) /
    NULLIF(2.0 * SUM(TP_Dose) + SUM(FP_Dose) + SUM(FN_Dose), 0),
    3
  ) AS f1_score
FROM myResult
GROUP BY run_name

UNION ALL

SELECT
  run_name,
  'AdverseEvent' AS field,
  SUM(TP_AdverseEvent) AS TP,
  SUM(FP_AdverseEvent) AS FP,
  SUM(FN_AdverseEvent) AS FN,
  SUM(TN_AdverseEvent) AS TN,
  ROUND(1.0 * SUM(TP_AdverseEvent) / NULLIF(SUM(TP_AdverseEvent) + SUM(FP_AdverseEvent), 0), 3) AS precision,
  ROUND(1.0 * SUM(TP_AdverseEvent) / NULLIF(SUM(TP_AdverseEvent) + SUM(FN_AdverseEvent), 0), 3) AS recall,
  ROUND(
    2.0 * (1.0 * SUM(TP_AdverseEvent)) /
    NULLIF(2.0 * SUM(TP_AdverseEvent) + SUM(FP_AdverseEvent) + SUM(FN_AdverseEvent), 0),
    3
  ) AS f1_score
FROM myResult
GROUP BY run_name

ORDER BY run_name, field;
```
