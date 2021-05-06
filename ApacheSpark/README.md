   Apache Spark Learning Note: Introduction
==============================================

   ### Introduction

   When I was looking for the data engineer job position, I have been asked whether I have experience in Apache Spark. However, in the field of earth science, it is hard to find applications to use it. The biggest two reasons are 1. We are using the "NetCDF/HDF" data format (now, maybe Zarr) to store our data or model outputs, and the science field is usually updated slowly. Recently, if we really want to use Apache Spark + netCDF data, we can actually find two python packages, "spark-xarray" and "sci-spark." In this material, I will focus on the spark-xarray because xarray is a common library now in earth science to work on our datasets. Spark-array is an open-source project and is an integrated python library for working with netCDF climate model data with Apache Spark, developed in 2017 summer. You can find more detail  here **(Spark Apache)[https://ncar.github.io/PySpark4Climate/sparkxarray/overview/]**.     

   I personally think **Spark + netCDF** is a great approach, and it is the right route to go in the near future. Although we already have Python Libraries, such as Xarray and Dask (lazy function and parallel computing), to deal with Big Data problems. Nowadays, NOAA is promoting the ERDDAP and THREDDS data servers (tomcat webapps). Both of them are good since it is easy to access (kind-of, if you know how) on clouds. To the py-people, you can use pandas to retrieve ERDDAP data, while use xarray to access THREDDS. However, I believe both ERDDAP and THREDDS currently have to read the entire data request into memory before sending it to the client. This slows down the access and becomes a big issue in dealing with real-time problems. Anyhow, efficient storage and data server access are still under the table. In my vision, I believe that all of the data warehouses will eventually ingest their data into the Apache Spark server shortly. The benefits of the Apache Spark is the Spark operates to split datasets across time/space, e.g., a decade-long time-series of geo-variables can be split across time to enable parallel "speedups" of analysis by day, month, or season or partition the dataset into spatial tiles for parallel operations across rows, columns, or blocks.   

   As the data warehouse frontier, data engineer, and data manager at GCOOS, I would like to share my learning notes on this data storage transform problem. It is time to think about how to save data into apache spark and read from it.   

<br/><br/>

   #### 30 Days of Learning Notes    

     - Day 00: Introduction  
     - Day 01: What is Apahe Spark  
     - Day 02: Build up Apache Spark cluster using docker  
     - Day 03: RDD, map operation, flatMap  
     - Day 04: Scala, RDD Implicit Conversion  
     - Day 05: 1st Spark App  
     - Day 06: For expression, Set, SparkSQL UDF by Use Case  
     - Day 07: Broadcast and Spark-submit  
     - Day 08:   
     - Day 09:   
     - Day 10:   
     - Day 11:   
     - Day 12:   
     - Day 13:   
     - Day 14:   
     - Day 15:    
     - Day 16:   
     - Day 17:   
     - Day 18:   
     - Day 19:   
     - Day 20: What is spark-xarray  
     - Day 21: spark-xarray installation  
     - Day 22: Read-In Dataset using spark-xarray: Oceanic Nino Index  
     - Day 23: Read-In Dataset using spark-xarray: Statistical Techniques  
     - Day 24: Read-In Dataset using spark-xarray: PySparkGeoAnalysis  
     - Day 25: Write-Out Dataset using spark-xarray  
     - Day 26:   
     - Day 27:  
     - Day 28:   
     - Day 29:  
     - Day 30:  


   Basic Knowledge Requirements: Python - Xarray, Numpy, Scipy, Dask  
                                 Linux/Unix - Bash Script  
                                 Docker - docker, docker-compose  

