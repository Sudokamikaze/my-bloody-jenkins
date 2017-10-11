# My Bloody Jenkins - An opinionated Jenkins Docker Image

## Introduction
I've been working a lot with Jenkins/Pipline and Docker in the last couple of years, and wanted to share my experience on these subjects.

Jenkins is great! Jenkins combined with Docker is even greater...

But...

It is HARD to get it work.

Many small tweaks, plugins that do not work as expected in combination with other plugins, and not to mention configuration and automation.

When it comes to Jenkins combined with Docker it is even harder:

* How to cope with Jenkins master data in a cluster of ECS/Kubernetes?
    * IP Address is not static...
    * JENKINS_HOME contents should be available at all time (Distributed Filesystem/NFS/EFS) - but master workspace should be faster.
    * Ephemeral JNLP Docker Slaves can take time to start due to untuned node provisioning strategy
    * How to keep Docker slaves build docker images
    * Host mounted volumes permissions issues
    * ...

So... Since I spilled some blood on that matter, I've decided to create an ***opinionated*** Jenkins Docker Image that covers some of these subjects...

Therefore ***My Bloody Jenkins***...

### Main Decisions

* Jenkins does not rely on external configuration management tools. I've decided to make it an 'autopilot' docker container, using environment variables that are passed to the container or can be fetched from a centrailized KV store such as Consul.
* I am focusing in docker cloud environment - meaning Jenkins master is running inside docker, slaves are ephemeral docker containers running in Kubernetes/ECS/Swarm clusters
* Only using JNLP slaves and not SSH slaves
* Jenkins master does not have any executer of its own - let the slaves do the actual job
* SCM - Focusing only on git
* Complex configuration items are defined in yaml and can be passed as environment variables
* Plugins
    * Focus on pipeline based plugins
    * Plugins must be baked inside the image and should be treated as a the jenkins binary itself. Need to update/add/remove a plugin?? - Create a new image!
* Focus on the following configuration items:
    * Credentials
        * User/Password
        * SSH Keys
        * Secret Text
        * AWS Credentials
    * Security
        * Jenkins database
        * LDAP
        * Using Project Matrix Authorization Strategy
    * Clouds - As I said above - Only docker based and only JNLP
        * ECS
        * Kubernetes
        * Docker plugin
    * Notifications
        * Email
        * Slack
        * Hipchat
    * Script Approvals - since we are using pipeline, sometimes you must approve some groovy methods (We all understand why it is needed, but this one is bloody...)
    * Tools and installers
        * Apache Ant
        * Apache Maven
        * Gradle
        * JDK
        * Xvfb
        * SonarQube Runner
    * Global Pipeline Libraries - Yes yes yes. Use them a lot...
    * Seed Jobs - As I said, it is an 'autoplilot' Jenkins! We do not want to add/remove/update jobs from the UI. Seed Jobs are also Jenkins pipeline jobs that can use JobDSL scripts to drive the jobs CRUD operations
    * Misc R&D lifecycle tools
        * Checkmarx
        * Jira
        * SonarQube

Ok, enough talking...

## Docker Image Reference
## Environment Variables
The following Environment variables are supported

* ___JENKINS_ENV_ADMIN_USER___ - (***mandatory***) Represents the name of the admin user. If LDAP is your choice of authentication, then this should be a valid LDAP user id. If Using Jenkins Database, then you also need to pass the password of this user within the [configuration](#configuration-reference).
* ___JAVA_OPTS\_*___ - All JAVA_OPTS_ variables will be appended to the JAVA_OPTS during startup. Use them to control options (system properties) or memory/gc options. I am using few of them by default to tweak some known issues:
    * JAVA_OPTS_DISABLE_WIZARD - disables the Jenkins 2 startup wizard
    * JAVA_OPTS_CSP - Default content security policy for HTML Publisher/Gatling plugins - See https://wiki.jenkins.io/display/JENKINS/Configuring+Content+Security+Policy
    * JAVA_OPTS_LOAD_STATS_CLOCK - This one is sweet (: - Reducing the load stats clock enables ephemeral slaves to start immediately without waiting for suspended slaves to be reaped
* ___JENKINS_ENV_CONFIG_YAML___ - The [configuration](#configuration-reference) is stored in '/etc/jenkins-config.yml' file. When this variable is set, the contents of this variable is written to the file before starting Jenkins and can be fetched from Consul and also be watched so jenkins can update its configuration everytime this variable is being changed. Since the contents of this variable contains secrets, it is wise to store and pass it from Consul/S3 bucket. In any case, once the file is written, this variable is being unset, so it won't appear in Jenkins 'System Information' page (As I said, blood...)
* ___JENKINS_ENV_HOST_IP___ - When Jenkins is running behind an ELB or a reverse proxy, JNLP slaves must know about the real IP of Jenkins, so they can access the 50000 port. Usually they are using the Jenkins URL to try to get to it, so it is very important to let them know what is the original Jenkins IP Address. If the master has a static IP address, then this variable should be set with the static IP address of the host.
* ___JENKINS_ENV_HOST_IP_CMD___ - Same as ___JENKINS_ENV_HOST_IP___, but this time a shell command expression to fetch the IP Address. In AWS, it is useful to use the EC2 Magic IP: ```JENKINS_ENV_HOST_IP_CMD='curl http://169.254.169.254/latest/meta-data/local-ipv4'```


## Configuration Reference
The '/etc/jenkins-config.yml' file is divided into main configuration sections. Each section is responsible for a specific aspect of jenkins configuration.

### Security Section
Responsible for:
* Setting up security realm
    * jenkins_database - the adminPassword must be provided
    * ldap - LDAP Configuration must be provided
* User/Group Permissions dict - Each key represent a user or a group and its value is a list of Jenkins [Permissions IDs](https://wiki.jenkins.io/display/JENKINS/Matrix-based+security)

```yaml
# jenkins_database - adminPassword must be provided
security:
    realm: jenkins_database
    adminPassword: S3cr3t
```

```yaml
# ldap - ldap configuration must be provided
security:
    realm: ldap
    server: myldap.server.com:389 # mandatory
    rootDN: dc=mydomain,dc=com # mandatory
    managerDN: cn=search-user,ou=users,dc=mydomain,dc=com # mandatory
    managerPassword: <passowrd> # mandatory
    userSearchBase: ou=users
    userSearchFilter: uid={0}
    groupSearchBase: ou=groups
    groupSearchFilter: cn={0}
    ########################
    # Only one is mandatory - depends on the group membership strategy
    # If a user record contains the groups, then we need to set the
    # groupMembershipAttribute.
    # If a group contains the users belong to it, then groupMembershipFilter
    # should be set.
    groupMembershipAttribute: group
    groupMembershipFilter: memberUid={1}
    ########################
    disableMailAddressResolver: false # default = false
    connectTimeout: 5000 # default = 5000
    readTimeout: 60000 # default = 60000
    displayNameAttr: cn
    emailAttr: email
```

```yaml
# Permissions - each key represents a user/group and has list of Jenkins Permissions
security:
    realm: ...
    permissions:
        authenticated: # Special group
            - hudson.model.Hudson.Read # Permission Id - see
            - hudson.model.Item.Read
            - hudson.model.Item.Discover
            - hudson.model.Item.Cancel
        junior-developers:
            - hudson.model.Item.Build
```

### Tools Section
Responsible for:
* Setting up tools locations and auto installers

The following tools are currently supported:
* JDK - (type: jdk)
* Apache Ant (type: ant)
* Apache Maven (type: maven)
* Gradle (type: gradle)
* Xvfb (type: xvfb)
* SonarQube Runner (type: sonarQubeRunner)

The following auto installers are currently supported:
* Oracle JDK installers
* Maven/Gradle/Ant/SonarQube version installers
* Shell command installers (type: command)
* Remote Zip/Tar files installers (type: zip)

The tools section is a dict of tools. Each key represents a tool location/installer ID.
Each tools should have either home property or a list of installers property.

Note: For Oracle JDK Downloaders to work, the oracle download user/password should be provided as well.

```yaml
tools:
  oracle_jdk_download: # The oracle download user/password should be provided
    username: xxx
    password: yyy
  installations:
    JDK8-u144: # The Tool ID
      type: jdk
      installers:
        - id: jdk-8u144-oth-JPR # The exact oracle version id
    JDK7-Latest:
      type: jdk
      home: /usr/java/jdk7/latest # The location of the jdk
    MAVEN-3.1.1:
      type: maven
      home: /usr/share/apache-maven-3.1.1
    MAVEN-3.5.0:
      installers:
        - id: '3.5.0' # The exact maven version to be downloaded
    ANT-1.9.4:
      type: ant
      home: /usr/share/apache-ant-1.9.4
    ANT-1.10.1:
      type: ant
      installers:
        - id: '1.10.1' # The exact ant version to be downloaded
    GRADLE-4.2.1:
      type: gradle
      ... # Same as the above - home/installers with id: <gradle version>
    SONAR-Default:
      type: sonarQubeRunner
      installers:
        - id: '3.0.3.778'
      ... # Same as the above - home/installers with id: <sonar runner version>

    # zip installer and shell command installers
    ANT-XYZ:
      type: ant
      installers:
        - type: zip
          label: centos7 # nodes labels that will use this installer
          url: http://mycompany.domain/ant/xyz/ant.tar.gz
          subdir: apache-ant-zyz # the sub directoy where the tool exists within the zip file
        - type: command
          label: centos6 # nodes labels that will use this installer
          command: /opt/install-ant-xyz
          toolHome: # the directoy on the node where the tool exists after running the command
```

### Credentials Section
Responsible for:
* Setting up credentials to be used later on by pipelines/tools
* Each credential has an id, type, description and arbitary attributes according to its type
* The following types are supported:
    * type: text - simple secret. Mandatory attributes:
        * text - the text to encrypt
    * type: aws - an aws secret. Mandatory attributes:
        * access_key - AWS access key
        * secret_access_key - AWS secret access key
    * type: userpass - a user/password pair. Mandatory attributes:
        * username
        * password
    * type: sshkey - an ssh private key in PEM format. Mandatory attributes:
        * username
        * privatekey - PEM format text
        * passphrase - not mandatory, but encouraged

```yaml
# Each top level key represents the credential id
credentials:
  slack:
    type: text
    description: The slace secret token
    text: slack-secret-token
  hipchat:
    type: text
    text: hipchat-token
  awscred:
    type: aws
    access_key: xxxx
    secret_access_key: yyyy
  gituserpass:
    type: userpass
    username: user
    password: password1234
  gitsshkey:
    type: sshkey
    description: git-ssh-key
    username: user
    passphrase: password1234
    privatekey: | ## This is why I love yaml... So simple to handle text with newlines
      -----BEGIN RSA PRIVATE KEY-----
      Proc-Type: 4,ENCRYPTED
      DEK-Info: AES-128-CBC,B1615A0CA4F11333D058A5A9C4D0144E

      wxr6qxA/gcj+Bf1he0YRaqH2HbvHXIshrXTFq7oet5OeF1oaX4yyejotI9oPXvvM
      X9jzLpPwhdpriuFKJKr9jc+1rto/71QExTYEaAWwfYi1EVb1ERGmG4DMANqBKssO
      FTn01t4wew3d6DPcAIwFBT7gvlJfg0poCxae1fhsXGijMy2YryiU+BcV0BYsM6Lj
      VAfn9+djoxPKTv3wPFZPXVrzSkWG8IFUcLBIKE6hf3xxwV5FPHDSegAnwTBV9sVB
      BkVfAHDkzecEtK3iHqa9QUsW014TTLZ7Rbbzh6mskrGxgjgDXXjbdEYbJDtSES6E
      d2o1nXqJEsDdUrSWAoaViR052KyW8f8n//LEjI1a6aveOQWoWXgPwD9jnp5cPrGv
      mWGrZmhWLh0rx61qG+bBVEfbGmLmbi74jxq1/6vaAF4ChtfBks2mpOMMOKf+bKar
      w1laJKgwFRWO7iYql9dHzny2GXy4z6hjD5g3omEdFyWh3GSW8NNkXyojml7tlIa6
      ln286//PSmN0dLZptULAr8A4Bp+aGgybnla7H2F5/s2mGc39MrodFPFbpjp50d4R
      2x4uV4ofVvVExv5wWSdQ60o7trSvBWqwu4MDm2yWCxUiay8I8EF2iM0etutlWm5N
      V9aD8TTC5zHLbwY7YI0OvOXyCWgI5CW1tTsnoDxR1H0aDByuSL+Q+8I5gxRKdJlb
      fYZ683g8AuTkKilHQ6KINAAUuvMvgSWzOa/BU9L7Xqn99w2WgdheLMkdp6O4dJX4
      f4vFTzms+tBVXwqybac/8HZ4uW163pgnBpH2bVFi7/qyd8sl8TYAODN70R2oLv3c
      HBjzI/078G9B4WkpL99FK9cWsfCleIM+HIGeQ9jPDK/hzAhlKIIQxzIaomhcRZlW
      Oc3+zlioBuQSBVtf1UJMrCFSfr9Gq+lbhsD5k8bscU38d5R8EHDGvicpr/KIwjGn
      piNmO7Nz2KcJaLUX6D1oG0p4ioan7js27eBGVnur73hNySbFycwiTGdZkp1CHM7D
      OH2iOMibQGMGYWoci/SoIOgp54Meq4WkEpV5wwr6aCuTzsgnVJXVuMmBk8dnvyat
      EKflp+2kKCj9uPo/QUoozNAuNzyDJU95E8ZfLyT5G9GCQtlf2EnrFZIeQ040xBQC
      6vGQLodjxIG1X8ejDv69FIW66qyofAWVuwCO6wuCWEdLLZrNNhjyPCnnxv5Kw8oC
      nB8/YDntqKg2GqpDu4s0bzAt0CPbMQydvD5x4AWKYCm4HQIF3qX544yUKd9vuO3l
      5t8JE8l4ETGMieKFpE9YiVbobye8iNRyIYVBuvlk4lq+xifw7i3Crmr/+KB+2ABZ
      8TshgEjw8G7TR7qZjZGONCBJ6ozUNR5ipUANc81AA3AUGilBeC4lLUcvatsixtWz
      6BDHwOYfLbfm8YIxDELMt13f++sxKoed4EjFJu+JIjEZlRomPf9pZEmZwTRVASsc
      CbwJjRc022b7HIsetGBYu76KK/Fs25D5JTZjQ3ylMKwhBjrOT7d8Xm90/6eg4hvE
      4c9bdLH+2Xuc6qv/oBoFzVd19c3DiVfns2/5BohfG+pbNwZUVR1vjP/BVDgwDBc+
      -----END RSA PRIVATE KEY-----

```        

### Notifiers Section
TBD

### Pipeline Libraries Section
TBD

### Script Approval Section
TBD

### Clouds Section
TBD

### Seed Jobs Section
TBD

### Sonar Qube Servers Section
TBD

### Checkmarks Section
TBD

### Jira Section
TBD
