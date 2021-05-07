   Apache Spark Learning Note: Day02 - Construct an Apache Spark Cluster using docker
========================================================================================

<br/>

### System Requirements     

- Docker 1.13.0+,  
- Docker Compose 3.0+   


From Day 01 of this learning note series, you should understand why we should learn Apache Spark (Because we would like to use this framework to execute parallel computing tasks in distributed systems using Python, Scala, and R). To get started, it is necessary to have a testbed on our machine. Thus, in today's material, I will start to introduce how to construct this distribution system based on Docker.  

What we will do today including...    
1. Apache Spark with one master and two worker nodes,    
2. JupyterLab.   
3. Simulated HDFS.   
4. Python and PySpark and Java 8.  

<br/>  

## Cluster Overview  

The cluster is composed of four main components: the JupyterLab, the Spark master node and two Spark workers nodes. The user connects to the master node and submits Spark commands through the JupyterLab notebooks. The master node then processes the input and distributes the computing workload to workers nodes, and sending back the results to the JupyterLab interface. The components are connected using a localhost network and share data among each other thru a shared mounted volume that simulates an HDFS.  
![Concept Images][https://www.kdnuggets.com/wp-content/uploads/perez-spark-docker-1.png]     

In other words, we need to create, build and compose the Docker images for JupyterLab and Spark nodes. The cluster base image will download and install common software tools, i.e. Python, and will create the shared directory for the HDFS. Spark notes will be integrated into a Spark base image.   

<br/>

## Docker Images  

Here I am introducing the docker images for each component.  
If you don't know dockerfile well, I will recommend that you should go google it or go to my docker learning note to get basic knowledge.  


- cluster-base.Dockerfile  
```dockerfile
#-- Choose docker images: small Debian image with a built-in Java 8 runtime environment (JRE)
ARG debian_buster_image_tag=8-jre-slim
FROM openjdk:${debian_buster_image_tag}

#-- Layer: OS + Python 3.7
#-- Create the shared volume
ARG shared_workspace=/opt/workspace

RUN mkdir -p ${shared_workspace} && \
    apt-get update -y && \
    apt-get install -y python3 && \
    ln -s /usr/bin/python3 /usr/bin/python && \
    rm -rf /var/lib/apt/lists/*

ENV SHARED_WORKSPACE=${shared_workspace}

# -- Runtime

VOLUME ${shared_workspace}
CMD ["bash"]
```  
    
- spark-base.Dockerfile.Dockerfile
``` dockerfile
FROM cluster-base

#-- Layer: Apache Spark

ARG spark_version=3.0.0
ARG hadoop_version=2.7

RUN apt-get update -y && \
    apt-get install -y curl && \
    curl https://archive.apache.org/dist/spark/spark-${spark_version}/spark-${spark_version}-bin-hadoop${hadoop_version}.tgz -o spark.tgz && \
    tar -xf spark.tgz && \
    mv spark-${spark_version}-bin-hadoop${hadoop_version} /usr/bin/ && \
    mkdir /usr/bin/spark-${spark_version}-bin-hadoop${hadoop_version}/logs && \
    rm spark.tgz

#-- Apache Spark location used by the framework for setting up tasks  
ENV SPARK_HOME /usr/bin/spark-${spark_version}-bin-hadoop${hadoop_version}

#-- the master node hostname used by worker nodes to connect  
ENV SPARK_MASTER_HOST spark-master

#-- the master node port used by worker nodes to connect  
ENV SPARK_MASTER_PORT 7077

#-- the Python location that is used by Apache Spark to support its Python API
ENV PYSPARK_PYTHON python3

# -- Runtime

WORKDIR ${SPARK_HOME}
```

- spark-master.Dockerfile
``` dockerfile
FROM spark-base

#-- Runtime
#-- the spark_master_web_ui port is for letting us access the master web UI page
#-- the SPARK_MASTER_PORT is to allow workers to connect to the master node.
ARG spark_master_web_ui=8080

#-- run Spark built-in deploy script with the master class as its argument
EXPOSE ${spark_master_web_ui} ${SPARK_MASTER_PORT}
CMD bin/spark-class org.apache.spark.deploy.master.Master >> logs/spark-master.out
```

- spark-worker.Dockerfile
``` dockerfile
FROM spark-base

#-- Runtime
#-- the spark_master_web_ui port is for letting us access the master web UI page
ARG spark_worker_web_ui=8081

#-- run Spark built-in deploy script with the worker class and the master network address as its arguments
#-- this will make workers nodes connect to the master node on its startup process.
EXPOSE ${spark_worker_web_ui}
CMD bin/spark-class org.apache.spark.deploy.worker.Worker spark://${SPARK_MASTER_HOST}:${SPARK_MASTER_PORT} >> logs/spark-worker.out

```

- jupyterlab.Dockerfile
``` dockerfile
FROM cluster-base

#-- Layer: JupyterLab

ARG spark_version=3.0.0
ARG jupyterlab_version=2.1.5

RUN apt-get update -y && \
    apt-get install -y python3-pip && \
    pip3 install wget pyspark==${spark_version} jupyterlab==${jupyterlab_version} \
    netCDF4==${netCDF4_version} xarray==${xarray_version} scipy==${scipy_version} && \
    pip3 install "dask[complete]" && \
    git clone git clone https://github.com/andersy005/spark-xarray.git && \
    cd spark-xarray && python setup.py install

# -- Runtime

EXPOSE 8888
WORKDIR ${SHARED_WORKSPACE}
CMD jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root --NotebookApp.token=
```

Note that since we use Docker *ARG* on Dockerfiles to specify software versions, it is easily change the default Apache Spark and JupyterLab versions in the future.   


<br/>  

## Building Docker images

Now, if you finished the inputs, the Docker images are ready. Let's build them up.

- build.sh
```bash

#-- Software Stack Version
SPARK_VERSION="3.0.0"
HADOOP_VERSION="2.7"
JUPYTERLAB_VERSION="2.1.5"

#-- Building Images

docker build \
  -f cluster-base.Dockerfile \
  -t cluster-base .

docker build \
  --build-arg spark_version="${SPARK_VERSION}" \
  --build-arg hadoop_version="${HADOOP_VERSION}" \
  -f spark-base.Dockerfile \
  -t spark-base .

docker build \
  -f spark-master.Dockerfile \
  -t spark-master .

docker build \
  -f spark-worker.Dockerfile \
  -t spark-worker .

docker build \
  --build-arg spark_version="${SPARK_VERSION}" \
  --build-arg jupyterlab_version="${JUPYTERLAB_VERSION}" \
  -f jupyterlab.Dockerfile \
  -t jupyterlab .
``` 


<br/>  

## Composing the cluster   

Now, we can use **docker-compose** for our cluster. Again, you need to know the basic knowledge of Docker. Here, we will create the JuyterLab and Spark nodes containers, expose their ports for the localhost network and connect them to the simulated HDFS.   

- docker-compose.yml
```yml
version: "3.6"
volumes:
  shared-workspace:
    name: "hadoop-distributed-file-system"
    driver: local
    driver_opts:
      o: bind
      type: none 
      device: /Users/cyhsu/dev/git-blog/LearningNote/ApacheSpark/Volumes
services:
  jupyterlab:
    image: jupyterlab
    container_name: jupyterlab
    ports:
      - 8888:8888
    volumes:
      - shared-workspace:/opt/workspace
  spark-master:
    image: spark-master
    container_name: spark-master
    ports:
      - 8080:8080
      - 7077:7077
    volumes:
      - shared-workspace:/opt/workspace
  spark-worker-1:
    image: spark-worker
    container_name: spark-worker-1
    environment:
      - SPARK_WORKER_CORES=1
      - SPARK_WORKER_MEMORY=512m
    ports:
      - 8081:8081
    volumes:
      - shared-workspace:/opt/workspace
    depends_on:
      - spark-master
  spark-worker-2:
    image: spark-worker
    container_name: spark-worker-2
    environment:
      - SPARK_WORKER_CORES=1
      - SPARK_WORKER_MEMORY=512m
    ports:
      - 8082:8081
    volumes:
      - shared-workspace:/opt/workspace
    depends_on:
      - spark-master
```


In the docker-compose.yml, we first start creating the Docker volume for the simulated HDFS (hadoop-distributed-file-system). After, we create one container for each cluster component. The JupyterLab container exposes its port on 8888 and binds the shared-workspace directory to the HDFS volume. Next, the spark-master container exposes its web UI port to 8080,  its master-worker connection port to 7077, and also binds to the HDFS volume. Two Spark worker containers named spark-worker-1 and spark-worker-2. Each container exposes its web UI port (mapped at 8081 and 8082 respectively) and binds to the HDFS volume. These containers have an environment step that specifies their hardware allocation:    

**SPARK_WORKER_CORE is the number of cores**    
**SPARK_WORKER_MEMORY is the amount of RAM**    

By default, we are selecting one core and 512 MB of RAM for each container, but feel free to play with the hardware allocation; however, make sure to respect your machine limits to avoid memory issues. To the end, make sure you have provided enough resources for your Docker application to handle the selected values. Oh, right, don't forgot to ** docker-compose up -d ** to complete the docker building mission.   

Once finished, check out 
- JupyterLab at localhost:8888  
- Spark-master at localhost:8080  
- Spark-worker-1 at localhost:8081  
- Spark-worker-2 at localhost:8082  
