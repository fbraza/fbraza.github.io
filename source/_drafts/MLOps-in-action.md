# ML Engineering in action

When I landed my first tech jobs, I quickly realize that no amount of side projects, certifications, or online courses fully prepared me for professional reality. Indeed every company operates in its own unique ecosystem of tools, constraints, and engineering practices. For example I have worked with Banking systems running Hadoop clusters deployed with Cloudera. Startups liing entirely in the cloud. Teams swearing by Python while others champion R or Scala. Even when companies use the same language, their engineering practices can be worlds apart.

One team might have sophisticated CI/CD pipelines on GitHub, while another struggles with basic version control on Bitbucket. Sometimes the constraints get even more unexpected but still interesting. At my work with AXA Belgium, we could not use Docker containers for security reasons, period. This meant deploying an LLM interface required other solutions like AWS Lambda instead of the straightforward Fargate approach we would have preferred. Not ideal, but it worked.

So before you read this article, my advice will be to not fall in love with specific technologies or try to copy what others do. Master the fundamentals instead. Whether it is git (regardless of the platform), containerization principles, or data pipeline concepts (whether cloud or on-premise). Because these core ideas translate across any tech stack.

This brings me to why I'm writing this article. I want to share how we tackled some data and machine learning challenges at Sunrise. Our deployment strategies, our attempt at MLOps best practices, and the solutions we developed for our specific constraints. This is not a blueprint you should copy. It is simply the story I participated to build: how we approached technical challenges with their particular stack, requirements, and trade-offs.

## What is sunrise by the way

Sleep apnea affects millions but remains poorly diagnosed—a condition that contributes to cardiac and metabolic diseases. Traditional diagnosis typically requires expensive sleep labs and overnight stays, creating barriers to access for many patients. Moreover, polysomnography required the patient to suit-up with a lot of instrument. Look by yourself.

![](/images/mlops-in-action-figure-01.png)

>from a marketing perspective this illustration is actually very compelling

To address this challenge, Sunrise built their company around sleep medicine and AI, focusing on making sleep monitoring more accessible. Their approach is based on research showing that mandibular muscle contractions (MMC) can effectively monitor sleep quality and detect apneas. Specifically, they developed a small device that attaches to the chin and records these muscle contractions through gyroscopic data. This raw data is then processed by their algorithms to generate insights about sleep patterns and potential issues. Sunrise combines their monitoring device with a mobile app that enables at-home sleep assessment, where users receive reports detailing their sleep phases and clinical metrics after monitoring—information that was traditionally only available through lab-based studies. The result is a more accessible approach to sleep monitoring that users can implement from home. The model they used is a simple XGBoost that classify each 30-seconds period of sleep into: `Awake`, `Deep sleep`, `Light sleep`, `REM sleep`.

![](/images/mlops-in-action-figure-02.png)

## The starting point

The first infrastructure looked like this

![](/images/mlops-in-action-figure-03.png)

Briefly, we have two distinct “infrastructure” here.

1. The predictions service which was deployed on the google cloud with on App Engines. One worker service that was processing the recorded nights to generate predictions and clinical metrics. And one poller service that was constantly ping by a cron jobs to trigger the processing of new nights uploaded in our data lake (Google Storage).

2. The training infrastructure consisted of one computer with a graphic card. Moreover, labeled data that were a mix of nights recorded by polysomnography and the sunrise devices were stored, kind of “everywhere”. On the cloud, on hard drives and on DropBox.

## Speed of iteration

In software engineering and more broadly in active process, speed of iteration is critical. I am quite fan of the Boyd law of iteration:

>Boyd decided that the primary determinant to winning dogfights was not observing, orienting, planning, or acting better. The primary determinant to winning dogfights was observing, orienting, planning, and acting faster. In other words, how quickly one could iterate.

To give you an example, to prepare the training data, the data scientist was running locally its code and was processing files one by one. We are talking about 2000 nights being individually processed to be cleant, extract clinical metadata and prepare the data in the right format for the training. Process was lasting between 30 seconds to maximum one and a half minute depending on the recorded night length. In other words, this could take between one to three days. And after there was the training who took a couple more day. So it took them a bit more than a week of work to process, train evaluate and re-iterate.

## Improving the process: speed up data processing

The first task was to improve the pipeline used to prepare our data for features extraction and predictions. From local processing we moved to Google Cloud. We used a service called Cloud Functions, a serveless platofrm that allows us to parallelize the processing. For that we centraize in our data lake all the data from sunrise device and polysomnography spread over different buckets, dropox folder and hard drive in one bucket: the training bucket. This was our single source of truth, our Bronze layer (if you are familiar with medallion architecture from databricks).

IMAGE

Our approach was quite simple. We decided to first write a CLI that scrapes our data in the bucket, find the nights recorded both with the sunrise and polysomnography, anonymised the file, collect all metadata (night ID, bucket name, folder, filename, ect) and send all these information to right topic defined in Pub/Sub. This service was then distributer the metadata information (encapsulated in our `SunriseFilePayload`) to a cloud function. A snippet of the code can be found below.

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

```text
suntraining
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

To deploy a cloud function you need to provide a `main.py` file with the code logic, a `requirements.txt` and if needed a .`gcloudignore` to exclude any file from the deployment in the cloud. The `triggers.py` contains the code shown just before. `cf_cli.py` was part our `app.py`, a CLI we developed using Typer. With this library yo can decompose your cli application in several small pieces and unify the whole application in one file. Pretty neat to use and very easy to learn the API. Our CLI appications in `app.py` contains commands to:

1. Deploy the cloud functions, which basically calls a `bash` script

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

2. Send the metadata to the right PubSub topic so that we trigger the right Cloud Functions processing.

Here the processed sunrise polysomnography (our labels) were saved in two distinct buckets.

IMAGE

## Features extractions and predictions

Once the sunrise data was processed, it was ready to be used by our `sunalgo` application. The `sunalgo` was a monolithic service that:

1. process and clean the raw data
2. Process the signal
3. extract the required features
4. run predictions of sleep stages, apneas, respiratory distress
5. calculate several clinical metrics linked to sleep quality
