# Onboarding a New Sport or League

Checklist for adding a new sport/league to the pipeline.
The pipeline handles it automatically — this is the surrounding work.

## Analytics (collaborator)

- [ ] Create `egress/sport/league/` folder
- [ ] Push first dataset files to `egress/sport/league/season/`

## Pipeline (Logan)

- [ ] Write schema TOML contracts for each new dataset at `schema/sport/league/dataset_name.toml`
- [ ] Decide write mode and primary keys per dataset with collaborator
- [ ] Add transform logic in `transform/json_shaper.py` for Gold JSON shape per dataset
- [ ] Confirm Gold JSON shape with collaborator — what does the web page need to display

## Web (Logan)

- [ ] Add Flask routes at `routes/sport/league/`
- [ ] Add Jinja templates at `templates/sport/league/`

## Verify

- [ ] Push a test file through the full pipeline end to end
- [ ] Confirm Silver parquet lands at `parquet/sport/league/table/season/`
- [ ] Confirm Gold JSON lands at `web/sport/league/entity/`
- [ ] Confirm Discord notification fires with correct run summary
- [ ] Confirm Render redeploys and Flask serves the new pages
