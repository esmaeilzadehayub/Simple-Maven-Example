
# Simple-Maven-Example


# How to deploy Maven projects to Artifactory with GitLab CI/CD

## Introduction

In this project, we show how you can leverage the power of to build a [Maven](https://maven.apache.org/) project, deploy it to [Artifactory](https://jfrog.com/artifactory/), and then use it from another Maven application as a dependency.



First, you need an application to work with: in this specific case we'll use a
simple one, but it could be any Maven application. This will be the dependency
you want to package and deploy to Artifactory, to be available to other
projects.

### Prepare the dependency application

For this article you'll use a Maven app that can be cloned from our example
project:

1. Log in to your GitLab account
1. Create a new project by selecting **Import project from > Repo by URL**
1. Add the following URL:

   ```plaintext
   https://github.com/esmaeilzadehayub/Simple-Maven-Example.git
   ```

1. Click **Create project**

This application is nothing more than a basic class with a stub for a JUnit based test suite.
It exposes a method called `hello` that accepts a string as input, and prints a hello message on the screen.

The project structure is really simple, and you should consider these two resources:

- `pom.xml`: project object model (POM) configuration file
- `src/main/java/com/example/dep/Dep.java`: source of our application

### Configure the Artifactory deployment

The application is ready to use, but you need some additional steps to deploy it to Artifactory:

1. Log in to Artifactory with your user's credentials.
1. From the main screen, click on the `libs-release-local` item in the **Set Me Up** panel.
1. Copy to clipboard the configuration snippet under the **Deploy** paragraph.
1. Change the `url` value to have it configurable by using variables.
1. Copy the snippet in the `pom.xml` file for your project, just after the
   `dependencies` section. The snippet should look like this:

   ```xml
   <distributionManagement>
     <repository>
       <id>central</id>
       <name>83d43b5afeb5-releases</name>
       <url>${env.MAVEN_REPO_URL}/libs-release-local</url>
     </repository>
   </distributionManagement>
   ```

Another step you need to do before you can deploy the dependency to Artifactory
is to configure the authentication data. It is a simple task, but Maven requires
it to stay in a file called `settings.xml` that has to be in the `.m2` subdirectory
in the user's homedir.

Since you want to use a runner to automatically deploy the application, you
should create the file in the project's home directory and set a command line
parameter in `.gitlab-ci.yml` to use the custom location instead of the default one:

1. Create a folder called `.m2` in the root of your repository
1. Create a file called `settings.xml` in the `.m2` folder
1. Copy the following content into a `settings.xml` file:

   ```xml
   <settings xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd"
       xmlns="http://maven.apache.org/SETTINGS/1.1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
     <servers>
       <server>
         <id>central</id>
         <username>${env.MAVEN_REPO_USER}</username>
         <password>${env.MAVEN_REPO_PASS}</password>
       </server>
     </servers>
   </settings>
   ```

    Username and password will be replaced by the correct values using variables.

### Configure GitLab CI/CD for `simple-maven-dep`

Now it's time we set up [GitLab CI/CD](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) to automatically build, test and deploy the dependency!

GitLab CI/CD uses a file in the root of the repository, named `.gitlab-ci.yml`, to read the definitions for jobs
that will be executed by the configured runners. You can read more about this file in the [GitLab Documentation](https://docs.gitlab.com/ee/ci/yaml/README.md).

First of all, remember to set up variables for your deployment. Navigate to your project's **Settings > CI/CD > Environment variables** page
and add the following ones (replace them with your current values, of course):

- **MAVEN_REPO_URL**: `http://artifactory.example.com:8081/artifactory` (your Artifactory URL)
- **MAVEN_REPO_USER**: `gitlab` (your Artifactory username)
- **MAVEN_REPO_PASS**: `AKCp2WXr3G61Xjz1PLmYa3arm3yfBozPxSta4taP3SeNu2HPXYa7FhNYosnndFNNgoEds8BCS` (your Artifactory Encrypted Password)

Now it's time to define jobs in `.gitlab-ci.yml` and push it to the repository:

```yaml
image: maven:latest
variables:
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
cache:
  paths:
    - .m2/repository/
    - target/
build:
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS compile
test:
  stage: test
  script:
    - mvn $MAVEN_CLI_OPTS test
deploy:
  stage: deploy
  script:
    - mvn $MAVEN_CLI_OPTS deploy
  only:
    - master
```

The runner uses the latest [Maven Docker image](https://hub.docker.com/_/maven/),
which contains all of the tools and dependencies needed to manage the project
and to run the jobs.

Environment variables are set to instruct Maven to use the `homedir` of the repository instead of the user's home when searching for configuration and dependencies.

Caching the `.m2/repository folder` (where all the Maven files are stored), and the `target` folder (where our application will be created), is useful for speeding up the process
by running all Maven phases in a sequential order, therefore, executing `mvn test` will automatically run `mvn compile` if necessary.

Both `build` and `test` jobs leverage the `mvn` command to compile the application and to test it as defined in the test suite that is part of the application.

Deploy to Artifactory is done as defined by the variables we have just set up.
The deployment occurs only if we're pushing or merging to `master` branch, so that the development versions are tested but not published.

Done! Now you have all the changes in the GitLab repository, and a pipeline has already been started for this commit. In the **Pipelines** tab you can see what's happening.
If the deployment has been successful, the deploy job log will output:

```plaintext
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.983 s
```

>**Note**:
the `mvn` command downloads a lot of files from the internet, so you'll see a lot of extra activity in the log the first time you run it.
Yay! You did it! Checking in Artifactory will confirm that you have a new artifact available in the `libs-release-local` repository.

## Create the main Maven application

Now that you have the dependency available on Artifactory, it's time to use it!
Let's see how we can have it as a dependency to our main application.

### Prepare the main application

We'll use again a Maven app that can be cloned from our example project:

1. Create a new project by selecting **Import project from ➔ Repo by URL**
1. Add the following URL:

   ```plaintext
  https://github.com/esmaeilzadehayub/Simple-Maven-Example/edit/main/README.git
   ```

1. Click **Create project**

This one is a simple app as well. If you look at the `src/main/java/com/example/app/App.java`
file you can see that it imports the `com.example.dep.Dep` class and calls the `hello` method passing `GitLab` as a parameter.

Since Maven doesn't know how to resolve the dependency, you need to modify the configuration:

1. Go back to Artifactory
1. Browse the `libs-release-local` repository
1. Select the `simple-maven-dep-1.0.jar` file
1. Find the configuration snippet from the **Dependency Declaration** section of the main panel
1. Copy the snippet in the `dependencies` section of the `pom.xml` file.
   The snippet should look like this:

   ```xml
   <dependency>
     <groupId>com.example.dep</groupId>
     <artifactId>simple-maven-dep</artifactId>
     <version>1.0</version>
   </dependency>
   ```

### Configure the Artifactory repository location

At this point you defined the dependency for the application, but you still miss where you can find the required files.
You need to create a `.m2/settings.xml` file as you did for the dependency project, and let Maven know the location using environment variables.

Here is how you can get the content of the file directly from Artifactory:

1. From the main screen, click on the `libs-release-local` item in the **Set Me Up** panel
1. Click on **Generate Maven Settings**
1. Click on **Generate Settings**
1. Copy to clipboard the configuration file
1. Save the file as `.m2/settings.xml` in your repository

Now you are ready to use the Artifactory repository to resolve dependencies and use `simple-maven-dep` in your main application!

### Configure GitLab CI/CD for `simple-maven-app`

You need a last step to have everything in place: configure the `.gitlab-ci.yml` file for this project, as you already did for `simple-maven-dep`.

You want to leverage [GitLab CI/CD](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) to automatically build, test and run your awesome application,
and see if you can get the greeting as expected!

All you need to do is to add the following `.gitlab-ci.yml` to the repository:

```yaml
image: maven:latest
stages:
  - build
  - test
  - run
variables:
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
cache:
  paths:
    - .m2/repository/
    - target/
build:
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS compile
test:
  stage: test
  script:
    - mvn $MAVEN_CLI_OPTS test
run:
  stage: run
  script:
    - mvn $MAVEN_CLI_OPTS package
    - mvn $MAVEN_CLI_OPTS exec:java -Dexec.mainClass="com.example.app.App"
```

It is very similar to the configuration used for `simple-maven-dep`, but instead of the `deploy` job there is a `run` job.
Probably something that you don't want to use in real projects, but here it is useful to see the application executed automatically.

And that's it! In the `run` job output log you will find a friendly hello to GitLab!

## Conclusion

In this article we covered the basic steps to use an Artifactory Maven repository to automatically publish and consume artifacts.

A similar approach could be used to interact with any other Maven compatible Binary Repository Manager.
Obviously, you can improve these examples, optimizing the `.gitlab-ci.yml` file to better suit your needs, and adapting to your workflow.


# Create a Dockerfile for the application

Next, we need to add a line in our Dockerfile that tells Docker what base image we would like to use for our application.

```Dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:17-jdk-jammy as base
WORKDIR /app
COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:resolve
COPY src ./src

FROM base as development
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql", "-Dspring-boot.run.jvmArguments='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000'"]

FROM base as build
RUN ./mvnw package

FROM eclipse-temurin:17-jre-jammy as production
EXPOSE 8080
COPY --from=build /app/target/spring-petclinic-*.jar /spring-petclinic.jar
CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/spring-petclinic.jar"]

```
# Helm Charts

👷 Collection of Helm Charts for Kubernetes deployments ☸️

[[_TOC_]]

## Intro

[Helm](https://helm.sh/) is generally described as the package manager for Kubernetes and Charts are the format of packaging an application for Kubernetes.

A Helm Chart consists at least of the following components:

```plain
example-chart
├── Chart.yaml                # Contains information about the Chart
├── templates                 # Contains all Kubernetes manifests to be rendered
│   └── example-manifest.yaml
└── values.yaml               # Contains all variables the the manifests are rendered with
```

Creating a Helm Chart is as simple as gathering all your Kubernetes manifests in the `templates` folder and replacing everything you want to customize with variables ( `{{ .Values.<VARIABLE> }}`) you then place in your `values.yaml` file. Add the info about your Chart to `Chart.yaml` and you are ready to install it to Kubernetes with `helm install <DEPLOYMENT_NAME> <CHART_LOCATION> `.

## Best Practices

To create a Helm Chart quickly you can just run `helm create <CHART_NAME>`, which will create a generic Helm Chart for a web service. You can then customize the Chart to your liking.

This method ensures, that all Charts have the same structure, at least for their base, which makes it easier for CI and CD pipelines to work with the Charts.


### Testing your Chart

If you develop a new Helm Chart it's good to test it before deploying or pushing it. There are two basic ways to do this.

#### 1. `helm lint`

With `helm lint` you can lint a given Helm Chart to check for general errors and the following of best practices.
Simply run the following command:

```bash
helm lint <CHART> -f <PATH/TO/VALUES_FILE>
# for example
helm lint demo -f demo/values.yaml
```

#### 2. `helm template`

With `helm template` you can template a given Helm Chart, which will output the compiled Kubernetes manifests.
With this you see if your Helm Chart is able to compile and you can check the output to see what will be applied to Kubernetes.
Simply run the following command:

```bash
helm template <RELEASE_NAME> <CHART> -f <PATH/TO/VALUES_FILE>
# for example
helm template test-demo demo -f demo/values.yaml
```

<img width="490" alt="Screenshot 2022-10-30 at 09 42 05" src="https://user-images.githubusercontent.com/28998255/198869968-73370138-6cc5-4dd4-8322-a63e80c3beac.png">


# ArgoCD


[ArgoCD](https://argoproj.github.io/argo-cd/) is a declarative, GitOps continous delivery tool for Kubernetes. This means you describe the state of a resource in an ArgoCD config file and ArgoCD takes care that this resource in the cluster always comlies with the description you gave in the configfile.

By using ArgoCD and the concept of GitOps we get some major benefits:

- Our Kubernetes manifests stay valid
- We have a single source of truth for the application state on Kubernetes
- Easy versioning and rollback of changes to the application
.


## Workflow

The complete workflow of our CI + CD summarises to the following steps:

1. A developer pushes code to his projects repository
2. Gitlab CI performes the CI tasks specified in the `.gitlab-ci.yml` file, i.e.:
   - Running tests against the source code
   - Building a Dockerfile
   - Pushing the Dockerimage to a registry
   - Security & vulnerability scanning
3. After the above tasks one more CI task will update the corresponding Helm chart for this project, with the new Dockerimage tag
4. ArgoCD watches it's configured apps and will notice the changes in the Helm chart and will update the current deployment with the new version

![argocd-workflow](https://user-images.githubusercontent.com/28998255/198868788-86970581-c210-4b55-8f64-5615abb57523.png)


## Create a new ArgoCD managed app

ArgoCD is configured to manage all resources in the app directory.
Because ArgoCD provides Kubernetes CRDs for it's configuration we can place our
new ArgoCD app in the `apps` directory and ArgoCD will automatically pick it up.
The directory structure inside the `apps` directory represents our clusters,
but is just for better overview, the placement of an application definition
inside a specific directory does not mean it will be deployed to that cluster,
this is defined in the `spec.destination` section of the application definition.

An ArgoCD app definition looks something like the following:

```yaml
apiVersion: argoproj.io/v1alpha1                              # The Argo CRD version
kind: Application                                             # The Argo CRD kind
metadata:
  name: demo-app                                              # Name of the ArgoCD app
  namespace: argocd                                           # Namespace to deploy the app definition to (should always be `argocd`)
spec:
  project: development                                        # The ArgoCD project to deploy the app in
  source:
    repoURL: YourRepo  # The repo URL to get the kubernetes manifests from
    targetRevision: HEAD                                      # The branch to use
    path: demo                                                # The path inside the repo storing the manifests
    helm:
      valueFiles:                                             # A list of values files, inside the helm chart, to use for this deployment
        - values.yaml
  destination:
    server: YourServer    # The target cluster (Check Rancher cluster URLs)
    namespace: demo                                           # The namespace in which to deploy the manifests
  syncPolicy:
    automated:
      prune: true                                             # Prune the app on sync
      selfHeal: true                                          # Try to heal the app on failure
    syncOptions:
      - CreateNamespace=true                                  # Create the specified namespace, if non existent
```

To create a new ArgoCD managed app just create a YAML file like the one above in the `apps` directory and customize the parts you want to change.

# check your service 
url:8000/api/v1

<img width="890" alt="Screenshot 2022-10-30 at 09 37 40" src="https://user-images.githubusercontent.com/28998255/198869799-6163e4ca-c3d7-48f5-b1bf-947d90e21f25.png">

