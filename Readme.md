# Kubernetes Continuous Deployment Pipeline

An open source Jenkins pipeline tool purpose built for Kubernetes containers.

* [Quickstart](#quickstart)
* [Project Details](#details)


## Requirements:
Our Jenkins plugin deploys all containers to Kubernetes minions that have a label of

```
role=minion
```
This can be added to your workers/minions with
```
kubectl label nodes <node_name> role=minion
```

<a id="quickstart"></a>
## Quickstart

  * Create Repo
  * Deploy Jenkins
  * Configure Bitesize files

<br>
### Create a Repo
Create a git repo where you will store the configuration files. This can be pretty much anywhere. You will need to provide an ssh key for access to the repo from Jenkins. Read-Only access is recommended.
<br><br>

### Deploy Jenkins

precursors:<br>
**git repo link**  - like 'git@github.com:pearsontechnology/deployment-pipeline-jenkins-plugin.git'<br>
**ssh private key**<br><br>

#### Jenkins Kubernetes Config Template

**%%NAMESPACE%%** - Namespace you want to deploy Jenkins into<br>
**%%JENKINS_ADMIN_USER%%** - Jenkins Admin user name<br>
**%%JENKINS_ADMIN_PASSWORD%%** - Jenkins Admin password<br>
**%%SEED_JOBS_REPO%%** - location of git repo where config files will exist<br>
**%%GIT_PRIVATE_KEY%%** - Private SSH key used to access the git repo<br>
**%%JENKINS_IMAGE%%** - as of this writing - bitesize-registry.default.svc.cluster.local:5000/geribatai/jenkins:3.4.28<br><br>


```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: jenkins
  name: jenkins
  namespace: %%NAMESPACE%%
spec:
  replicas: 1
  selector:
    matchLabels:
      name: jenkins
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: jenkins
      name: jenkins
    spec:
      containers:
      - name: jenkins
        env:
        - name: JAVA_OPTS
          value: -Djava.awt.headless=true -Xms512m -Xmx2g -XX:MaxPermSize=1048M -Dorg.apache.commons.jelly.tags.fmt.timeZone=America/New_York
            -Dcom.sun.management.jmxremote.local.only=false
        - name: JENKINS_ADMIN_USER
          value: %%JENKINS_ADMIN_USER%%
        - name: JENKINS_ADMIN_PASSWORD
          value: %%JENKINS_ADMIN_PASSWORD%%
        - name: SEED_JOBS_REPO
          value: %%SEED_JOBS_REPO%%
        - name: GIT_PRIVATE_KEY
          value: |
                 %%GIT_PRIVATE_KEY%%
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: %%JENKINS_IMAGE%%
        imagePullPolicy: Always
        securityContext:
          runAsUser: 1000
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 50000
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /var/jenkins_home
          name: jenkins-data
        - mountPath: /var/jenkins_home/repository
          name: aptly-repository
      - image: geribatai/aptly:1.0.0
        imagePullPolicy: Always
        name: aptly
        ports:
        - containerPort: 9797
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /aptly
          name: aptly-repository
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext:
        fsGroup: 1000
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: jenkins-data
      - emptyDir: {}
        name: aptly-repository
```
<br>

#### Configure Bitesize files
Now lets setup the config files. These files are called bitesize files because that's how they originally came about. Thus all the files end with .bitesize.<br>

These config files will help you get a sample app up and running. Refer "Project Details" for additional information on Build and Job definitions.


##### environments.bitesize
```
project: docs-dev
environments:
  - name: production
    namespace: docs-dev
    deployment:
      method: rolling-upgrade
    services:
      - name: docs-app
        external_url: kubecon.dev-bite.io
        port: 80 # this is the port number the application responds on in each container/instance/pod
        ssl: "true"  # must be in double quotes - interpolated as a string
        replicas: 2
```

##### build.bitesize
```
# see https://mycloud.atlassian.net/wiki/display/BITE/Jenkins
project: docs-dev
components:
  - name: docs-app
    os: linux
    dependencies:
      - type: debian-package
        package: php5
        repository: ppa:ondrej/php
      - type: debian-package
        package: libapache2-mod-php5
      - type: debian-package
        package: python2.7
      - type: debian-package
        package: python-pip
      - type: pip-package
        package: PyGithub
      - type: pip-package
        package: pyyaml
      - type: debian-package
        package: couscous
        location: https://s3.amazonaws.com/bitesize-sandbox-files/couscous.deb_1.0_amd64.deb
    repository:
      git: git@github.com:pearsontechnology/kubecon_docs.git
      branch: master
    env:
      - name: GIT_USERNAME
        value: kubecondemos@gmail.com
      - name: GIT_PASSWORD
        value: 21874b392e38ded25c91a3ecfba57ba384126087
    build:
      - shell: cat /dev/null > couscous.yml
      - shell: python docsgen.py
      - shell: couscous generate
      - shell: mkdir -p var/www/html
      - shell: cp run.sh var/
      - shell: cp -a .couscous/generated/* var/www/html
      - shell: fpm -s dir -n docswebsite --iteration $(date "+%Y%m%d%H%M%S") -t deb var
    artifacts:
      - location: "*.deb"
```

##### application.bitesize
```
project: docs-dev # aka namespace
applications:
  - name: docs-app
    runtime: ubuntu-httpdfcgi:1.3
    version: "0.8.35"
    dependencies:
      - name: docswebsite
        type: debian-package
        origin:
          build: docs-app
        version: 1.0
    command: "/var/run.sh"
```



<a id="details"></a>
## Project Details

Jenkins uses a custom workflow to build and deploy applications. The whole CI/CD pipeline is built using just three manifest files, which have very distinctive roles in the build process.

-   **build.bitesize** - defines how to build one or more components
-   **application.bitesize** - defines how to build one or more
    applications using required components and
    external dependencies
-   **environments.bitesize** - defines how to layout environments,
    which applications (services) to run in them, what tests to run
    against your applications and the method to deploy applications.
    Builds out the whole CI/CD pipeline


Your project will store these files in a git repository. It can be either your source repository, or a repository dedicated just to managing these three files. At Pearson we've found our dev teams prefer to manage these files separately from their code repositories.
<br>
First, let’s define some terms to understand what exactly they mean in our context:

| Term | Definition |
| --- | :--- |
| *Component* | Individual code repository necessary to create an Application. Your application will have one or more internal components. |
| *Application*      | Artifacts as a single Artifact necessary to create a running instance (a collection of components). Application includes the full stack required to run an instance. |
| *Build* | Process that outputs artifacts (Debian Package, Docker Image, etc…) |
| *Job* | An instance (success or fail) of a Build |
| *Build Definition* | What actions need to be executed to generate the output artifact. |
| *Job Definition* | Specifics related to the Job, generally Version Number, Tags, .... |
| *Build Dependency* | A tool or utility necessary to create a Component Artifact. |
| *Service* | Instance (or multiple grouped instances in HA mode) of a running application. |
| *Environment* | Collection of services, grouped together to represent a fully working application stack. |
