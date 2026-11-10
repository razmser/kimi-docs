> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Using the Console for Batch Inference

> Create, monitor, and download batch inference results from the Kimi Open Platform console without writing code.

Batch inference allows you to submit large-scale inference tasks through the Kimi Platform console without writing any code. This tutorial shows how to create, monitor, and retrieve results of batch inference tasks from the console.

> [!NOTE]
>
> Batch inference is conducted for a specific project and is only available to users at Tier1 or above. If you prefer to use the API for batch processing, see the [Batch API Guide](use-batch-api.md).

## Steps

### 1. Create a Batch Task

Open the [Kimi Platform](https://platform.kimi.ai), go to **User Center** → **Projects** → **View Project** → **Batches** → **Create Batches**.

![Project list](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step1-project-list.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=cc7d36288c674eeef97ca35e00ea0aec)

![Batches page](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step1-batch-page.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=7e5f7bcffb0ec2734c743ec32f701f8d)

### 2. Configure Task Parameters

In the Create Batch Task dialog, set the following:

* **Batch Name**: Set a name for the task
* **Completion Window**: Select the maximum wait time for the task
* **Data File**: Upload a new file or select an existing file

Click **OK** to submit the task.

![Create Batch Task](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step2-create-task.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=230dfb53a04c5c019a072d16b9808dd4)

### 3. Wait for Inference to Complete

After submission, the task starts running. You can monitor the task status in the Batches list.

![Task validating](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step3-task-running.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=50d7068074e7b1aad83de0e666f493c5)

### 4. Download Output Results

After execution is completed, click **Details** to view the task details and download the output file.

![Task completed](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step4-task-completed.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=ed41ffa52536ec8df532d8f42b5d8774)

![Batch details](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step4-task-details.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=c0fe800070183be4d9d982448a973cf5)

### 5. View File History

You may also find your previously uploaded files and output results in the project's **Files** page.

![Files list](https://mintcdn.com/moonshotai/O578FwQ-LzmT8N5y/assets/pics/batch-inference/step5-files.png?fit=max\&auto=format\&n=O578FwQ-LzmT8N5y\&q=85\&s=1b23b0667ca98609d2323e7695621c39)
