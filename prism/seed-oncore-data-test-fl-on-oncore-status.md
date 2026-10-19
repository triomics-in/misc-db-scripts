# E2E test: PATIENT_ENROLLMENT filtering logic driven by OnCore status

Companion to `docs/ctms-enrolment-in-filtering-logic.md`. Verifies that a patient who is enrolled in
a non-observational study **per OnCore only** (no ENROLLED subtask in Prism) makes the "not enrolled
in another non-observational study" criterion fail on a sibling study, so that subtask goes
NOT_RELEVANT.

Placeholders: `<TENANT_ID>`, `<PATIENT_ID>` (business id), `<STUDY_A>` (OnCore study the patient is
enrolled in), `<STUDY_B>` (sibling study whose subtask should flip), `<PRISM_HOST>`, `<ADMIN_HOST>`,
`<TOKEN>`, `<MEMBER_ID>`.

## 0. Preconditions

1. Prism build contains `feature/PRISM-4306-fl-on-oncore-status`.
2. The tenant has the CTMS migrations (`ctms_patient_data_sink` exists). Without them the OnCore read
   would fail inside the refresh transaction; the tenant flag is what keeps it from running there.
3. Tenant flag on, and Prism's 4 h cache of it cleared:
   ```sql
   UPDATE tmx_admin.tmx_organisation
   SET org_settings = COALESCE(org_settings, '{}'::jsonb) || '{"prism.ctmsEnabled": "true"}'::jsonb
   WHERE tenant_id = '<TENANT_ID>' AND deleted = false;
   ```
   ```
   redis-cli -h <app.cache.redis.host> -n <app.cache.redis.db> DEL 'ctmsEnabledCache::<TENANT_ID>'
   ```
4. Reporting views current (otherwise the patients list 500s with `column ... does not exist`):
   ```
   psql "<TENANT_DB_URL>" -v ON_ERROR_STOP=1 \
     -f src/main/resources/db/changelog/migrations_gen_01/sql/prism/2/prism-views-psme-2.0.0-1.sql
   ```
5. The `PATIENT_ENROLLMENT` SM entity exists for the tenant. It is never created by a migration; the
   refresh throws `SMEntityAttribute not found for ID: PATIENT_ENROLLMENT.enrolledInNonObservationalStudy`
   without it. Seed through the bulk-create API (creates entity, attribute and the Yes/No value set):
   ```bash
   curl -X POST 'https://<PRISM_HOST>/prism/vista/api/v1.0/smentities' \
     -H 'Authorization: Bearer <TOKEN>' -H 'X-TMX-TENANT-ID: <TENANT_ID>' -H 'X-TMX-MEMBER-ID: <MEMBER_ID>' \
     -H 'Content-Type: application/json' \
     -d '{"entities":[{"entityId":"PATIENT_ENROLLMENT","entityName":"Patient Enrollment",
          "isEntityRepeating":false,"order":99,"considerNoDataPscAs":"UNKNOWN",
          "attributes":[{"attributeName":"enrolledInNonObservationalStudy",
            "label":"Enrolled in non-observational study","attributeValueType":"STRING_CHOICE",
            "attributeValueChoices":["Yes","No"],"level":1,
            "isVisibleInReport":false,"isPrimaryAttribute":false,"isMappedToEsIndex":false}]}]}'
   ```
   ```sql
   SELECT e.sm_entity_id, a.sm_entity_attribute_id, a.value_type, a.value_set_id
   FROM tmx_prism_trialfinder.smentity e
   JOIN tmx_prism_trialfinder.smentity_attribute a ON a.fk_sm_entity_db_id = e.db_id
   WHERE e.tenant_id = '<TENANT_ID>' AND e.sm_entity_id = 'PATIENT_ENROLLMENT';
   ```

## 1. Study A: an OnCore-linked, non-observational, enabled study

```sql
SELECT study_id, protocol_id, ctms_protocol_no, clinical_trial_type, clinical_trial_category, availability_status
FROM tmx_admin.tmx_clinical_study
WHERE tenant_id = '<TENANT_ID>' AND deleted = false AND ctms_protocol_id IS NOT NULL
  AND clinical_trial_type IS DISTINCT FROM 'OBSERVATIONAL'
  AND availability_status IS DISTINCT FROM 'DISABLED';
```

## 2. Study B: a different enabled, non-observational study with an enabled cohort on its latest version

The criterion excludes the study it sits on, so B must not be A. The CTE also requires a RELEVANT,
non-ENROLLED subtask to exist on B and skips studies that already carry the rule.

```sql
WITH candidate_studies AS (
  SELECT lv.study_id
  FROM tmx_prism_trialfinder_reporting.latest_study_version_view lv
  JOIN tmx_admin.tmx_clinical_study s
    ON s.db_id = lv.study_db_id AND s.tenant_id = lv.tenant_id AND s.deleted = false
   AND s.availability_status = 'ENABLED' AND s.clinical_trial_type IS DISTINCT FROM 'OBSERVATIONAL'
  JOIN tmx_admin.tmx_study_version sv
    ON sv.version_id = lv.latest_version_id AND sv.fk_study_db_id = lv.study_db_id
   AND sv.tenant_id = lv.tenant_id AND sv.status = 'PUBLISHED' AND sv.deleted = false
  JOIN tmx_admin.cohort c
    ON c.fk_study_version_db_id = sv.db_id AND c.tenant_id = sv.tenant_id AND c.deleted = false
  LEFT JOIN tmx_admin.cohort_criteria_link ccl ON ccl.fk_cohort_db_id = c.db_id AND ccl.deleted = false
  LEFT JOIN tmx_admin.cohort_criterion enr
    ON enr.db_id = ccl.fk_cohort_criterion_db_id AND enr.deleted = false
   AND enr.criterion_detail->>'smEntityId' = 'PATIENT_ENROLLMENT'
  WHERE lv.tenant_id = '<TENANT_ID>' AND lv.latest_version_id IS NOT NULL AND lv.study_id <> '<STUDY_A>'
  GROUP BY lv.tenant_id, lv.study_id
  HAVING COUNT(c.db_id) FILTER (WHERE c.cohort_availability_status = 'ENABLED') > 0
     AND bool_or(enr.db_id IS NOT NULL) = false
     AND EXISTS (SELECT 1 FROM tmx_prism_trialfinder.patient_review_subtask prs
                 WHERE prs.tenant_id = lv.tenant_id AND prs.study_id = lv.study_id
                   AND prs.deleted = false AND prs.relevancy = 'RELEVANT' AND prs.status <> 'ENROLLED')
)
SELECT lv.study_id, c.cohort_name, cc.db_id,
       cc.criterion_detail->>'variableName' AS var, cc.criterion_detail->>'smEntityId' AS sm_entity,
       cc.criterion_detail->>'criterionText' AS text
FROM tmx_prism_trialfinder_reporting.latest_study_version_view lv
JOIN candidate_studies cs ON cs.study_id = lv.study_id
JOIN tmx_admin.tmx_study_version sv
  ON sv.version_id = lv.latest_version_id AND sv.fk_study_db_id = lv.study_db_id AND sv.tenant_id = lv.tenant_id
JOIN tmx_admin.cohort c ON c.fk_study_version_db_id = sv.db_id AND c.tenant_id = sv.tenant_id AND c.deleted = false
JOIN tmx_admin.cohort_criteria_link ccl ON ccl.fk_cohort_db_id = c.db_id AND ccl.deleted = false
JOIN tmx_admin.cohort_criterion cc ON cc.db_id = ccl.fk_cohort_criterion_db_id AND cc.deleted = false
WHERE lv.tenant_id = '<TENANT_ID>'
  AND cc.criterion_type = 'FILTERING_LOGIC' AND cc.criterion_node_type = 'STANDARD'
  AND cc.criterion_detail->>'smEntityId' = 'VITAL_STATUS'
ORDER BY lv.study_id, c.cohort_name;
```

The rows returned are the "Vital Status = Alive" filtering-logic children on B's cohorts, which step 3
repurposes.

## 3. Put the enrolment criterion on B's cohorts

Proper way: the Admin migration endpoint (B must be category GENERIC and non-observational):

```bash
curl -X POST '<ADMIN_HOST>/admin/v1/administration/internal/migration/study-filtering-logic/<STUDY_B>/enrolled-in-non-observational-study' \
  -H 'Authorization: Bearer <TOKEN>' -H 'X-TMX-TENANT-ID: <TENANT_ID>' -H 'X-TMX-MEMBER-ID: <MEMBER_ID>'
```

Test shortcut used instead: repurpose the existing filtering-logic child from step 2. Text and type
alone are not enough; the engine routes on `smEntityId` and reads the EQUALS "No" condition.

```sql
CREATE TABLE tmx_admin.e2e_fl_backup AS
SELECT cc.db_id, cc.criterion_type, cc.criterion_detail
FROM tmx_admin.cohort_criterion cc WHERE cc.db_id IN (<db_ids from step 2>);

UPDATE tmx_admin.cohort_criterion cc
SET criterion_type   = 'FILTERING_LOGIC',
    criterion_detail = cc.criterion_detail || jsonb_build_object(
        'smEntityId',     'PATIENT_ENROLLMENT',
        'criterionText',  'Patient is not enrolled in another non-observational study',
        'criterionLabel', 'Enrolled in non-observational study',
        'smEntityAttributeOperatorValueDetailList', jsonb_build_array(jsonb_build_object(
            'smEntityId', 'PATIENT_ENROLLMENT', 'smEntityName', 'Patient Enrollment',
            'smEntityAttributeId', 'PATIENT_ENROLLMENT.enrolledInNonObservationalStudy',
            'smEntityAttributeName', 'Enrolled in non-observational study',
            'smEntityAttributeValueType', 'STRING_CHOICE', 'stdVarValueMatchOp', 'EQUALS', 'value', 'No'))),
    updated_by = 'E2E_SEED_FL', updated_on = now()
WHERE cc.tenant_id = '<TENANT_ID>' AND cc.db_id IN (SELECT db_id FROM tmx_admin.e2e_fl_backup);
```

Prism's refresh reads cohorts with an id filter that bypasses Admin's cached full report, so the
change is live at once.

## 4. Patient: RELEVANT non-ENROLLED subtask on B, no ENROLLED subtask anywhere, no real OnCore enrolment

```sql
SELECT p.patient_id, string_agg(prs.study_id || ':' || prs.status || ':' || prs.relevancy, ', ') AS subtasks
FROM tmx_prism_trialfinder.patient_review_subtask prs
JOIN tmx_prism_trialfinder.patient_review_task prt
  ON prt.db_id = prs.fk_patient_review_task_db_id AND prt.tenant_id = prs.tenant_id
 AND prt.deleted = false AND prt.status <> 'CLOSED'
JOIN tmx_prism_trialfinder.patient p ON p.db_id = prt.fk_patient_db_id AND p.tenant_id = prt.tenant_id
WHERE prs.tenant_id = '<TENANT_ID>' AND prs.deleted = false
GROUP BY p.patient_id
HAVING bool_or(prs.study_id = '<STUDY_B>' AND prs.relevancy = 'RELEVANT' AND prs.status <> 'ENROLLED')
   AND bool_and(prs.status <> 'ENROLLED')
LIMIT 10;

SELECT DISTINCT ON (study_id) study_id, ctms_raw_status, observed_at
FROM tmx_prism_trialfinder_reporting.ctms_patient_data_sink
WHERE tenant_id = '<TENANT_ID>' AND prism_patient_id = '<PATIENT_ID>' AND matched = true
ORDER BY study_id, observed_at DESC, db_id DESC;   -- expect no enrolled-type status
```

B's subtask must sit on the latest version (the one edited in step 3). Check
`patient_review_subtask.study_version_id` against `latest_study_version_view.latest_version_id`.

## 5. Snapshot before (rerun as step 8)

```sql
SELECT prs.study_id, prs.study_version_id, prs.status, prs.relevancy, psc.psc_status, psc.observation_text
FROM tmx_prism_trialfinder.patient_review_task prt
JOIN tmx_prism_trialfinder.patient p ON p.db_id = prt.fk_patient_db_id AND p.tenant_id = prt.tenant_id
JOIN tmx_prism_trialfinder.patient_review_subtask prs
  ON prs.fk_patient_review_task_db_id = prt.db_id AND prs.tenant_id = prt.tenant_id AND prs.deleted = false
LEFT JOIN tmx_prism_trialfinder.patient_study_criterion psc
  ON psc.fk_patient_review_subtask_db_id = prs.db_id AND psc.tenant_id = prs.tenant_id AND psc.deleted = false
 AND psc.criterion_hr_text ILIKE '%non-observational study%'
WHERE prt.tenant_id = '<TENANT_ID>' AND p.patient_id = '<PATIENT_ID>' AND prt.status <> 'CLOSED' AND prt.deleted = false
ORDER BY prs.study_id;
```

## 6. Seed the OnCore enrolment in study A (sink row, ON STUDY)

```sql
INSERT INTO tmx_prism_trialfinder_reporting.ctms_patient_data_sink
  (db_id, tenant_id, ctms_id, sync_run_id, observed_at, event_type,
   prism_patient_id, study_id, ctms_protocol_id, ctms_protocol_subject_id, matched,
   ctms_raw_status, prism_status, on_study_date, has_consented, created_on, created_by, deleted, version)
VALUES
  ((extract(epoch FROM clock_timestamp()) * 1000000)::bigint,
   '<TENANT_ID>', 'ONCORE', 'E2E-SEED-FL-ONCORE', now(), 'BASELINE',
   '<PATIENT_ID>', '<STUDY_A>',
   (SELECT ctms_protocol_id FROM tmx_admin.tmx_clinical_study WHERE study_id = '<STUDY_A>' AND tenant_id = '<TENANT_ID>'),
   900000001, true, 'ON STUDY', 'ENROLLED', current_date - 30, true, now(), 'SYSTEM_USER', false, 0);
```

## 7. Run the filtering logic

```bash
curl -X POST 'https://<PRISM_HOST>/prism/trialfinder/api/v1.0/patients/<PATIENT_ID>/evaluate-relevant-studies' \
  -H 'Authorization: Bearer <TOKEN>' -H 'X-TMX-TENANT-ID: <TENANT_ID>' -H 'X-TMX-MEMBER-ID: <MEMBER_ID>' \
  -H 'Content-Type: application/json' -d '{"versionUpdateSetting": "DONT_UPDATE"}'
```

`DONT_UPDATE` evaluates each subtask on the version it already sits on and changes nothing else. The
other values move subtasks to the latest version first and clear their status, eligibility and
evaluation metadata, which adds noise to the snapshot. Use `UPDATE_BASED_ON_TASK_AND_SUBTASK_STATUS`
only if B's subtask is on an older version than the one edited in step 3.

## 8. Expected after

- B's subtask: `relevancy = NOT_RELEVANT`; criterion row `psc_status = DOES_NOT_MEET`; observation text
  "Data present in Patient enrollment. Enrolled in non-observational study: True. Enrolled Study:
  <protocol of A> (<trial type>)".
- A's subtask, if any: unchanged (the study a patient is enrolled in is never failed by its own enrolment).
- Observational sibling studies: unchanged.
- Prism log: no `[CTMS]` warning during the call.

## 9. Controls (each followed by step 7 again)

- Newest row wins: insert a second sink row like step 6 with `ctms_raw_status = 'OFF STUDY'`,
  `event_type = 'STATUS_CHANGE'`, `off_study_date = current_date`. B returns to RELEVANT.
- Observational study as A: no change on B.
- `prism.ctmsEnabled` = "false" plus Redis key deleted: no change on B.

## 10. Cleanup

```sql
DELETE FROM tmx_prism_trialfinder_reporting.ctms_patient_data_sink
WHERE tenant_id = '<TENANT_ID>' AND sync_run_id = 'E2E-SEED-FL-ONCORE';

UPDATE tmx_admin.cohort_criterion cc
SET criterion_type = b.criterion_type, criterion_detail = b.criterion_detail, updated_on = now()
FROM tmx_admin.e2e_fl_backup b WHERE b.db_id = cc.db_id;
DROP TABLE tmx_admin.e2e_fl_backup;
```

Run step 7 once more so B's relevancy is recomputed back. The seed never writes
`patient_review_subtask` or `patient_study_criterion`; only the refresh does.

## Errors seen and their fixes

| Error | Cause | Fix |
|---|---|---|
| `column op1_0.disease_progression does not exist` on `orchestrated_patient_list_view` | views on the environment behind the deployed code | precondition 4 |
| `SMEntityAttribute not found for ID: PATIENT_ENROLLMENT.enrolledInNonObservationalStudy` | SM entity never seeded on the tenant | precondition 5 |
