# Tekton Triggers + Knative Eventing = <3

This repository contains an example CI system that links `Knative Brokers` to `Tekton Triggers` to demonstrate a simple but powerful pattern that these cool projects enable when combined.

The demo provides the tools to launch a self-contained k8s demo environment locally using [kind](https://github.com/kubernetes-sigs/kind#installation-and-usage). Follow, we will interact with the cluster and examine this pattern further by following a workflow using [ko](https://github.com/google/ko#installation) and [kail](https://github.com/boz/kail#installing) to iterate on some changes and view logs.

## Purpose

Markdown and diagrams are great, but one of my favorite ways to learn is to experiment with a running system. This project provides a self-contained demo system which makes use of the Knative Brokers and Triggers, Tekton Triggers and Pipelines projects. Once booted, we can check out:

- Sending new CloudEvents into our new system
- how Brokers can shuttle our events around our system
- use Triggers can be used to subscribe to events
- create a TriggerFilter that can be used to ensure delivery of only a subset of a Brokers messages
- examine how Tekton Triggers Eventlistener recieves and processes messages
- watch our CI resources get dynamically created on event reception
- Parse the results of our successful CI run, examine Tekton PipelineRun status

## Pre-Reqs

To boot the experiment, you will need to install the following software.

- [kind](https://github.com/kubernetes-sigs/kind#installation-and-usage)
- [ko](https://github.com/google/ko#installation)
- [kail](https://github.com/boz/kail#installing)

The demo system is designed to run locally against the provided Kind cluster, but the same process should work against any valid Kubernetes cluster > `v1.15.0`. If you have your own cluster, just skip `./bin/up` and go directly to `./bin/apply`.

## Create the Demo Environment

To get started, we will bring up our own demo environment. You can also use any valid Kubernetes cluster, but this demo was built around Kind.

First lets run `./bin/up` to create our demo cluster. This will boot a new Kind cluster named `tekton-dream` and prep the cluster environment.

## Configure the Demo system

Next lets install the required resources. For this demo we will be using:

- [Knative Brokers and Triggers](https://github.com/knative/docs/blob/master/docs/eventing/broker-trigger.md)
- [CloudEvents](https://github.com/cloudevents/spec/blob/master/spec.md)
- [Tekton Triggers](https://github.com/tektoncd/triggers/blob/master/docs/README.md)
- [Tekton Pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md)
- [Gloo](https://docs.solo.io/gloo/latest/installation/knative)

To install them, lets run `./bin/apply`. This will handle installing and updating all of these projects.

Finally, lets set our kubeconfig so we can access the new cluster:

`export KUBECONFIG="$(kind get kubeconfig-path --name="$CLUSTER_NAME")"`

## Review

Now, we have a local Kubernetes cluster with Knative and Tekton resources installed. We also have a new namespace `tekton-dream` which contains:

- A `Knative Broker`, which we will send our Event into and will distribute them across all subscribe services.
- `Tekton Triggers`, which exposes a Listener. This is the Addressable for our eventing source and will respond to events by spawning PipelineRuns.
- A `Tekton Pipeline`, which clones, tests a source repository. Then, it builds and pushes a docker image for the code.
- A `Knative Trigger`, which defines that our Tekton Trigger is where events of type `com.github.push` will go, if they match the correct repository (in this case, `iancoffey/ulmaceae`).

We are ready to test the system!

## Send demo event

Lets send our demo event to our Broker and see the whole thing work!

`ko apply -f manifests/sendevent.yaml | kail -n tekton-dream`

This will output a ton of data at the system roars to life. But what just happened?

## Its alive! What is happening?

When we applied the YAML above, we created a `ContainerSource` event source, which will send our example CloudEvent into our Knative Broker!

- We created a `CloudEvent` with the ContainerSource event source we applied above
- The new CloudEvent was sent into our Broker `ci-builds`
- The Broker then determined that our `Tekton Triggers EventListener` needed to have a copy of the event sent to it. This is because we defined a `Knative Trigger` for this service.
- Our EventListener is the CloudEvent by the Broker there since the criteria matched
- The listener receives the event and parses the Event data
- The `TriggerTemplate` templates are all populated with Event Data and created
- The `PipelineResources` are created finally our `PipelineRun` beginds
- The `Pipeline` downloads the source repository defined in the CloudEvent
- The two Task are run. They combine to test, build and push the source code we defined in our Event
- Finally, the last Task starts a new Deployment using our new image and it boots on our Cluster!

## Diagram

![Broker Tekton Diagram](/images/diagram.png)

## Iterating

The `./bin/clear` script provides a means to reset the environment back to scratch, erasing all of the `PipelineRuns` and `PipelineResources` in the process.

To completely erase the development environment when you are done, just delete the Kind cluster. Dont worry - its easy to bring it back up.

To delete, just run `kind delete cluster --name tekton-dream`.
