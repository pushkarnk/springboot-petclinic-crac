## Checkpointing and restoring SpringBoot PetClinic using OpenJDK CRaC

The following are steps to demonstrate checkpointing and restoring of the SpringBoot PetClinic application. The Oracular Oriole (24.10) release of Ubuntu comes with new OpenJDK CRaC packages ([openjdk-17-crac](https://launchpad.net/ubuntu/+source/openjdk-17-crac) and [openjdk-21-crac](https://launchpad.net/ubuntu/+source/openjdk-21-crac). 

However, the restore step for the PetClinic application causes a CRIU crash on Ubuntu 24.10. This issue is under investigation. As a result, we execute this demo on Ubuntu 24.04. The OpenJDK CRaC packages are published to [this](https://launchpad.net/~pushkarnk/+archive/ubuntu/openjdk-crac-noble) PPA.

### Step 1

Configure the PPA containing openjdk-17-crac and crac-criu for Noble

```
sudo add-apt-repository ppa:pushkarnk/openjdk-crac-noble
sudo apt update
```

### Step 2

Install openjdk-17-crac

```
sudo apt install openjdk-17-crac-jdk-headless
```
If this is not the only openjdk / Java installation in your environment, please configure `java` and `javac` to point to binaries in /usr/lib/jvm/java-17-openjdk-crac-{arch}/bin/ using `update-alternatives`.

### Step 3

Clone and build the SpringBoot PetClinic application

```
git clone https://github.com/spring-projects/spring-petclinic && cd spring-petclinic
```

SpringBoot 3.2 uses the org.crac dependency to provide a smooth transition to using `jdk.crac` and `javax.crac`, falling back to a dummy implementation if a CRaC implementation is not found. Add this dependency in the pom.xml just under the comment "Spring and SprintBoot dependencies:

```
<dependency>
    <groupId>org.crac</groupId>
    <artifactId>crac</artifactId>
    <version>1.4.0</version>
</dependency>
```

### Step 4

Build the project using the Maven wrapper.

```
./mvnw package
```
This must produce a spring-petclinic-<springboot-version>-SNAPSHOT.jar in the `target` directory. In my setup spring-petclinic-3.3.0-SNAPSHOT.jar was created.

### Step 5

Next, lets launch the application. It must take 7-10 seconds to launch (and consequently be available on localhost:8080 through a browser). We add an additional JVM option named `-XX:CRaCCheckpointTo` to let the JVM know that we might want to checkpoint this application. The value of `-XX:CRaCCheckpointTo` is a valid location where the process image will be dumped.

```
java -XX:CRaCCheckpointTo=cr-data -jar ./target/spring-petclinic-3.3.0-SNAPSHOT.jar
```

A startup time of 7-10s is not promising in the cloud-native world.


### Step 6

It is now time to checkpoint the PetClinic application. In a new terminal window, enter this `jcmd` command:

```
jcmd spring-petclinic JDK.checkpoint
```

This command will kill the PetClinic application and produce a snapshot of the process in the `cr-data` directory in the current working directory.

### Step 7

Finally, lets try to restore the PetClinic app using checkpoint data in the `cr-data` directory.

```
java -XX:CRaCRestoreFrom=cr-data
```

You must see PetClinic coming up in much lesser than a second!

