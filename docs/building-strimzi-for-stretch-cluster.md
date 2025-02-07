# Building Strimzi Kafka Operator for Stretch Clusters  

The Strimzi community is actively transitioning to KRaft, leading to frequent changes in the main branch. As of the time of writing, Strimzi is in the process of removing ZooKeeper-related code. To ensure stability, we have based our development for stretch clusters on Strimzi **0.44.0**.  

The prototype code for stretch clusters is available in the following [branch](https://github.com/aswinayyolath/strimzi-kafka-operator/tree/strecth-cluster-prototype). This branch is derived from the **Strimzi 0.44.0** release tag. To view the changes made on top of Strimzi **0.44.0**, refer to the following comparison [link](https://github.com/aswinayyolath/strimzi-kafka-operator/compare/main...aswinayyolath:strimzi-kafka-operator:strecth-cluster-prototype?expand=1)  

---

## Building the Operator  

### Clone the Repository  

```bash
git clone https://github.com/aswinayyolath/strimzi-kafka-operator.git
```

### Checkout the Stretch Cluster Branch

```bash
git checkout strecth-cluster-prototype
```

### Set Up Docker Registry Variables

Before building the operator, ensure that the `DOCKER_ORG` and `DOCKER_REGISTRY` environment variables are set correctly. These should match your Docker Hub username and the Docker registry you are using. If `DOCKER_REGISTRY` is unset, it defaults to `docker.io`.

For more details, refer to the Strimzi Developer [Guide](https://github.com/strimzi/strimzi-kafka-operator/blob/main/development-docs/DEV_GUIDE.md).

```bash
export DOCKER_ORG=<your_docker_hub_username>
export DOCKER_REGISTRY=<your_docker_registry>  # Defaults to docker.io if unset
```

### Build the Operator
Since this is a prototype, we have not implemented test coverage yet. To avoid test compilation errors during the build process, disable test execution:

```bash
DOCKER_PLATFORM="--platform linux/amd64" make MVN_ARGS='-Dmaven.test.skip=true' all
```

!!! note

    🚨 If you encounter an error related to unused declared dependencies, such as
        `Unused declared dependencies found` You may need to comment out the dependencies mentioned in the error message within the corresponding pom.xml file. 🚨




## Testing the Operator Image Without Building

If you prefer to test the operator without building it from source, you can use the pre-built image available on `Docker Hub`

```bash
aswinayyolath/stretchcluster:latest
```

A detailed guide on testing the stretch cluster will be provided in the later section