# eq-concourse-peformance-testing

This repo holds the pipeline to performance test eQ Survey Runner.

## Authenticate with Concourse

Log in to concourse with:

```
fly -t census-eq login -c <concourse_url>
```

This will configure the target `census-eq` to point to the eq team on Concourse.

## Development Environments

#### Prerequisites
You must have a project with a *Google Container Registry (GCR)* already set up to be able to push built images.
Set your registry in *variables.yaml* to `eu.gcr.io/<gcp_project_id>`. The `gcp_project_id` should be the ID of the GCP project under which the container registry is hosted.

#### Scheduled pipeline triggers
The first time you create a pipeline that depend on a time resource, you *will not* be able to trigger tasks that depend on these time resources because there will be no available versions of time resource. In our case, a version of this resource is only created at the time specified by the pipeline, for example, the benchmark-timed-schedule pipeline is initiated at 3 AM.
Therefore, for development purposes, you will need to set the source start time to a short while in the future so that you have enough time to create and unpause the pipeline. This only needs to be done once to kick start the initial build, after which this can be reset to default.

**Resource for benchmark-timed-schedule.yaml**
```yaml
- name: scheduled-benchmark-trigger
    type: time
    source:
      start: 03:00 AM
      stop: 04:00 AM
      location: Europe/London
```

#### Pipelines

Pipeline | Schedule | Notes |
--- | --- | --- |
benchmark-timed-schedule.yaml | 3 AM every day | Deploys infrastructure required to carries out a performance test using eq-survey-runner-benchmark for which the outputs are stored in a GCS bucket. Once complete the infrastructure is torn down ready for the next scheduled test.

#### Deployment process

To ***create*** a pipeline to deploy your own environments (in order to test pipelines etc.) you can do so as follows:

1. Fill in any missing variables specified in *variables.yaml* *(copied from variables.yaml.example)*. Ask a team member for help with the values.
1. Upload your pipeline to Concourse using a command like:

    ```
    fly -t census-eq set-pipeline -p <PIPELINE_NAME> -c <PIPELINE_CONFIG_FILE> --load-vars-from variables.yaml
    ```
   *For example*
    ```bash
    fly -t census-eq set-pipeline -p my-benchmark-pipeline -c benchmark-timed-schedule.yaml --load-vars-from variables.yaml
    ```
    This will create a pipeline called *my-benchmark-pipeline*. This will also load a few templating variables from *variables.yaml*. There is a specification for this in the repo.
1. Navigate to the Concourse UI and unpause your pipeline to trigger the builds. Alternatively you can run:
    ```
    fly -t census-eq unpause-pipeline -p <PIPELINE_NAME>
    ```

---

To ***destroy*** your pipeline, you can run the following command:
```
fly -t census-eq destroy-pipeline -p <PIPELINE_NAME>
```

---

## Secrets

Secrets are currently hooked into Kubernetes secrets, which are available on the cluster. To get a full list of secrets available for use within the pipeline, you can log in to the cluster and get the secrets.

Log in to the cluster using:
```
gcloud container clusters get-credentials k8s-cd --region <region> --project <gcp_project_id>
```
Get the [Kubernetees secrets](https://kubernetes.io/docs/concepts/configuration/secret/) using:
```bash
kubectl --namespace=concourse-main get secrets
```

## Slack

Pipeline deployment notifications are alerted to the slack channel defined by `slack_channel_name` in *variables.yaml*. *(Do not include the `#`).*