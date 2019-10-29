# sbt-kubeyml
An sbt plugin to generate typesafe kubernetes deployment plans for scala projects

## Usage

### Add the plugin to your plugins.sbt
```
addSbtPlugin("org.vaslabs.kube" % "sbt-kubeyml" % "0.1.9")
```

Add the plugin in your project and enable it
```
enablePlugins(KubeDeploymentPlugin)
```
The plugin depends on a bunch of other plugins (sbt-native-packager)

```
enablePlugins(DockerPlugin)
```

Try to run

```
kubeyml:gen
```

## Properties

| sbt key  | description  | default  | 
|---|---|---|
| namespace  | The kubernetes namespace of the deployment   |  Default value is project name | 
|  application | The name of the deployment  |  Default value is project name  |
|  dockerImage | The docker image to deploy in a single container |  Default is the picked from sbt-native-packager |
| livenessProbe  | Healtcheck probe  | `HttpProbe(HttpGet("/health", 8080, List.empty), 5 seconds, 5 seconds, None)` |
| readinessProbe  |  Probe to check when deployment is ready to receive traffic  | livenessProbe  |
| annotations  | Fre `Map[String, String]` for spec template annotations (e.g. aws roles)  | empty  |
| replicas | the number of replicas to be deployed| 2 |
| envs | Map of environment variables, raw, field path or secret are supported| empty |
| resourceRequests | Resource requests (cpu in the form of m, memory in the form of MiB |  `Resource(Cpu(500), Memory(256))` |
| resourceLimits | Resource limits (cpu in the form of m, memory in the form of MiB |  `Resource(Cpu(1000), Memory(512))` |
| target | The directory to output the deployment.yml | target of this project |

## Recipes

### Single namespace, two types of deployments with secret and dependency

```scala
import kubeyml.deployment.{Cpu, EnvName, EnvRawValue, EnvSecretValue, Memory, Resource}
import kubeyml.deployment.sbt.Keys._

lazy val deploymentName = sys.env.getOrElse("DEPLOYMENT_NAME", "myservice-test")
lazy val secretsName = sys.env.getOrElse("SECRETS_NAME", "myservice-test-secrets")
lazy val serviceDependencyConnection = sys.env.getOrElse("MY_DEPENDENCY", "https://localhost:8080")

lazy val deploymentSettings = Seq(
  namespace in kube := "my-namespace", //default is name in thisProject
  name in kube := deploymentName, //default is name in thisProject
  envs in kube := Map(
    EnvName("JAVA_OPTS") -> EnvRawValue("-Xms256 -Xmx2048M"),
    EnvName("MY_DEPENDENCY_SERVICE") -> EnvRawValue(serviceDependencyConnection),
    EnvName("MY_SECRET_TOKEN") -> EnvSecretValue(name = secretsName, key "my-token")
  ),
  resourceLimits := Resource(Cpu.fromCores(2), Memory(2048+512)),
  resourceRequests := Resource(Cpu(500), Memory(512))
)
```

### Gitlab CI/CD usage (followup from previous)

```yaml
stages:
  - publish-image
  - deploy

.publish-template:
  stage: publish-image
  script:
      - sbt docker:publishLocal
      - sbt kubeyml:gen
  artifacts:
      untracked: true
      paths:
        - target/kubeyml/deployment.yml

.deploy-template:
  stage: deploy
  image: docker-image-that-has-your-kubectl-config
  script:
     - kubectl apply -f target/kubeyml/deployment.yml

publish-test:
  before_script:
      export MY_DEPENDENCY=${MY_TEST_DEPENDENCY}
  extends: .publish-template

deploy-test:
  extends: .deploy-template
  dependencies:
     - publish-test

publish-prod:
  before_script:
    - export MY_DEPENDENCY=${MY_PROD_DEPENDENCY}
    - export SECRETS_NAME=${MY_PROD_SECRET_NAME}
    - export DEPLOYMENT_NAME=my-service-prod
  extends: .publish-template

deploy-prod:
  extends: .deploy-template
  dependencies:
   - publish-prod
```


