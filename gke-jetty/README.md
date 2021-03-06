gke-jetty
=========

This project builds a Docker Image for 
Google App Engine [Java Managed VM](https://cloud.google.com/appengine/docs/managed-vms/)
that provides the Jetty 9.3 Servlet container on top of the gke-debian-openjdk8 image.

The layout of this image is intended to mostly mimic the official 
[docker-jetty](https://github.com/appropriate/docker-jetty) image and unless otherwise noted,
the official [docker-jetty documentation](https://github.com/docker-library/docs/tree/master/jetty)
should apply.

## Building the Jetty image
To build the image you need git, docker and maven installed and to have the gke-debian-openjdk:8-jre
image available in your docker repository:
```console
git clone https://github.com/GoogleCloudPlatform/appengine-java-vm-runtime.git
cd appengine-java-vm-runtime/gke-jetty
mvn clean install
```

## Running the Jetty image
The resulting image is called gke-jetty:9.3.5.v20151012 (or the current jetty version as the label) 
and can be run with:
```console
docker run gke-jetty:9.3.5.v20151012
```

## Configuring the Jetty image
Arguments passed to the docker run command are passed to Jetty, so the 
configuration of the jetty server can be seen with a command like:
```console
docker run gke-jetty:9.3.5.v20151012 --list-config
```

Alternate commands can also be passed to the docker run command, so the
image can be explored with 
```console
docker run t --rm gke-jetty:9.3.5.v20151012 bash
```

To update the server configuration in a derived Docker image, the `Dockerfile` may
enable additional modules with `RUN` commands like:
```
WORKDIR $JETTY_BASE
RUN java -jar "$JETTY_HOME/start.jar" --add-to-startd=jmx,stats
```
Modules may be configured in a `Dockerfile` by editing the properties in the corresponding `/var/lib/jetty/start.d/*.mod` file or the module can be deactivated by removing that file.

## GAE Managed VMs
This image works with App Engine Managed VMs as a custom runtime.
In order to use it, you need to build the image (let's call it `YOUR_BUILT_IMAGE`), (and optionally push it to a Docker registery like gcr.io). Then, you can add to any pure Java EE 7 Web Application projects these 2 configuration files next to the exploded WAR directory:

`Dockerfile` file would be:
      
      FROM YOUR_BUILT_IMAGE
      add . /app
      
That will add the Web App Archive directory in the correct location for the Docker container.

Then, an `app.yaml` file to configure the GAE Managed VM product:

      runtime: custom
      vm: true
      api_version: 1
      
Once you have this configuration, you can use the Google Cloud SDK to deploy this directory containing the 2 configuration files and the Web App directory using:

     gcloud app deploy app.yaml
     

## Entry Point Features
The entry point for the image is [docker-entrypoint.bash](https://github.com/GoogleCloudPlatform/appengine-java-vm-runtime/blob/master/gke-jetty/src/main/docker/docker-entrypoint.bash), which does the processing of the passed command line arguments to look for an executable alternative or arguments to the default command (java).

If the default command (java) is used, then the entry point sources the [gke-env.bash](https://github.com/GoogleCloudPlatform/appengine-java-vm-runtime/blob/master/gke-debian-openjdk/src/main/docker/gke-env.bash), which looks for supported features: ALPN, Stackdriver Debugger & Cloud Profiler.  Each of these features must be explicitly enabled and not disable by environment variables, and each has a script that is run to determine the required JVM arguments:

| Feature              | directory    | Enable            | Disable        | JVM args      |
|----------------------|--------------|-------------------|----------------|---------------|
| ALPN                 | /opt/alpn/   | $ALPN_ENABLE      | $ALPN_DISABLE  | $ALPN_BOOT    |
| Stackdriver Debugger | /opt/cdbg/   | \<on by default\> | $CDBG_DISABLE  | $DBG_AGENT    |
| Cloud Profile        | /opt/cprof/  | $CPROF_ENABLE     | $CPROF_DISABLE | $PROF_AGENT   |
| Temporary file       |              | $TMPDIR           |                | $SET_TMP      |
| Java options         |              | $JAVA_OPTS        |                | $JAVA_OPTS    |

The command line executed is effectively (where $@ are the args passed into the 
docker entry point):
```
java $ALPN_BOOT \
     $DBG_AGENT \
     $PROF_AGENT \
     $SET_TMP \
     $JAVA_OPTS \
     -Djetty.base=$JETTY_BASE -jar $JETTY_HOME/start.jar \
     "$@"
```





Enjoy...
