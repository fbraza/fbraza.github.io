---
title: Service to service communication in Google Cloud
date: "2023-11-23"
tags:
    - Google Cloud
---

At my work we process our data using Google Cloud services. For different use cases, we mainly use Cloud functions to process files and Cloud Run services to generate features, sleep scores and predictions. We use PubSub to trigger our cloud functions and, Cloud Task or Scheduler to trigger our respective Cloud Run services. All of them use HTTP requests to communicate with each others. By default, communication with Cloud run services requires authentication. Here, I am going to present how to set up authenticated communications with google Cloud Run services.

## Our data flow

In our data infrastructure we use a Scheduler that pings one of our Cloud Run service that we call the `poller`. The role of the `poller` is, once triggered, to scan a Google Cloud bucket where are stored our new data. If new data is found, it sends the necessary metadata to another Cloud Run service, called the `worker`. The `worker` will then process the data. If the processing succeeds or fails the `worker` sends back an HTTP message (HTTP code `200` or `500`) that prompts the `poller` to move the processed file to the required bucket.

## Scheduler to Cloud Run

Make Cloud Scheduler and Cloud run communicate with authentication is fairly easy. First you need to create a service account for your scheduler that will have the `roles/run.invoker` role. When creating your scheduler, configure the `Configure the execution` as showed below.

![](/images/2023-02-13-service-to-service-communication-gcp-figure-01.png)

Note the Google-signed OpenID Connect (OIDC) token passed in the header of the request. It is a token signed by google that allows the scheduler to communicate with other services in a authenticated way. Two other services can automatically include an ID token, Cloud Task and Pub/Sub.

## Cloud Run to Cloud Run

The approach is a bit different for communication between two cloud run services. First, attach to the sender a service account with the `roles/run.invoker` role. Next, you need to write a little bit of code in your request. Our poller, list files and send their path to the worker so that it can upload the data and process it. In the request code we need to add some lines to authenticate:

```python
def whatever_function_used_for_request():
    # Code for authentication
    http = requests.Session()
    auth_req = google.auth.transport.requests.Request()
    audience = os.environ["worker_url"]
    id_token = google.oauth2.id_token.fetch_id_token(auth_req, audience)
    headers = {"Authorization": f"Bearer {id_token}"}
    response = http.post(url, json=json_dict, headers=headers)
```

The interesting line of code is the following:

```python
id_token = google.oauth2.id_token.fetch_id_token(auth_req, audiance)
```

The function `fetch_id_token` acquires ID token from your environment in the following order:

1. If the environment variable `GOOGLE_APPLICATION_CREDENTIALS` is set to the path of a valid service account `JSON` file, then ID token is acquired using this service account credentials.
2. If the application is running in `Compute Engine`, `App Engine` or `Cloud Run`, then the ID token is obtained from the metadata server.
3. If metadata server doesn't exist and no valid service account credentials are found, it raise the `google.auth.exceptions DefaultCredentialsError` error.

## Wrap-up

And that's it. Keeping authenticated communication is very important to secure your infrastructure. Making cloud services communicate between them is fairly easy but not always clearly documented online, even in the official documentation that is sometimes too verbose. I decided to digest what i learnt in this process and provide the reader this short guide.
