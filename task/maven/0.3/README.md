# Maven

This Task can be used to run a Maven goals on a simple maven project or on a multi-module maven project.

## Install the Task

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/maven/0.3/maven.yaml
```

## Parameters

- **MAVEN_IMAGE**: The base image for maven (_default_: `gcr.io/cloud-builders/mvn`)
- **GOALS**: Maven `goals` to be executed
- **MAVEN_MIRROR_URL**: Maven mirror url (to be inserted into ~/.m2/settings.xml)
- **SERVER_USER**: Username to authenticate to the server (to be inserted into ~/.m2/settings.xml)
- **SERVER_PASSWORD**: Password to authenticate to the server (to be inserted into ~/.m2/settings.xml)
- **PROXY_USER**: Username to login to the proxy server (to be inserted into ~/.m2/settings.xml)
- **PROXY_PASSWORD**: Password to login to the proxy server (to be inserted into ~/.m2/settings.xml)
- **PROXY_HOST**: Hostname of the proxy server (to be inserted into ~/.m2/settings.xml)
- **PROXY_NON_PROXY_HOSTS**: Non proxy hosts to be reached directly bypassing the proxy (to be inserted into ~/.m2/settings.xml)
- **PROXY_PORT**: Port number on which the proxy port listens (to be inserted into ~/.m2/settings.xml)
- **PROXY_PROTOCOL**: http or https protocol whichever is applicable (to be inserted into ~/.m2/settings.xml)
- **CONTEXT_DIR**: The context directory within the repository for sources on which we want to execute maven goals. (_Default_: ".")

## Workspaces

- **source**: `PersistentVolumeClaim`-type so that volume can be shared among `git-clone` and `maven` task
- **cache**: Directory where maven artifact cache are stored.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-source-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

## Usage

This Pipeline and PipelineRun runs a Maven build on a particular module in a multi-module maven project

### With Defaults

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: maven-test-pipeline
spec:
  workspaces:
    - name: shared-workspace
    - name: maven-settings
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: https://github.com/redhat-developer-demos/tekton-tutorial
        - name: subdirectory
          value: ""
        - name: deleteExisting
          value: "true"
    - name: maven-run
      taskRef:
        name: maven
      runAfter:
        - fetch-repository
      params:
        - name: CONTEXT_DIR
          value: "apps/greeter/java/quarkus"
        - name: GOALS
          value:
            - -DskipTests
            - clean
            - package
      workspaces:
        - name: maven-settings
          workspace: maven-settings
        - name: source
          workspace: shared-workspace
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: maven-test-pipeline-run
spec:
  pipelineRef:
    name: maven-test-pipeline
  workspaces:
    - name: maven-settings
      emptyDir: {}
    - name: shared-workspace
      persistentvolumeclaim:
        claimName: maven-source-pvc
```

---

### With Custom Maven Params

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: maven-test-pipeline
spec:
  workspaces:
    - name: shared-workspace
    - name: maven-settings
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: https://github.com/redhat-developer-demos/tekton-tutorial
        - name: subdirectory
          value: ""
        - name: deleteExisting
          value: "true"
    - name: maven-run
      taskRef:
        name: maven
      runAfter:
        - fetch-repository
      params:
        - name: MAVEN_MIRROR_URL
          value: http://repo1.maven.org/maven2
        - name: CONTEXT_DIR
          value: "apps/greeter/java/quarkus"
        - name: GOALS
          value:
            - -DskipTests
            - clean
            - package
      workspaces:
        - name: maven-settings
          workspace: maven-settings
        - name: source
          workspace: shared-workspace
```

`PipelineRun` same as above in case of default values

---

### With Custom /.m2/settings.yaml

A user provided custom `settings.xml` can be used with the Maven Task. To do this we need to mount the `settings.xml` on the Maven Task.
Following steps demonstrate the use of a ConfigMap to mount a custom `settings.xml`.

1. create configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-maven-settings
data:
  settings.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <settings>
      <mirrors>
        <mirror>
          <id>maven.org</id>
          <name>Default mirror</name>
          <url>http://repo1.maven.org/maven2</url>
          <mirrorOf>central</mirrorOf>
        </mirror>
      </mirrors>
    </settings>
```

or

```bash
oc create configmap custom-maven-settings --from-file=settings.xml
```

2. create `Pipeline` and `PipelineRun`

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: maven-test-pipeline
spec:
  workspaces:
    - name: shared-workspace
    - name: maven-settings
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: https://github.com/redhat-developer-demos/tekton-tutorial
        - name: subdirectory
          value: ""
        - name: deleteExisting
          value: "true"
    - name: maven-run
      taskRef:
        name: maven
      runAfter:
        - fetch-repository
      params:
        - name: CONTEXT_DIR
          value: "apps/greeter/java/quarkus"
        - name: GOALS
          value:
            - -DskipTests
            - clean
            - package
      workspaces:
        - name: maven-settings
          workspace: maven-settings
        - name: source
          workspace: shared-workspace
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: maven-test-pipeline-run
spec:
  pipelineRef:
    name: maven-test-pipeline
  workspaces:
    - name: maven-settings
      configMap:
        name: custom-maven-settings
    - name: shared-workspace
      persistentvolumeclaim:
        claimName: maven-source-pvc
```