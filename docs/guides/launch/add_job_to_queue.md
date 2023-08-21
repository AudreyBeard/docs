---
displayed_sidebar: default
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Enqueue jobs

TEXT.


## Launch job names

By default, W&B automatically generates a job name for you. The name is generated depending on how the job is created (GitHub, code artifact, or Docker image). Alternatively, you can define a launch job's name with environment variables or with the W&B Python SDK.

### Default launch job names

The following table describes the job naming convention used by default based on job source:

| Source        | Naming convention                       |
| ------------- | --------------------------------------- |
| GitHub        | `job-<git-remote-url>-<path-to-script>` |
| Code artifact | `job-<code-artifact-name>`              |
| Docker image  | `job-<image-name>`                      |


### Name your launch job
Name your job with a W&B environment variable or with the W&B Python SDK

<Tabs
  defaultValue="env_var"
  values={[
    {label: 'Environment variable', value: 'env_var'},
    {label: 'W&B Python SDK', value: 'python_sdk'},
  ]}>
  <TabItem value="env_var">

Set the `WANDB_JOB_NAME` environment variable to your preferred job name. For example:

```bash
WANDB_JOB_NAME=awesome-job-name
```

  </TabItem>
  <TabItem value="python_sdk">

Define the name of your job with `wandb.Settings`. Then pass this object when you initialize W&B with `wandb.init`. For example:

```python
settings = wandb.Settings(job_name="my-job-name")
wandb.init(settings=settings)
```

  </TabItem>
</Tabs>


:::note
For docker image jobs, the image tag is automatically added as an alias to the job.
:::

## Add jobs to your queue

Add jobs to your queue interactively with the W&B App or programmatically with the W&B CLI.

<Tabs
  defaultValue="app"
  values={[
    {label: 'W&B App', value: 'app'},
    {label: 'W&B CLI', value: 'cli'},
  ]}>
  <TabItem value="app">
Add a job to your queue with the W&B App.

1. Navigate to your W&B Project Page.
2. Select the **Jobs** icon on the left panel:

![](/images/launch/project_jobs_tab_gs.png)

3. The **Jobs** page displays a list of W&B Jobs that were created from previously executed W&B Runs. 

![](/images/launch/view_jobs.png)

4. Select the **Launch** button next to the name of the Job name. A modal will appear on the right side of the page.
5. Within the modal select the:
    * Job version you want to add to your queue from the **Job version** dropdown. In this example we only have one version, so we select `v0`.
    * Select the **Paste from…** button to automatically propagate hyperparameters used from a specific W&B Run. In the following image, we have two runs to choose from.

![](/images/launch/create_starter_queue_gs.png)

6. Next, select **Starter queue** from the **Queue** dropdown to create a queue.
7. Select the **Launch now** button. 


  </TabItem>
    <TabItem value="cli">

Use the `wandb launch` command to add jobs to a queue. Create a JSON configuration with hyperparameter overrides. For example, using the script from the [Quickstart](./walkthrough.md) guide, we create a JSON file with the following overrides:

```json
// config.json
{
    "args": [],
    "run_config": {
        "learning_rate": 0,
        "epochs": 0
    },
    "entry_point": []
}
```
W&B Launch will use the default parameters if you do not provide a JSON configuration file.


Provide the name of the queue for the `queue`(`-q`) flag, the name of the job for the `job`(`-j`) flag, and the path to the configuration file for the `config`(`-c`) flag.

```bash
wandb launch -j <job> -q <queue-name> -e <entity-name> -c path/to/config.json
```
If you work within a W&B Team, we suggest you specify the `entity` flag (`-e`) to indicate which entity the queue will use.

  </TabItem>
</Tabs>


