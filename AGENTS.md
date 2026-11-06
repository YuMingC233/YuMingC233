# Repository guidance

## Machine inventory

The public migration inventory for the Ubuntu workstation is in `docs/machine-inventory/README.md`. Its maintenance instructions are in `docs/machine-inventory/MAINTENANCE.md`.

When work on this workstation changes installed software, drivers, input methods, desktop extensions, runtime environments, or startup behavior, update the affected inventory records when this repository is available. The owner prefers updates alongside environment changes, with no scheduled inventory job. Verify the current machine before refreshing facts; do not treat dated inventory as current evidence.

Publish only reviewed package metadata and configuration descriptions. Keep credentials, private network endpoints, personal dictionaries, application data, raw configuration dumps, and operational logs out of this public repository.

Preserve unrelated working-tree changes. Stage only the files relevant to the current task. Documentation-only inventory updates require link, table, factual, and secret checks rather than new unit tests. Ask the owner before introducing unit tests for functional code changes. Do not add AI attribution to commit messages.
