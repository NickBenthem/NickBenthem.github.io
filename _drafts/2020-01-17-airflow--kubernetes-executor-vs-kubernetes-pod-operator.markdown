---
layout: post
title: "KubernetesExecutor or KubernetesPodOperator?"
date: "2020-01-17 14:12:10 -0800"
tags: Kubernetes R DataScience Docker Airflow
---
# **TL;DR**
KubernetesExecutor spins up a new Kubernetes pod with an Airflow instance that runs your operator. KubernetesPodOperator spins up a new Docker image for each task you want to call.

# Background

We've been exploring standardizing our ETL orchestration tools at [Everlane](https://www.everlane.com/) and we landed on using Airflow for the data team; however, some of our ETL and data science jobs are memory and compute intensive - we wanted something that could both scale up to handle our needs as well as scale down when not needed <sup id="a1">[1](#f1)</sup> (AKA -  scale to 0). We also have some workflows that require very specific R and python libraries to be installed that can interact poorly. This can cause headaches for data analysts when they typically use a small set of libraries and want to perform a nightly job.

Both the local executor and the Celery executor have troubles separating dependencies as all libraries must be present on the worker machines <sup id="a2">[2](#f2)</sup>. The Celery worker also requires monitoring of job workers and the overhead that comes with. With these in mind, we decided that going on Airflow with Kubernetes using Docker images could solve quite a few of our problems leaving us more time to do analytics and data science.

With Docker and Kubernetes, we're able to spin up containers with complicated R libraries pre-installed and specify the exact amount of compute and memory to allocate. Airflow presents two main options to implement Kubernetes right now:

1. Use Airflow with a Local (or Celery<sup id="a3">[3](#f3)</sup>) and implement tasks using KubernetesPodOperator, or

1. Use Airflow with the Kubernetes Executor and use Airflow Operators.

We went with #2 and let me explain why.

## How the KubernetesPodOperator works

To get to knowing which method of Kubernetes to use - let's go through how the KubernetesPodOperator works. To keep things simple, we're going to go ahead and assume a Local Executor; though, the idea is very similar with a Celery Executor.

Airflow consists of three components:

1. A Webserver - This handles us being able to interact with Kubernetes
1. A Scheduler - To update the SQL database and monitor workers.
1. A SQL Database - to track success/failure and to write to.

Whenever you call up a KubernetesPodOperator, Airflow will go ahead and

   1. Create a Pod on the Kubernetes cluster with your Kubernetes cluster
   2. Load the Docker image you want to run on the Kubernetes pod
   3. Execute the `entrypoint` of the Docker image (a shell script saying what to do when initiated)
   4. Return the result as a status to the Airflow SQL database.
   5. Shutdown the Kubernetes Pod.

![KubernetesPodOperator](/assets/img/airflow--kubernetes-executor-vs-kubernetes-pod-operator/kubernetespodoperator.png)

This is generally great - we have isolated code from our webserver, reducing the webserver as a potential bottleneck.

 However, there is one main drawback to this: You're not able to use Airflow Operators. If you have jobs that behave similarly to already written Airflow operators, you will have to rewrite all jobs to run independently in a docker container. These jobs must be self contained and you need to have a good amount of Docker and Kubernetes knowledge to write them. If you're porting over SQL / Python / R code from a different system as we at Everlane have been doing, you now have to write custom implementations of these operators.


### How the KubernetesExecutor works

With the KubernetesExecutor, a similar operation to the KubernetesPodOperator happens, except each time you run a task, you're creating a Kubernetes Pod with a copy of Airflow inside. You now have a new Airflow Scheduler and Webserver that is created **for each operator** you run. The SQL backend is still our original SQL instance, but we've transferred the worry of tracking a job off to the Kubernetes child worker. The main Kubernetes scheduler schedules the task in SQL, at which point the Airflow goes and create a Kubernetes pod that will take our job and run it. The "master" scheduler no longer needs to check if to see if the pod is running / scheduled / cancelled / failed. The child Kubernetes pod is now in charge of talking to the SQL backend.

The steps that the KubernetesExecutor uses then is


   1. Create a Pod on the Kubernetes cluster with your Kubernetes cluster
   2. Load the Docker image *that must contain Airflow* you want to run on the Kubernetes pod
   3. Run the Operator on the Webserver of the child Kubernetes Airflow
   4. The child Kubernetes Airflow then writes to the original SQL database success/failure
   5. Shutdown the Kubernetes Pod.


![KubernetesExecutor](/assets/img/airflow--kubernetes-executor-vs-kubernetes-pod-operator/kubernetesexecutor.png)


We get a few advantages from this:

1. We can now use all built-in Airflow Operators. Each operator runs on the Webserver of the pod. This can save a large amount of time for developers if your workflows are often similar, and you get easy access to [new contributor operators](https://github.com/apache/airflow/tree/master/airflow/contrib/operators).

1. We're still able to execute our code and specify Kubernetes options such as `image`, `cpu`, and `memory` using `executor_config`

1. We're able to maintain our credentials in a centralized spot in Airflow without needing to pass Kubernetes secrets or needing to download/create credentialing libraries into our docker images.


For instance, to connect to our data warehouse we can simply use a predefined Operator -

```
class EverlaneSQLOperator(PostgresOperator):
    PostgresOperator.ui_color = '#007aa5'
    @apply_defaults
    def __init__(
            self,
            postgres_conn_id="ReportingOne", # Main reporting server
            *args, **kwargs):

        kwargs.setdefault('executor_config', {"KubernetesExecutor": {"image": "everlane/data:latest",
                                                                     "image_pull_policy": "Always"
                                                                     }})

        super(EverlaneSQLOperator, self).__init__(postgres_conn_id = postgres_conn_id,
                                                  *args, **kwargs)

```

Note here that `everlane/data:latest` is a custom docker image with prebuilt libraries installed.


If you use the Kubernetes Executor with a KubernetesPodOperator, you will infact launch two pods inside Kubernetes for each task - one for your task pod's Airflow instance, and one for your task that you need to execute.
![KubernetesPodWExecutor](/assets/img/airflow--kubernetes-executor-vs-kubernetes-pod-operator/kubernetespodwexecutor.png)
[Insert Photo Here]


# Why we chose KubernetesExecutor:

 Being able to use Airflow operators is a large part of why we wanted to use Airflow and having access to the included operators that contributors is a large win for us in terms of development time. Rather than implement a custom solution to handle new types of operations (say, s3 to SFTP), we can just extend the existing functions and save development time. By using built-in Airflow operators rather than building Docker images, we save the time complexity of developing custom Operators. We still retain the ability to run self contained docker applications given to us by other teams such as engineering by using the KubernetesPodOperator, we just pay a small overhead for running an extra KubernetesPod.

 At the end of the day, we actively maintain relatively few Docker images and gain little benefit in customizing each job. Building custom implementations for existing functions is more expensive for us than just extending existing Operators. Our R scripts generally use the same libraries, take a long time to compile,  and can break often due to dependency conflicts.  Our default images install many of our default R and Python libraries once and provide that to all images. We can still be flexible by allowing custom Docker images for complex dependency requirements, though any image we use with native operators **must** contain the airflow package.


## Addendum -

<b id="f1">1</b> https://www.astronomer.io/guides/airflow-executors-explained/ - If you're looking for an overview of the different executors and aren't decided on Kubernetes as your compute/container platform, Astronomer has a great layout of what each executor does, as well as some common terms. We use Astronomer to handle all the non-Airflow complexities of Airflow (DevOps, deployments, authentication, etc.)  [↩](#a1)

<b id="f2">2</b> https://medium.com/bluecore-engineering/were-all-using-airflow-wrong-and-how-to-fix-it-a56f14cb0753 - Jessica Laughlin's excellent summary of the KubernetesPodOperator. This helped me get started with how Kubernetes works inside of Airflow. [↩](#a2)


<b id="f3">3</b> https://www.sicara.ai/blog/2019-04-08-apache-airflow-celery-workers - More on how the Celery executor behaves. A celery executor is probably unnecessary if you solely use the KubernetesPodOperator unless you have massive numbers of jobs and require the ability to send off the task to be picked up by the celery worker to then spin up the KubernetesPod [↩](#a3)

 <b id="f4">4</b>  We actually use Operators that extend the Operator we are trying to use - I.e., ExtendedPostgresOperator to allow us to integrate and kick off certain job tracking utilities as well as help codify our team's decisions and provide default options (`image`) to the analysts without modifying the original Airflow code. [↩](#a4)
