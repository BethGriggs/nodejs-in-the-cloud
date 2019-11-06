# Node.js in the Cloud

Welcome :wave: to the Node.js in the Cloud workshop at NodeConfEU!

The first part of this workshop will teach you how to extend a simple Express.js application to leverage cloud capabilities.

The second part of the workshop will demonstrate tooling to help you develop all of your cloud-native applications from a consistent base.

## Part 1: Extending an Express.js application to leverage Cloud Capabilties 

### Building a Cloud Ready Express.js Application

This will show you how to take a Node.js application and make it "cloud ready": adding support for Cloud Native Computing Foundation (CNCF) technologies using the package and templates provided by the [CloudNativeJS](https://www.cloudnativejs.io/) project.

In this self-paced tutorial, you will:

* Create an Express.js application
* Add Health Checks and Metrics to your application
* Build your application with Docker
* Package your application with Helm
* Deploy your application to Kubernetes
* Monitor your application using Prometheus

The application you'll use is a simple Express.js app built using the Express Generator. You'll learn about Health Checks, Metrics, Docker, Kubernetes, Prometheus and Grafana. At the end you'll have a fully functioning application running as a cluster in Kubernetes, with production monitoring.

### Prerequisites

Before getting started, make sure you have the following prerequisites installed on your system.

1. [Node.js 8 or later](https://nodejs.org/en/download/) or using [nvm](https://github.com/nvm-sh/nvm#installation-and-update)
2. Your IDE of choice
3. Docker and Kubernetes
    - ***On Mac or Windows***: [Docker for Desktop](https://www.docker.com/products/docker-desktop)
    - ***On Linux***: Docker for Desktop is not available, alternatives are:
      - [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
      - [microk8s](https://microk8s.io/#quick-start)

### Setting up

How to start Kubernetes will depend on how you intend to run it. Also, note
that the Prometheus Helm chart is
[not compatible](https://github.com/helm/charts/pull/17268) with
Kubernetes 1.16, so make sure to install 1.14, see below.

#### Starting Kubernetes

#### `Docker for Desktop`

Ensure you have installed Docker for Desktop on your Mac and enabled Kubernetes within the application. To do so:

1. Select the Docker icon in the Menu Bar
2. Click Preferences/Settings > Kubernetes Tab > Enable Kubernetes.

It will take a few moments to install and start up. If you already use Kubernetes, ensure that you are configured to use the `docker-for-desktop` cluster. To do so:

1. Select the Docker icon in the Menu Bar
2. Click Kubernetes and select the `docker-for-desktop` context

#### `microk8s`

<details>

```sh
snap install --channel 1.14/stable microk8s --classic
sudo microk8s.start
snap alias microk8s.kubectl kubectl
export PATH=/snap/bin:$PATH
microk8s.config >~/.kube/config
microk8s.enable dns registry
```

You may be prompted to add your userid to the 'microk8s' group to avoid having to use `sudo` for all the commands.

</details>

#### `minikube`

<details>

```sh
minikube start --kubernetes-version=1.14.7
eval $(minikube docker-env)
```
</details>

#### Installing Helm

Helm is a package manager for Kubernetes. By installing a Helm "chart" into your Kubernetes cluster you can quickly run all kinds of different applications. You can install Helm using one of the options below:

**Using a Package Manager:**

* macOS with Homebrew: `brew install kubernetes-helm`
* Linux with Snap: `sudo snap install helm --classic`
* Windows with Chocolatey: `choco install kubernetes-helm`

**Using a Script:**

```sh
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Once you have Helm ready, you can initialize the local CLI and also install Tiller into your configured Kubernetes cluster in one step:

```sh
helm init
```

This will install Tiller into the Kubernetes cluster, which controls the deployment of your Helm charts.

Now that Helm is installed, you should also configure access to the "stable" Helm repository as follows:

```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

This makes it easy for you to install a number of applications and services into your Kubernetes cluster. You'll use this to install Prometheus and Grafana later in the workshop.

### 1. Create your Express.js Application

Use the following steps to create your Express.js application:

1. Create a directory to host your project

   ```sh
   mkdir nodeserver
   cd nodeserver
   ```

2. Globally install and run the Express generator to build your skeleton application:

   ```sh
   npx express-generator --view=ejs
   npm version 1.0.0
   ```
   
   Note: `express-generator` still uses `var`.

This has built a simple Express.js application called `nodeserver`, after the name of the directory you are in.

3. Install your applications dependencies and start your application:

    ```sh
    npm install
    npm start
    ```

Your application should now be visible at [http://localhost:3000](http://localhost:3000).

### 2. Add Health Checks to your Application

Kubernetes, and a number of other cloud deployment technologies, provide "Health Checking" as a system that allows the cloud deployment technology to monitor the deployed application and to take action should the application fail or report itself as "unhealthy".

The simplest form of Health Check is process level health checking, where Kubernetes checks to see if the application process still exists and restarts the container (and therefore the application process) if it is not. This provides a basic restart capability but does not handle scenarios where the application exists but is un-responsive, or where it would be desirable to restart the application for other reasons.

The next level of Health Check is HTTP based, where the application exposes a "livenessProbe" URL endpoint that Kubernetes can make requests of in order to determine whether the application is running and responsive. Additionally, the request can be used to drive self-checking capabilities in the application.

The `@cloudnative/health-connect` package provides a Connect Middleware that makes it easy to add a default health check endpoint and provides a Promise based API for adding self-checking capabilities. The module is written in TypeScript. In this workshop but here we will be using `var` for consistency with the `express-generator`.

Add a Health Check endpoint to your Express.js application using the following steps:

1. Add the `@cloudnative/health-connect` dependency to your project:
   
   ```sh
   npm install --save @cloudnative/health-connect
   ```

2. Set up a HealthChecker in `app.js`:

   ```js
   var health = require('@cloudnative/health-connect');
   var healthcheck = new health.HealthChecker();
   ```
   
3. Register a Liveness endpoint in `app.js`:

   ```js
   app.use('/health', health.LivenessEndpoint(healthcheck));
   ```
   
This has added a `/health` endpoint to your application. As no liveness checks are registered, this will return as status code of 200 OK and a JSON payload of `{"status":"UP","checks":[]}`.

Check that your `livenessProbe` Health Check endpoint is running:

1. Start your application:

   ```sh
   npm start
   ```

2. Visit the `health` endpoint [http://localhost:3000/health](http://localhost:3000/health).

For information on how to register health/liveness checks, and additional support for start-up, readiness and shutdown checks, see the [Cloud Health documentation](https://github.com/CloudNativeJS/cloud-health/blob/master/README.md).

### 3. Add Metrics to your Application

For any application deployed to a cloud, it is important that the application is "observable": that you have sufficient information about an application and its dependencies such that it is possible to discover, understand and diagnose the state of the application. One important aspect of application observability is metrics-based monitoring data for the application.

The CNCF recommended metrics system is [Prometheus](http://prometheus.io), which works by collecting metrics data by making requests of a URL endpoint provided by the application. Prometheus is widely supported inside Kubernetes, meaning that Prometheus also collects data from Kubernetes itself, and application data provided to Prometheus can also be used to automatically scale your application.

The `appmetrics-prometheus` package provides an easy to use library that auto-instruments your application to collect metrics and exposes them on a `/metrics` endpoint for consumption by Prometheus.

Add a `/metrics` Prometheus endpoint to your Express.js application using the following steps:

1. Add the `appmetrics-prometheus` dependency to your project
   
   ```sh
   npm install --save appmetrics-prometheus
   ```

2. 	Require `appmetrics-prometheus` and attach it to your Express server:

   ```js
   var prometheus = require('appmetrics-prometheus').attach();
   ```
   
This has added a `/metrics` endpoint to your application. This automatically starts collecting CPU, Memory and HTTP responsiveness data from your application and exposes it in a format that Prometheus understands.

Check that your metrics endpoint is running:

1. Start your application:

   ```sh
   npm start
   ```

2. Visit the `metrics` endpoint [http://localhost:3000/metrics](http://localhost:3000/metrics).

For information on how to configure the appmetrics-prometheus library see the [appmetrics-prometheus documentation](https://github.com/CloudNativeJS/appmetrics-prometheus).

You can install a local Prometheus server to graph and visualize the data, and additionally to set up alerts. For this workshop you'll use Prometheus once you've deployed your application to Kubernetes.

### 4. Building your Application with Docker

Before you can deploy your application to Kubernetes, you first need to build your application into a Docker container and produce a Docker image. This packages your application along with all of its dependencies in a ready to run format.

CloudNativeJS provides a "[Docker](https://github.com/CloudNativeJS/docker)" project that provides a number of best-practice Dockerfile templates that can be used to build your Docker container and produce your image.

For this workshop, you'll use the `Dockerfile-run` template, which builds a production-ready Docker image for your application.

Build a production Docker image for your Express.js application using the following steps:

1. Copy the `Dockerfile-run` template into the root of your project:

   ```sh
   wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/Dockerfile-run
   ```
   
2. Copy the `.dockerignore` file into the root of your project:

   ```sh
   wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/.dockerignore
   ```

3. Build the Docker run image for your application:

   ```sh
   docker build --tag nodeserver-run:1.0.0 --file Dockerfile-run .
   ```
   
You have now built a Docker image for your application called `nodeserver-run` with a version of `1.0.0`. Use the following to run your application inside the Docker container:

  ```sh
  docker run --interactive --publish 3000:3000 --tty nodeserver-run:1.0.0
  ```

This runs your Docker image in a Docker container, mapping port 3000 from the container to port 3000 on your laptop so that you can access the application.

<details>
<summary>minikube only</summary>

Docker runs in the minikube VM, so an additional step is
required to expose the application to localhost:
```sh
kubectl port-forward service/nodeserver-service 3000
```
</details>

Visit your applications endpoints to check that it is running successfully:

* Homepage: [http://localhost:3000/](http://localhost:3000/)
* Health: [http://localhost:3000/health](http://localhost:3000/health)
* Metrics: [http://localhost:3000/metrics](http://localhost:3000/metrics)

## 5. Packaging your Application with Helm

In order to deploy your Docker image to Kubernetes you need to supply Kubernetes with configuration on how you need your application to be run, including which Docker image to use, how many replicas (instances) to deploy and much memory and CPU to provide to each.

Helm charts provide an easy way to package your application with this information. 

CloudNativeJS provides a "[Helm](https://github.com/CloudNativeJS/helm)" project that provides a template best-practice Helm chart template that can be used to package your application for Kubernetes.

Add a Helm chart for your Express.js application using the following steps:

1. Download the template Helm chart

   ```sh
   wget https://github.com/CloudNativeJS/helm/archive/master.zip
   ```

2. Unzip the downloaded template chart

   ```sh
   unzip master.zip
   ```

3. Move the chart to your projects root directory

   ```sh
   mv helm-master/chart chart
   rm -rf helm-master master.zip
	```
	
The provided Helm chart provides a number of configuration files, with the configurable values extracted into `chart/nodeserver/values.yaml`. In this file you provide the name of the Docker image to use, the number of replicas (instances) to deploy, etc.

Go ahead and modify the `chart/nodeserver/values.yaml` file to use your image, and to deploy 3 replicas:

1. Open the `chart/nodeserver/values.yaml` file
2. Change the `repository` field to `nodeserver-run`
3. Ensure that the `pullPolicy` is set to `IfNotPresent`
4. Change the `replicaCount` value to `3`

The `repository` field gives the name of the Docker image to use. The `pullPolicy` change tells Kuberentes to use a local Docker image if there is one available rather than always pulling the Docker image from a remote repository. Finally, the `replicaCount` states how many instances to deploy.
 
## 6. Deploying your Application to Kubernetes

Now that you have built a Helm chart for your application, the process for deploying your application has been greatly simplified.

Deploy your Express.js application into Kubernetes using the following steps:

<details>
<summary>microk8s only</summary>

You will need to push the image into the kubernetes container registry so that microk8s can access it. 

```sh
docker tag nodeserver-run:1.0.0 localhost:32000/nodeserver-run
docker push localhost:32000/nodeserver-run
helm install --name nodeserver \
  --set image.repository=localhost:32000/nodeserver-run  chart/nodeserver
```
</details>


1. Deploy your application into Kubernetes:

   ```sh
   helm install --name nodeserver chart/nodeserver
   ```

<details>
<summary>minikube only</summary>

If an error is encountered because the previous "docker run" is still running delete and retry the helm install:

   ```sh
   helm del --purge nodeserver
   helm install --name nodeserver chart/nodeserver
   ```

</details>

2. Ensure that all the "pods" associated with your application are running:

   ```sh
   kubectl get pods
   ```

Now everything is up and running in Kubernetes. It is not possible to navigate to `localhost:3000` as usual because your cluster isn't part of the localhost network, and because there are several instances to choose from.

You can forward the nodeserver-service to your laptop by:

  ```sh
  kubectl port-forward service/nodeserver-service 3000
  ```

You can now access the application endpoints from your browser.

## 7. Monitoring your Application with Prometheus

Installing Prometheus into Kubernetes can be done using its provided Helm chart. This step needs to be done in a new Terminal window as you'll need to keep the application port-forwarded to `localhost:3000`.

  ```sh
    helm install stable/prometheus --name prometheus --namespace prometheus
  ```

You can then run the following two commands in order to be able to connect to Prometheus from your browser:

  ```sh
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9090
  ```

This may fail with a warning about status being "Pending" until Prometheus has started, retry once status is "Running" for all pods:
  ```sh
  kubectl -n prometheus get pods --watch
  ```

You can now connect to Prometheus at [http://localhost:9090](http://localhost:9090).

This should show the following screen:
![prometheus-dashboard](https://raw.githubusercontent.com/CloudNativeJS/tutorial/master/resources/prometheus-dashboard.png)
Prometheus will be automatically collecting data from your Express.js application, allowing you to create graphs of your data.

To build your first graph, type `os_cpu_used_ratio` into the **Expression** box and click on the **Graph** tab:

![prometheus-graph](https://raw.githubusercontent.com/CloudNativeJS/tutorial/master/resources/prometheus-graph.png)


Whilst Prometheus provides the ability to build simple graphs and alerts, Grafana is commonly used to build more sophisticated dashboards.

### Installing Grafana into Kubernetes

Installing Grafana into Kubernetes can be done using its provided Helm chart:

```sh
helm install stable/grafana --set adminPassword=PASSWORD --name grafana --namespace grafana
```

You can then run the following two commands in order to be able to connect to Grafana from your browser:

```sh
export POD_NAME=$(kubectl get pods --namespace grafana -l "app=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace grafana port-forward $POD_NAME 3001:3000
```

You can now connect to Grafana at the following address, using `admin` and `PASSWORD` to login:

* [http://localhost:3001](http://localhost:3001)

This should show the following screen:

![grafana-home](https://raw.githubusercontent.com/CloudNativeJS/tutorial/master/resources/grafana-home.png)

In order to connect Grafana to the Prometheus service, next click on **Add data source** and select the first Time series database, "Prometheus".

This opens the a panel that should be filled out with the following entries:

* Name: `Prometheus`
* Type: `Prometheus` (may not be preset for some versions of Grafana)
* URL: `http://prometheus-server.prometheus.svc.cluster.local`

![grafana-datasource](https://raw.githubusercontent.com/CloudNativeJS/tutorial/master/resources/grafana-datasource.png)

Now click on **Save & Test** to check the connection and save the Data Source configuration.

Grafana now has access to the data from Prometheus.

### Installing a Kubernetes Dashboard into Grafana

The Grafana community provides a large number of pre-created dashboards which are available for download, including some which are designed to display Kubernetes data.

To install one of those dashboards, click on the **+** icon and select **Import**

In the provided panel, enter `1621` into the **Grafana.com Dashboard** field in order to import dashboard number 1621, and press **Tab**.

Note: If `1621` is not recognized, it may be necessary to download the JSON for [1621](https://grafana.com/grafana/dashboards/1621) (select "Download JSON"), and use "Upload JSON" in the Grafana UI.

This then loads the information on dashboard `1621` from Grafana.com.

Change the **Prometheus** field to `Prometheus` from `Compliant with Prometheus 1.5.2` and click **Import**.

![grafana-dashboard-import](https://raw.githubusercontent.com/CloudNativeJS/tutorial/master/resources/grafana-import-select.png)

This will then open the dashboard, which will automatically start populating with data about your Kubernetes cluster.

### Adding Custom Graphs

In order to extend the dashboard with your own graphs, click the **Add panel** icon on the top toolbar and select **Graph**.

![grafana-add-graph](https://raw.githubusercontent.com/CloudNativeJS/tutorial/master/resources/grafana-add-graph.png)

On some Grafana versions, after you click Add panel in toolbar, it is necessary to select ***Choose Visualization*** before selecting **Graph**.

This creates a blank graph. Select the **Panel Title** pull down menu and select **Edit**.

This opens an editor panel where you can select data that you'd like to graph.

Type `os_cpu_used_ratio` into the data box (or "Metrics" box on some version of Grafana), and a graph of your applications CPU data will show on the panel.

You can create more complex queries and apply filters according to any kubernetes value. For example, the following will show all of the HTTP request durations for your specific application:

* `http_request_duration_microseconds{kubernetes_name="nodeserver-service"}`

You now have integrated monitoring for both your Kubernetes cluster and your deployed Express.js application.

Here are some ideas you could explore to further your learning.

* Add a Singlestat that shows how many instances of you Express.js application are currently running
* Add a Singlestat that shows how many requests your Express.js app has responded to

### Congratulations! 🎉

You now have an Express.js application deployed with scaling using Docker and Kubernetes, with automatic restart, and full metrics-based monitoring enabled!

## Part 2: Building Cloud-Native Apps with Appsody

### Introduction to Appsody

Appsody is designed to help you develop containerized applications for the cloud.

Imagine you've defined your chosen cloud technologies, and you want to reuse these technologies across all of your Node.js microservices. You can use Appsody to define the standards for your applications, which will allow you to control the base that all of your applications are built off. You can define a set of technologies that are configurable, reusable, and already infused with cloud native capabilities. You can then maintain your standards, ensuring consistency and reliability.

This also means that not all software developers in your organisation need to have the knowledge or burden of managing the full cloud-native software development stack. With Appsody, developers can build applications for the cloud that are ready to be deployed to Kubernetes without necessarily being an expert on the underlying container technology.

For more background information on Appsody - checkout [this Medium article](https://medium.com/appsody/overview-c0cf1f2a244c).

### Prerequisites

Before getting started, you’ll need to install the Appsody CLI.

1. Follow the [Installing Appsody](https://appsody.dev/docs/getting-started/installation#installing-appsody) guide to install the CLI for your platform.

Verify that Appsody is installed by typing `appsody version`.

### Getting to know Apposdy

`appsody`

You should see output similar to the following:

```
The Appsody command-line tool (CLI) enables the rapid development of cloud native applications.

Complete documentation is available at [https://appsody.dev](https://appsody.dev).

Usage:
  appsody [command]

Available Commands:
  build       Locally build a docker image of your appsody project
  completion  Generates bash tab completions
  debug       Run the local Appsody environment in debug mode
  deploy      Build and deploy your Appsody project to your Kubernetes cluster
  extract     Extract the stack and your Appsody project to a local directory
  help        Help about any command
  init        Initialize an Appsody project with a stack and template app
  list        List the Appsody stacks available to init
  operator    Install or uninstall the Appsody operator from your Kubernetes cluster.
  repo        Manage your Appsody repositories
  run         Run the local Appsody environment for your project
  stop        Stops the local Appsody docker container for your project
  test        Test your project in the local Appsody environment
  version     Show Appsody CLI version

Flags:
      --config string   config file (default is $HOME/.appsody/.appsody.yaml)
      --dryrun          Turns on dry run mode
  -h, --help            help for appsody
  -v, --verbose         Turns on debug output and logging to a file in $HOME/.appsody/logs

Use "appsody [command] --help" for more information about a command.
```

Let’s take a look at which pre-built stacks we have available by entering:

```sh
appsody list
```

```
REPO        	ID                       	VERSION  	TEMPLATES        	DESCRIPTION                                              
experimental	java-spring-boot2-liberty	0.1.10   	*default         	Spring Boot on Open Liberty & OpenJ9 using Maven         
experimental	nodejs-functions         	0.1.5    	*simple          	Serverless runtime for Node.js functions                 
experimental	quarkus                  	0.1.5    	*default         	Quarkus runtime for running Java applications            
experimental	rocket                   	0.1.0    	*simple          	Rocket web framework for Rust                            
experimental	rust                     	0.1.3    	*simple          	Runtime for Rust applications                            
experimental	vertx                    	0.1.4    	*default         	Eclipse Vert.x runtime for running Java applications     
*incubator  	java-microprofile        	0.2.18   	*default         	Eclipse MicroProfile on Open Liberty & OpenJ9 using Maven
*incubator  	java-spring-boot2        	0.3.15   	*default, kotlin 	Spring Boot using OpenJ9 and Maven                       
*incubator  	kitura                   	0.2.1    	*default         	Runtime for Kitura applications                          
*incubator  	nodejs                   	0.2.5    	*simple          	Runtime for Node.js applications                         
*incubator  	nodejs-express           	0.2.8    	scaffold, *simple	Express web framework for Node.js                        
*incubator  	nodejs-loopback          	0.1.5    	*scaffold        	LoopBack 4 API Framework for Node.js                     
*incubator  	python-flask             	0.1.5    	*simple          	Flask web Framework for Python                           
*incubator  	starter                  	0.1.1    	*simple          	Runnable starter stack, copy to create a new stack       
*incubator  	swift                    	0.2.0    	*simple          	Runtime for Swift applications   
```

You’ll see that with the stacks available, we can develop new cloud-native applications using many lanugauges with a number of different, popular frameworks.


You can also register new Appsody repositories containing stacks created from the ground up or as forks of the default stacks shipped with Appsody. 

To see the available Appsody repositories run the following command: 

```
appsody repo list
```

```
NAME        	URL                                                                               
*incubator  	https://github.com/appsody/stacks/releases/latest/download/incubator-index.yaml
experimental	https://github.com/appsody/stacks/releases/latest/download/experimental-index.yaml
```

Appsodyhub will be there by default. Appsodyhub is the location where the appsody project releases its stacks.

### Node.js Express with Appsody Tutorial

Creating a new application with the `nodejs-express` Appsody Stack.

New Appsody based applications are initialized using `appsody init <stack>`, where the name of the stack is one of those listed when running appsody list. This both downloads the most recent copy of the Appsody Stack, and populates the project directory with a template that provides a basic project structure.

This needs to be done in a new, empty project directory, and Appsody will then use the name of the directory as the default name for the project.

1. Create a new directory for your project:

```sh
mkdir express-app
cd express-app
```

2. Initialize a new application using the nodejs-express Stack:

```sh
appsody init nodejs-express
```

This provides output similar to the following:

```
Running appsody init…
Downloading nodejs-express template project from https://github.com/appsody/stacks/releases/download/nodejs-express-v0.2.1/incubator.nodejs-express.templates.simple.tar.gz
Download complete. Extracting files from nodejs-express.tar.gz
Setting up the development environment
Running command: docker[pull appsody/nodejs-express:0.2]
Running command: docker[run — rm — entrypoint /bin/bash appsody/nodejs-express:0.2 -c find /project -type f -name .appsody-init.sh]
Successfully initialized Appsody project
```

This has downloaded a project template that provides a very basic project structure, along with the latest nodejs-express Stack which is a container image that contains:
A continuous, containerized run, debug and test environment for use during development.

A pre-configured Express.js server with built-in cloud-native capabilities.

A build configuration to provide optimized production-read container images for your application.

Your newly created application contains the following files:

```sh
.appsody-config.yaml
.gitignore
.vscode
app.js
package-lock.json
package.json
test
```

Where:
- `.appsody-config.yaml` configures the Appsody project, primarily controlling with version(s) of the Appsody Stack that the project can use.
- `.vscode` provides very basic integration with VSCode, including adding Run Task… entries for the Appsody CLI commands.
- `test` contains a set of tests for the application based on the mocha and chai frameworks.
- `app.js` provides a very simple “Hello from Appsody” Express.js route as an example.

- `package*.json` configured your application, and allows you to add your own additional module dependencies as normal.

Looking at the `app.js` file in detail, it contains the following:

```js
const app = require('express')()
app.get('/', (req, res) => {
  res.send(“Hello from Appsody!”);
});
 
module.exports.app = app;
```

This creates an instance of an Express.js app, and then registers a handler for get() requests on `/` that send() a response of "Hello from Appsody!".

The crucial characteristic that is required for the application to work with the `nodejs-express` Appsody Stack is that the application exports the create Express.js app using the following line:

```js
module.exports.app = app;
```

Ths is required as the Appsody Stack will apply the exported app onto its own pre-configured Express.js server that has already had support for cloud-native capabilities such as liveness and readiness probes, and metrics and observability built in.

### Developing your application with Appsody

Now that you have created your application, the next step is to see it running. To do that you can use the appsody run command in a terminal window. Alternatively, if you use VS Code, you can use the tasks that have been configured in the .vscode directory that was added as part of the template project.

1. Run your applcation using:
    1. From the terminal: `appsody run`
    2. In VSCode: Terminal > Run Task… > Appsody: run

    This starts a continuous development environment for your application, running inside a container.

2. Connect to the application in your browser at [http://localhost:3000](http://localhost:3000). This responds with `Hello from Appsody!`.

In addition to the handler for get requests on `/` that was defined in `app.js`, some other capabilities have been added by the Appsody Stack itself, these include health, liveness and readiness endpoints, a metrics endpoint, and an application performance analysis dashboard (during development only).

3. View the additional cloud-native capabilities that come prepackaged with the `nodejs-express` stack:
    - Health Endpoint: http://localhost:3000/health
    - Liveness Endpoint: http://localhost:3000/live
    - Readiness Endpoint: http://localhost:3000/ready
    - Metrics Endpoint: http://localhost:3000/metrics
    - Performance Dashboard: http://localhost:3000/appmetrics-dash

Now that your application is running under `appsody run`, as you make and save code changes in your application, those will automatically cause your application to be restarted and the changes reflected in the browser.

Make a code change to your project that will be reflected in the browser:

4. Open the `app.js` file
5. Change the file contents to:

```js
const app = require('express')()
 
app.get('/', (req, res) => {
  res.send(“Hello from NodeConfEU!”);
});

module.exports.app = app;
```

Save the file.

6. Connect to the application in your browser at http://localhost:3000.

This will display: `Hello from NodeConfEU!`

7. Finally, stop the continuous run environment by either:
Using `Ctrl-C` in the terminal window where appsody run is executing
Running appsody stop from the project directory

## Deploying to Kubernetes

You’ve finished writing your code and want to deploy to Kubernetes.

```
appsody deploy
```

At the end of the deploy, you should see an output like this:

```
Built docker image dev.local/nodejs
Using applicationImage of: dev.local/nodejs
Attempting to apply resource in Kubernetes ...
Running command: kubectl apply -f app-deploy.yaml --namespace default
Deployment succeeded.
Appsody Deployment name is: nodejs
Running command: kubectl get rt nodejs -o jsonpath="{.status.url}" --namespace default
Attempting to get resource from Kubernetes ...
Running command: kubectl get route nodejs -o jsonpath={.status.ingress[0].host} --namespace default
Attempting to get resource from Kubernetes ...
Running command: kubectl get svc nodejs -o jsonpath=http://{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[0].nodePort} --namespace default
Deployed project running at http://localhost:31059
```

The very last line tells you where the application is available.

To take a look at the deployment. Enter:

```
kubectl get all
```

You should see an output similar to this:

```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/appsody-operator-6bbddbd455-r65vp   1/1     Running   0          6m57s
pod/nodejs-7d84ddc98d-r7bnj             1/1     Running   0          44s


NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/appsody-operator   ClusterIP   10.100.219.241   <none>        8383/TCP         6m51s
service/kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP          9m13s
service/nodejs             NodePort    10.110.138.128   <none>        3000:30062/TCP   44s


NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/appsody-operator   1/1     1            1           6m57s
deployment.apps/nodejs             1/1     1            1           44s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/appsody-operator-6bbddbd455   1         1         1       6m57s
replicaset.apps/nodejs-7d84ddc98d             1         1         1       44s
The entries with nodejs correspond to Kubernetes resources created to support your application. The appsody-operator resources are those used by Appsody to perform the deployment.
```

It is worth noting at this point that this deployment was achieved without the need for writing, or even understanding, a Dockerfile or Kubernetes deployment file.

Now we can list the files in the project directory, which should contain files like this:

```sh
-rw-r--r--  1 app-deploy.yaml
-rw-r--r--  1 app.js
-rw-r--r--  1 package-lock.json
-rw-r--r--  1 package.json
drwxr-xr-x  3 test
```

The `app-deploy.yaml` is generated from the stack and used to deploy the application to Kubernetes. If you look inside the file, you will see entries for liveness and readiness probes, metrics, and the service port.

Check out the live and ready endpoints by pointing your browser at the following URLs, remembering to replace the port numbers with the port numbers from the output of the appsody deploy command:

http://localhost:30062/live

http://localhost:30062/ready

You should see something like:

```json
{
    "status":"UP",
    "checks":[]
}
```

These endpoints are provided by the stack health checks generated by the project starter.

Finally, let’s undeploy the application by entering:

```
appsody deploy delete
```

You should see something like this in the command-line output:

```
....
Deleting deployment using deployment manifest app-deploy.yaml
Attempting to delete resource from Kubernetes...
Running command: kubectl delete -f app-deploy.yaml --namespace default
Deployment deleted
....
```

Check that everything was undeployed using:

```
kubectl get all
```

You should see output similar to this:

```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/appsody-operator-6bbddbd455-r65vp   1/1     Running   0          13m


NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/appsody-operator   ClusterIP   10.100.219.241   <none>        8383/TCP   13m
service/kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP    15m


NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/appsody-operator   1/1     1            1           13m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/appsody-operator-6bbddbd455   1         1         1       13m
```

What if you decide you want to see the Container and Kubernetes configuration that Appsody is using, or you want to take your project elsewhere? You can do this as follows. Enter:

```
appsody extract --target-dir tmp-extract
```

You should see output similar to:

```
Extracting project from development environment
Pulling docker image appsody/nodejs-express:0.2
Running command: docker pull appsody/nodejs-express:0.2
0.2: Pulling from appsody/nodejs-express
Digest: sha256:d76e8c10487b42df998335940125501684e710876071eeb1dad84709d8b0b3c0
Status: Image is up to date for appsody/nodejs-express:0.2
docker.io/appsody/nodejs-express:0.2
[Warning] The stack image does not contain APPSODY_PROJECT_DIR. Using /project
Running command: docker create --name test-appsody-extract -v /Users/bethgriggs/test-appsody/:/project/user-app appsody/nodejs-express:0.2
Running command: docker cp test-appsody-extract:/project /Users/bethgriggs/.appsody/extract/test-appsody
Running command: docker rm test-appsody-extract -f
Project extracted to /Users/bethgriggs/test-appsody/tmp-extract
```

Let’s take a look at the extracted project:

```
cd tmp-extract
ls -al
```

You should see output similar to the following:

```
drwxr-xr-x   10 bgriggs  staff    320 Oct  3 17:55 .
drwxr-xr-x   11 bgriggs  staff    352 Oct  8 14:15 ..
-rw-r--r--    1 bgriggs  staff     48 Oct  3 14:41 .dockerignore
-rw-r--r--    1 bgriggs  staff    878 Oct  3 14:41 Dockerfile
drwxr-xr-x  274 bgriggs  staff   8768 Oct  3 17:55 node_modules
-rw-r--r--    1 bgriggs  staff  92237 Oct  3 17:55 package-lock.json
-rw-r--r--    1 bgriggs  staff    659 Oct  3 14:41 package.json
-rw-r--r--    1 bgriggs  staff   1462 Oct  3 14:41 server.js
drwxr-xr-x    3 bgriggs  staff     96 Oct  3 14:41 test
drwxr-xr-x   10 bgriggs  staff    320 Oct  8 14:04 user-app
```

These are the files for the project, including those provided by the stack. For example, the `package.json` has the core application definition for your application, and the Dockerfile is the one used to build and package the application. The user-app directory contains the Node.js project for your application.

You have seen how Appsody stacks and templates make it easy to get started with a new project, using a curated and consistent development and production environment. You have also seen how Appsody makes it really easy to build production-ready containers and deploy them to a Kubernetes environment.

:tada: 

## Further reading 

- [Building Cloud-Native Apps with Appsody](https://medium.com/appsody/overview-c0cf1f2a244c)
- [Cloud Functions for Node.js using Express.js (Connect) APIs](https://medium.com/appsody/nodes-functions-839a70289b82)
