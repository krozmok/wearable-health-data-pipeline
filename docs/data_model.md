# Data Model

Model for the PMData synthetic dataset: 50 participants, 28 days
(2026-01-08 → 2026-02-04), study timezone UTC-05:00.
Diagram: [modeling/main_model.dbml](modeling/main_model.dbml).
Source field definitions: [../data_dictionary.md](../data_dictionary.md).

## Layers

| Layer | Tables | Grain |
|---|---|---|
| Source (CSV) | `participants`, `device_metadata`, `wellness_survey` | participant / device / participant-day |
| Source (JSON) | `sleep`, `steps`, `heart_rate` | participant-night / participant-minute |
| Flattened | `sleep_sessions`, `sleep_stages` | participant-night / stage |
| Consumption | `dim_participant`, `daily_participant_summary`, `weekly_participant_summary` | participant / participant-day / participant-week |

`sleep` mirrors `sleep.json` on disk, with stages as a nested array. It is flattened
into `sleep_sessions` (session fields) and `sleep_stages` (one row per stage).
`steps` and `heart_rate` are already flat.

## Consumption tables

### `dim_participant`

One row per participant: `participants` joined to `device_metadata`.
`firmware_version` is parsed from `device_label`, so it is filled even where the
source column is blank.

### `daily_participant_summary`

One row per `(participant_id, date)`, for teams that ingest daily. Combines
activity, heart rate, sleep and wellness, so no joins are needed.

- Built after cleaning: timestamps normalised to UTC-05:00, stuck-sensor
  heart-rate readings excluded.
- Sleep and survey are attributed to the wake-up / submission date.
- Each batch overwrites the date partition and rebuilds the trailing days, so
  re-runs and late data are safe.
- Carries sums and counts (`hr_sum_bpm`, `hr_valid_minutes`, stage minutes) as
  well as averages, so the weekly table can be derived from it.

### `weekly_participant_summary`

One row per `(participant_id, study_week)`, for teams that need weekly data.
Derived only from the daily table; minute-level sources are not re-read.

- Study weeks run Thursday to Wednesday, anchored to the study start, giving
  four complete weeks: `study_week = floor((date - 2026-01-08) / 7) + 1`.
- Averages are weighted from daily sums and counts, not averages of averages:
  `avg_hr_bpm = sum(hr_sum_bpm) / sum(hr_valid_minutes)`.
- `days_with_data`, `nights_recorded` and `surveys_submitted` show coverage;
  `is_complete_week` is true once all 7 days are loaded.
- Medians and percentiles cannot be derived from daily rows; they need the
  minute-level tables.

## Metric definitions

| Metric | Definition |
|---|---|
| `active_minutes` | Minutes with `steps` > 0 |
| `total_sleep_min` | Sum of `duration_min` over non-awake stages |
| `deep_sleep_pct` | `deep_min / total_sleep_min * 100` |
| `sleep_efficiency_pct` | `total_sleep_min / (total_sleep_min + awake_min) * 100`, from the stages |
| `resting_hr_bpm` | Average valid heart rate during the sleep session ending on `date` |
| `hr_valid_minutes` | Minutes with an in-range reading not flagged as stuck sensor |

## Open items

- The rule for flagging a stuck heart-rate sensor (how many repeated minutes
  count as signal loss) is not defined yet; `hr_valid_minutes` and
  `hr_stuck_minutes` depend on it.
- Primary keys describe the intended grain. The raw data may contain duplicates
  or mixed timestamp formats that violate them before cleaning.
