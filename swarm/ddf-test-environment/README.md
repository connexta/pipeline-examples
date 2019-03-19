# Deploying a DDF Test Environment

Basic example of a jenkins pipeline that deploys a two ddf nodes into a docker swarm and runs a test against them

## The Stack

The following is a breakdown of what is going on in the docker [stack](stack.yml) in this example

### Services used in this stack

```yaml
...
services:
  ddf1:
    ...
  ddf1-solr:
    ...
  ddf2:
    ...
  ddf2-solr:
    ...
```

This stack contains 4 services:

* ddf1
* ddf2
* ddf1-solr
* ddf2-solr

External solr instances are used for both ddf nodes

### DDF Configuration

Each ddf node includes the following configuration options:

```yaml
services: 
    ddf1:
      environment:
        SOLR_URL: "http://ddf1-solr:8983/solr"
        EXTERNAL_HTTPS_PORT: 443
        EXTERNAL_HTTP_PORT: 80
        JAVA_MAX_MEM: ${DDF_JAVA_MEM:-4}
        INSTALL_PROFILE: standard
        SSH_ENABLED: 'true'
    ...
```

These options configure ddf for the following:
* external solr: uses the name of the solr service in the stack file i.e. ddf1-solr and the default port for solr which is 8983
* external https port: set to 443 since this will be sitting behind a loadbalancer that is used to expose all swarm services
* external http port: set to 80 since this will be sitting behind a loadbalancer that is used to expose all swarm services
* java memory: sets the java memory allocation for ddf to 4 Gb. Allows override via the `DDF_JAVA_MEM` environment variable
* install profile: installs the standard profile
* ssh: keeps the ssh client port active (this is not technically necessary but without it ddf will unbind 8101)

### Solr Configuration

```yaml
services:
  ddf1-solr:
    environment:
      CORES: "catalog metacard_cache alerts preferences subscriptions notification activity workspace"
```

Solr configuration is pretty straightforward:
* cores: list of solr core names to pre-create. 
    * This is necessary because the ddf solr client can only create them automatically under 2 conditions:
        * When using solr cloud
        * When using local solr on the same filesystem

The solr image used is built in the ddf repo and simply extends the official image so it will use port 8983 by default

### Exposing DDF via Traefik Loadbalancer

The swarm has a [traefik](https://traefik.io/) loadbalancer for exposing services on the fly based on labels

To expose a DDF the following is done:

```yaml
services:
  ddf1: 
    ...
    deploy:
      ...
      labels:
        - "traefik.enable=true"
        - "traefik.https.frontend.rule=Host: ddf1${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}"
        - "traefik.https.protocol=https"
        - "traefik.https.port=8993"
        - "traefik.https.frontend.entryPoints=https"
        - "traefik.http.frontend.rule=Host: ddf1${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}"
        - "traefik.http.protocol=http"
        - "traefik.http.port=8181"
        - "traefik.http.frontend.entryPoints=http"
    hostname: ddf1${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}
    ...
  ...
  networks:
    - proxy
    ...
networks:
  proxy:
    external: true
```

Traefik uses a series of labels to discover services and define rules for routing external clients to those services

Breakdown of labels (in order):
* enable traefik
* define a frontend called `https` and set a routing rule for hostname matching: i.e. `Host: ddf1.test`
* set the https frontend to use https: this instructs traefik that this backend connection uses tls, otherwise it would assume backend requests were unencrypted
* set the https frontend to map to port 8993: this will route trafic from port 443 on the loadbalancer to port 8993 on the ddf container
* set the https frontend to only use the traefik https frontend: this is only necessary when exposing multiple ports/frontends for a single service
* define a frontend called `http` and set routing rule for hostname matching: note that this is the same exact routing rule as the https frontend that we defined before
* set the http frontend to use the http protocol: connections between the proxy and backend will _not_ use tls in this case
* set the http frontend to forward connections to port 8181 on the ddf container
* set the http frontend to only use the traefik http frontend: this is only necessary when exposing multiple ports/frontends for a single service

### Resources

For best placement on swarm nodes always add resources to services in stack files:

```yaml
services:
  ddf1:
    deploy:
      resources:
        limits:
          memory: 4g
          cpus: '2.0'
        reservations:
          memory: 3g
          cpus: '1.0'
```

* limits set upper boundaries
* reservations set minimum requirements
  * unlike limits this will determine what node has enough resources available when swarm deploys to a worker node

### Networks

Use multiple networks

```yaml
services:
  ddf1:
    networks:
      - proxy
      - default
      - ddf1-solr

  ddf1-solr:
    networks:
      - ddf1-solr 

networks:
  default:
  ddf1-solr:
  proxy:
    external: true
```

* define networks used in the overall stack in the top-level networks field
* the `proxy` network is global to the swarm cluster we will be using, it should be used on any service that is utilizing traefik annotations
* the `ddf1-solr` network is common to both the ddf and solr service, only they can talk to each other
* the default network is common to both ddf nodes (this would be used for federation between nodes)

### Variable Interpolation

In order to make the stack flexible environment variables are used to allow certain things to be overridden at runtime.
Specifically any hostname for an exposed service uses two environment variables in order to allow overriding the domain name and optionally adding a suffix to the hostname.

Why a suffix? Mostly so that multiple instances of the same test environment could be deployed at the same time and still create unique routing rules in the loadbalancer layer.

Variable references in compose files take the format `${SOME_VAR}`. To set a default value use the format `${SOME_VAR:-defaultValue}`

So for example the `hostname: ddf1${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}` variable can be read as `hostname: ddf1.test` if neither `DEPLOY_HOSTNAME_SUFFIX` or `DEPLOY_DOMAIN` are set.

#### Hacks

docker stack does not actually interpolate environment variables, but docker compose does. Due to this when we deploy we will actually use `docker-compose` to render the config. For example:

`docker-compose -f <stack-file-path> config`

This can be combined with the command for deploying a stack as follows: 

`docker stack deploy -c <(docker-compose -f <stack-file-path> config) <stack name>`

This will take the output of the `docker-compsoe config` command and pass it into the `docker stack deploy` command. the `-c` option can either take a path to a yaml file *or* read yaml from `stdin`


#### DDF Workarounds

Currently there is an issue with the ddf docker images that requires having `tty: true` added to each ddf service in the docker stack

For example: 
```yaml
services:
    ddf1:
      ...
      tty: true
      ...
    ddf2:
      ...
      tty: true
      ...
```

without this, errors occur during the startup process for ddf

## The compose file

Originally when this example was being written the tests were run via the `docker container run` command, but this started to get unwieldy as the number of arguments started to grow it made more sense to use a [compose](docker-compose.yml) file

```yaml
services:
  test:
    image: someTestImage
    extra_hosts:
      - "ddf1${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}:${SWARM_LB_IP:-127.0.0.1}"
      - "ddf2${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}:${SWARM_LB_IP:-127.0.0.1}"
    environment:
      SUT1_HOST: ddf1${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}
      SUT2_HOST: ddf2${DEPLOY_HOSTNAME_SUFFIX}.${DEPLOY_DOMAIN:-test}
```

### Important Bits:

The extra hosts field:

The loadbalancer uses dns names to map clients to backend services. This means we need to have those dns names resolve inside the test container, but it would be more inconvenient to only be able to use dns names that _actually_ existed in a dns server. We would have to know up front all dns names that might ever get used in a test. Instead we can pass in mappings for hostnames to ip addresses.

By default we are using `127.0.0.1` as the ip address in this file, but we will override this at deploy time

The environment variables:

This is just an example of one way to pass in dns names for the test to connect to, it all depends on the tool being used to test things. if the tool uses command line arguments instead of env vars use the [`command:`](https://docs.docker.com/compose/compose-file/#command) instead.

### Requirements

Any test container used should ensure that it waits for any hosts that it depends on are ready before running the tests. If this requires a custom entrypoint to poll a service for readiness be sure the test container image has this built in. A good example of this is from the [saml-conformance](https://github.com/codice/saml-conformance/blob/master/deployment/docker/wait.sh) project

## The Pipeline

The final part that ties all of this together is the [pipeline](Jenkinsfile).

### Jenkins Agent

Pick an appropriately sized jenkins agent via: 
```groovy
pipeline {
  agent { label 'linux-small' }
}
```

Remember that the stack will not be running local to the jenkins agent but on the swarm. Only the tests will be running on the jenkins agent.

### Global Environment Variables

There are some environment variables we will want to set globally in the pipeline

```groovy
pipeline {
  ...
  environment {
    DEPLOY_DOMAIN = "test.phx.connexta.com"
    DEPLOY_HOSTNAME_SUFFIX = "-${env.BUILD_TAG}"
  }
}
```

* the `DEPLOY_DOMAIN` corresponds to the env var used in the stack file. Here we set it to `test.phx.connexta.com` since the loadbalancer has a wildcard certificate for this domain
* the `DEPLOY_HOSTNAME_SUFFIX`: we add a suffix to the default hostnames in order to prevent clashes if multiple pipeline runs occur
  * this allows multiple branches to be tested at the same time since the `${env.BUILD_TAG}` variable will expand to something like jenkins-<pipeline-name>-<branch-name>-<run-number>
  * for example if the pipeline was called saml-conformance and we were running a test on master we would end up with hostnames like: ddf1-jenkins-saml-conformance-master-1.test.phx.connexta.com and ddf2-jenkins-saml-conformance-master-1.test.phx.connexta.com
  
### Deploying the test environment

The first stage in the pipeline will deploy the stack to the remote swarm cluster

```groovy
pipeline {
  stages {
    stage('Deploy Test Environment') {
      environment {
        DOCKER_HOST = "tcp://swarm-manager1.phx.connexta.com:2375"
      }
      steps {
        sh 'docker stack deploy -c <(docker-compose -f stack.yml config) ${env.BUILD_TAG}'
      }
    }
  }
}
```

The above stage sets an environment variable for communicating with the remote swarm. we have it scoped to this stage so that later stages that use docker will not run things on the remote swarm

The step in this stage uses the `docker-compose config` hack mentioned earlier and deploys a stack using the value of `${env.BUILD_TAG}` as the stack name. This ensures it doesn't clash with other deployments from other runs of the pipeline

### Running the tests

The second stage in the pipeline will execute the tests (local to the jenkins agent)

```groovy
@Library('github.com/connexta/cx-pipeline-library@master') _
pipeline {
  stages {
    stage('Run Tests') {
      steps {
        dockerd {}
        
        sh 'SWARM_LB_IP=$(getent hosts swarm-modern1.phx.connexta.com | awk \'{ print $1 }\') docker-compose up --abort-on-container-exit'
      }
      post {
        always {
          sh 'docker-compose down'
        }
      }
    }
  }
}
```

Breakdown of this stage:
* starts a local docker daemon (uses the cx-pipeline-library for this)
* runs the tests with docker-compose (we inherit the global env vars from above)
* the `SWARM_LB_IP` part uses `getent` to find the ip address of the swarm manager. this is where the swarm loadbalancer is running so any host to ip mappings need to point at this ip
  * this is used by the `extra_hosts:` field in the [compose](docker-compose.yml) file
* the `--abort-on-container-exit` option allows us to run compose in the foreground as a blocking process and exit after the process terminates
  * this makes it work similarly to running a regular command
  * if the tests fail (and properly return exit codes) the step will inherit the pass/fail status
  * this will cause test logs to stream to the jenkins step output
* the post steps ensure that after the tests exit the environment gets cleaned up

#### Cleanup

Finally some cleanup after everything is done

```groovy
pipeline {
  post {
    always {
      sh 'docker -H tcp://swarm-manager1.phx.connexta.com:2375 rm ${env.BUILD_TAG}'
    }
  }
}
```

This removes the stack from the remote swarm after the pipeline completes

## Things not covered in this example

* Deploying stacks that reference private images
  * this will require credentials for a docker registry
  * see [`docker stack deploy --with-registry-auth`](https://docs.docker.com/engine/reference/commandline/stack_deploy/)
  * see [jenkinsfile credentials](https://jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials)
* passing in config files
  * unlike compose, volumes are not a good option here, see [configs](), and [secrets]() instead
* automatic federation: see [ddf-base](https://github.com/oconnormi/docker-ddf-base/blob/master/2.14/linux/entrypoint/sources.sh)
  * could also just pass in static configs like mentioned above