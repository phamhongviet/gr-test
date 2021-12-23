# batch-jobs-min-nodes
Helm chart to deploy batch jobs that try to schedule pods in least number of nodes.

Please take a look at the default values.yaml file.

## Requirements
- Each release of the chart have a unique `jobsGroup`.
- `jobsGroup` value need to be the same as `jobs-group` in the `affinity` section. To enforce this, use helmfile.
