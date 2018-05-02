## Part 2

The demo app used in this part is [kubernauts/dok-example-us](https://github.com/kubernauts/dok-example-us). It is a simple microservices app, simulating the stock market domain. As a first step, check out the repo to learn more about its architecture.

### Preparation

As a preparation, clone the stock app repo and create a project like so:

```bash
$ git clone https://github.com/kubernauts/dok-example-us.git && cd dok-example-us
$ oc new-project dok
```

### Stock generator

First, we deploy the stock generator microservice by directly creating an app based off of a container image. Do this:

```bash
$ oc new-app --docker-image=quay.io/mhausenblas/stock-gen:0.3 \
           --name=stock-gen \
           --env=DOK_STOCKGEN_PORT=9999 \
           --env=DOK_STOCKGEN_CRASHEPOCH=100
```

Note that we're also setting two environment variables in above command: 1. `DOK_STOCKGEN_PORT` which tells the microservice on which port it should serve the stock data, and 2. `DOK_STOCKGEN_CRASHEPOCH` which defines how frequently a stock market crash should be introduced. You can experiment with the latter if you like.

Next, we expose the deployment config we created in the previous step, resulting in a service with the same name (`stock-gen`) to be created:

```bash
$ oc expose dc/stock-gen --port=9999
```

### Stock consumer

Now that we have the stock generator up and running, we can launch the stock consumer microservice. First, we create an app by using the [source-to-image build strategy](https://docs.openshift.com/container-platform/3.9/architecture/core_concepts/builds_and_image_streams.html#source-build):

```bash
$ oc new-app https://github.com/kubernauts/dok-example-us \
           --strategy=source \
           --name=stock-con \
           --context-dir=stock-con \
           --env=DOK_STOCKGEN_HOSTNAME=stock-gen \
           --env=DOK_STOCKGEN_PORT=9999
```

Note the following things, above:

- The `--context-dir` parameter tells the `oc` tool which directory to use for the source code (in our case it's the `stock-con` microservice written in Node.js)
- The environment variables `DOK_STOCKGEN_HOSTNAME` and `DOK_STOCKGEN_PORT` tell the `stock-con` microservice how it can connect to the `stock-gen` microservice.
- The `oc new-app` command in the form we used it here created both a deployment config and a service for us, however, we need to patch the service since we're using a non-standard port.

Now that the pod hosting the stock generator app is running, we need to patch the service to make it work with our non-standard `9898` port:

```bash
$ oc patch svc stock-con \
     --type=merge \
     --patch='{"spec": {"ports": [ { "name": "http", "port": 9898, "protocol": "TCP", "targetPort": 9898 } ] } }'
```

Since we want to make our stock consumer service accessible from outside the OpenShift cluster, we now expose it, effectively creating a [route](https://docs.openshift.com/container-platform/3.9/architecture/networking/routes.html) for it using the following command:

```bash
$ oc expose svc stock-con --port=9898
```

You should now be able to access the stock consumer service from your browser or via `curl` using a URL of the form `http://stock-con-dok.$BASEURL/average/NYSE:RHT` with `$BASEURL` being the pubic-facing load-balancer of your OpenShift cluster.

