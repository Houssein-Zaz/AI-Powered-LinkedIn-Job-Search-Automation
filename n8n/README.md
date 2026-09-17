# Workflow export

Export the live workflow from n8n and save it here as `workflow.json`:

1. Open the workflow in n8n
2. Click the `⋯` menu next to the workflow name (top left)
3. Choose **Export JSON**
4. Save the downloaded file into this folder as `workflow.json`

The export excludes credential values, so it is safe to commit. It keeps
credential *references* (names and types), which is what tells anyone importing
it which credentials they need to create.

Re-export and commit again whenever you change the workflow.
