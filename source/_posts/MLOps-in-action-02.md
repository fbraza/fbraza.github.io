---
title: From days to hours. How we speed up our data preparation pipelines at Sunrise.
summary: From days to hours. How we speed up our data preparation pipelines at Sunrise.
date: "2025-09-25"
tags:
    - Python
    - Cloud
---

The objective was clear. We needed to speep up our iteration cycles. From raw data and labels to the prediction outputs that will help to training and improving our model.

## Cleaning the messy data organisation and going to the cloud

First, we cleaned up our data management. We decided to centralize in our data lake all the data from spread over different buckets, dropox folders and hard drive in one bucket: the training bucket. This was our single source of truth, our Bronze layer (if you are familiar with medallion architecture from databricks).

Next we move from local computers to Google Cloud. To parallelize the pre-processing of our data, We used a service called Cloud Functions. It a serveless platform that package your code and spin up instances. You can scale up until 5000 thousands cloud functions in parallel and their processing times is 9 minutes maximum.

So we wrote the code necessary for the processing and packaged it to be shipped on Cloud Functions. Our approach was tool based. You will see that we developed a lot of CLI applications. The first one was leveraging Google Cloud python SDK to scrapes our data in the bucket, find the nights recorded both with the sunrise and polysomnography, anonymised the file, collect all metadata (night ID, bucket name, folder, filename, ect) and send all these information to subscription topic defined in Pub/Sub.

PubSub can then push the metadata payload as an HTTP request (encapsulated in our `SunriseFilePayload`) to trigger our cloud function.

A snippet of the CLI code can be found below.

```python
def send_nights_to_pubsub(
    input_dir: str,
    output_dir: str,
    topic: str,
    bucket_name: str = training_bucket,
):
    all_files = get_all_files(bucket_name, input_dir)
    all_nights = get_all_nights(all_files)
    publisher = pubsub_v1.PublisherClient()

    for night in all_nights:
        # removed for breivety but we were taking care of anonymisation
        # and checking that sunrise data and labeled data were available
        # for the same night. Once done we collected the hypnogram and
        # sunrise gyroscope data for the same patient.
        # collect the necessary files

        # Pydantic object with the metadata for one night
        sunrise_files_data = payloads.SunriseFilePayload(
            bucket=bucket_name,
            output_dir=output_dir,
            night=night,
            anomy=anonymised_file,
            hypno=hypno_file,
            accel=accel_file,
            gyros=gyros_file,
            patient=patient_file,
        )

        data = str(sunrise_files_data.model_dump()).encode("utf-8")

        # we send all metadata to a PubSub
        try:
            topic_path = publisher.topic_path(project="sensav2", topic=topic)
            future = publisher.publish(topic_path, data)
            future.result()
        except Exception as err_msg:
            logger.error(f"Failed to send to topic {topic} : {err_msg}")
            return "Status: failed"
        logger.info(f"Triggers sent for night {night}")
    logger.info(f"{len(all_nights) - missing}/{len(all_nights)} nights sent")
    return "Status: ok"
```

## Anatomy of a cloud function

With this approach, PubSub can trigger thousands of cloud functions in parallel. With each function receiveing the metadata for one file, we could process thousands of files in a matter of minutes. Two Cloud Functions are used at this stage: one to process specifically the sunrise data, and one to process the polysomnography data.

Here is how we organize our code in our repository:

```text
.
├── .github/
├── ai_vertex/
├── assets/
├── cloud_functions/        # here are our cloud functions
│   ├── labels/             # To process polysomnography nights
│   ├── resp_events/
│   ├── sleep_stages/
│   └── sunrise_files/       # To process sunrise nights
│       ├── .gcloudignore
│       ├── main.py
│       ├── requirements.txt
│       ├── triggers.py
│       ├── __init__.py
│       └── cf_cli.py
├── deploy/
├── scripts/
├── tests/
├── .flake8
├── .gitignore
├── .pre-commit-config.yaml
├── app.py
├── CI.Dockerfile
├── Makefile
├── README.md
├── requirements-dev.in
└── requirements-dev.txt
```

To deploy a cloud function you need to provide a `main.py` file with the code logic, a `requirements.txt` and if needed a `.gcloudignore` file to exclude any artifacts from the deployment in the cloud. The `triggers.py` contains the code shown just before. `cf_cli.py` was part our `app.py`, a CLI we developed using Typer. With this library you can decompose your cli application in several small pieces and unify the whole application in one file.

> Typer is pretty neat to use and very easy to learn the API.

Our CLI appications in `app.py` contains commands to:

1. Deploy the cloud functions, which basically calls this `bash` script:

   ```bash
   #! /bin/bash
   SOURCE_DIR="cloud_functions/labels"
   gcloud config set project sensav2;
   gcloud functions deploy write-psg-labels-domino \
   --region=europe-west1 \
   --runtime=python310 \
   --memory=2048 \
   --source=${SOURCE_DIR} \
   --entry-point=write_labels_for_domino \
   --trigger-topic=write-labels-domino \
   --timeout=540
   ```

2. Send the metadata to the right PubSub topic so that we trigger the right Cloud Function processing.

The pre-processed sunrise and polysomnography data were then saved in two distinct buckets.

![](/images/mlops-in-action-figure-04.png)

## Predictions and features extration

Once the sunrise data was pre-processed, it was ready to be used by our `sunalgo` application. The `sunalgo` was a monolithic service that accept the pre-processed sunrise data as input and generate the following outputs:

1. Model features
2. Model predictions
3. Diverse clinical scores for sleep quality

Our `sunalgo` code was packaged as a docker container and stored in our artifact repository on Google Cloud. The container could then be used and deployed on Cloud Run instances. Cloud Run is a serveless service that can scale up and parallelize processing (up to 1000 instances), and scales down (even to zero) when no instance is needed. Our dockerfile looks like this:

```Dockerfile
name: sunalgo-cicd-dev
on:
  push:
    branches: ["dev"]

env:
  GCP_PROJECT_ID: "sensav2"
  ARTIFACT_REGISTRY: europe-west1-docker.pkg.dev/sensav2/sunalgo
  TAG: ${{ github.ref_name }}-${{github.run_id}}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Execute tests
        id: tests
        uses: docker/build-push-action@v4
        with:
          tags: sunalgo_dev_tests
          context: .
          target: test-env
          secrets: |
            "GCP_KEY=${{ secrets.SENSAV2_KEY }}"
          file: build.Dockerfile

      - name: GCP authentication
        id: auth
        uses: google-github-actions/auth@v1
        with:
          token_format: 'access_token'
          workload_identity_provider: projects/350215846911/locations/global/workloadIdentityPools/sunrise-github

      - name: Login to Artifact Registry
        uses: docker/login-action@v2
        with:
          registry: europe-west1-docker.pkg.dev
          username: oauth2accesstoken
          password: ${{ steps.auth.outputs.access_token }}

      - name: Push worker
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            ${{ env.ARTIFACT_REGISTRY }}/sunalgo:${{ env.TAG }}_worker
          context: .
          target: runtime-worker
          secrets: |
            "GCP_KEY=${{ secrets.SENSAV2_KEY }}"
          file: build.Dockerfile

      - name: Deploy worker
        uses: "google-github-actions/deploy-cloudrun@v1"
        with:
          service: "worker-dev"
          image: "${{ env.ARTIFACT_REGISTRY }}/sunalgo:${{ env.TAG }}_worker"
          region: europe-west1
          flags: '--concurrency=1 --max-instances=1000 --cpu=8000m --memory=32Gi --port'
```

>There is a lot to unpack here and the idea is not to enter on all the details. But you may pay attention to the way we (i) authetnicate with workload identity to let our github actions being able to interact with Google Cloud services, (ii) tag our docker image based on the `Dev` or `Prod` environment and (iii) directly deploy on Cloud Run.

To not hit the limit of Cloud Run we decided to process our sunrise files by group of 800 tasks. To queue these tasks we used a service called Cloud Task. We could queue thousands tasks and cap it to a maximum of 800 active tasks. When a Cloud Run is finished it sends a message (HTTP payload) to the cloud task to receive the next task.

>Cloud task is the type of service you can "click-click" to configure. Very easy and simple. No rocket science.

## The trigger

With cloud task configured, our cloud run services up and running, we just needed to trigger the whole pipeline. And for that we used ... a CLI application!

```python
@click.option("--file_prefix", required=False)
@click.option(
    "--append_file_name_to_output_dir",
    default=True,
    help="Add the filename to output_dir name",
)
def send_tasks_to_cloud_queue(
    worker_url,
    input_bucket,
    input_dir,
    output_bucket,
    output_dir,
    max_files,
    file_prefix,
    append_file_name_to_output_dir,
):
    # Instantiate client
    client = tasks_v2.CloudTasksClient()
    parent = client.queue_path(
        project="sensav2", location="europe-west1", queue="poller-queue"
    )
    # task dict
    task = {
        "http_request": {
            "http_method": tasks_v2.HttpMethod.POST,
            "url": f"{worker_url}/score",
        }
    }
    # change the timeout limit for cloud task, maximum is 30 minutes
    # https://github.com/googleapis/python-tasks/issues/93#issuecomment-827752337
    timeout = duration_pb2.Duration()
    timeout.FromSeconds(60 * 30)
    task["dispatch_deadline"] = timeout
    # base payload
    payload_dict: dict[str, str] = {
        "input_bucket": input_bucket,
        "output_bucket": output_bucket,
    }
    # iterate over files and send the task
    target_files = files_names[:max_files] if max_files else files_names
    for file_name in target_files:
        logger.info(f"Processing file : {file_name}")
        payload_dict["file_name"] = file_name
        if append_file_name_to_output_dir:
            payload_dict["output_dir"] = (
                f"{output_dir}/{file_name}" if output_dir else file_name
            )
        bytes_payload = json.dumps(payload_dict).encode()
        date_str = datetime.now().strftime("%m-%d-%Y-%H-%M-%S-%f")
        task["name"] = client.task_path(
            project="sensav2",
            location="europe-west1",
            queue="poller-queue",
            task=f"{date_str}____{file_name.split('.')[0].replace(':', '_')}",
        )
        task["http_request"]["body"] = bytes_payload
        response = client.create_task(request={"parent": parent, "task": task})
        logger.info(f"Create task {response.name}")
    logger.info(f"Cloud Task : {len(target_files)} tasks sent to the queue")


if __name__ == "__main__":
    cli()
```

Below is a diagram of the our whole pipeline.

![](/images/mlops-in-action-figure-05.png)

## Wrap up

By running just a couple of commands we could process more than 2000 files in some hours instead of days. The gain in time and efficiency was huge. At each step we enforce good DevOps practices with continous integration and development with containarization and Github Actions.

At this step we had another huge challenge to tackle: the training of the model. The generated predictions will be used with the labelled data and be ingested by our model for training. In the next article I will describe how we did that with Vertex.AI.
