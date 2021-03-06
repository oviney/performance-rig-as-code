# Performance Testing Infrastructure as Code
An example of how to build performance test infra on demand.

In this workshop, we'll explore how to use a popular open source performance testing solution JMeter + InfluxDB + Grafana combo, with JMeter executing the performance tests, InfluxDB storing the test results and Grafana visualizing the results.

# Problem Statement
While JMeter + InfluxDB + Grafana is already a widely-used performance testing suite, preparing the test setup is a complicated and time-consuming process. In order to load test an app (Web, Mobile or Microservice) and see the results displayed in Grafana, we need to:

- install JMeter, Grafana, InfluxDB and run all three services
- write a JMeter test plan to be executed during testing
- add a JMeter backend listener to send real-time test results to InfluxDB
- configure JMeter with test properties including the test duration, ramp up time, number of threads, as well as the url of the webpage to be tested
- create a database in InfluxDB to store JMeter results
- add an InfluxDB datasource to Grafana and specify the host and port of the running InfluxDB service
- create a dashboard in Grafana and query the InfluxDB data to visualize the JMeter test results

In order to automate this tedious and error-prone process, we will implemented a solution that automates the entire setup, so that users who want to use the JMeter + InfluxDB + Grafana setup to conduct performance testing can do so with a single command.  The target use case for this workshop is to demonstrate how to use this setup in-sprint to provide component performance testing.

If you are new to component performance testing, I would suggest reading the following reference links:

- https://martinfowler.com/articles/microservice-testing/ (This one is my favorite)
- https://dzone.com/articles/improving-app-performance-with-isolated-component
- https://smartbear.com/blog/test-and-monitor/7-types-of-web-performance-tests-and-how-they-fit/
- https://stackify.com/ultimate-guide-performance-testing-and-software-testing/

Simply put, a component test limits the scope of the exercised software to a portion of the system under test, manipulating the system through internal code interfaces and using test doubles to isolate the code under test from other components (Ref: https://martinfowler.com/articles/microservice-testing/#testing-component-introduction).

# Solution
The solution is to run the entire test setup as a multi-container Docker application, managed by Docker Compose, with three separate containers running JMeter, Grafana and InfluxDB respectively.

Some constraints that we consider while designing and implementing this solution are:
- the solution must allow users to specify the url of the app they want to put load on
- the solution should be lightweight and resource-efficient
- the solution should be intuitive and user-friendly

# Technologies

This section gives a brief overview of the technologies used in this solution. In a nutshell, JMeter runs the load tests, InfluxDB stores the test results, Grafana displays the test results, and all of these services are run in Docker.

## JMeter
Apache JMeter is an open-source performance testing software application. JMeter conducts its testing by simulating a group of users sending requests to a target server (web application, microservice(s), database, etc.). This is the software we use to put load on an app.

## InfluxDB
InfluxDB is a time series database that is mainly used to store monitoring data, metrics, real-time analytics and other data that is dependent on time. InfluxDB provides the connection between JMeter and Grafana; InfluxDB listens to JMeter and stores the test results in a database, and Grafana reads the data from the InfluxDB database to display in its graphs.

## Grafana
Grafana is an open-source real-time data visualization tool. It provides a webpage where users can create, view and interact with graphs and dashboards displaying the data from a datasource (in this case InfluxDB). This is the software we use to gain visual feedback on the test results.

## Docker
Docker is an open-source software that runs software packages called containers, which gather all the software and dependencies together. The basic Docker workflow is:
- we write a Dockerfile in which we define our environment and install any dependencies
- we build a Docker image from the Dockerfile
- we run the Docker image to start a Docker container, which starts the service

In our case, JMeter, Grafana and InfluxDB all run in separate Docker containers which talk to each other.

# Implementation
This section will outline the various engineering decisions made.

## Why Docker

**Reproducibility:** running a set of Docker containers is guaranteed to produce the same results every time with no possibility of random error. This is very important in testing, since tests are meant to be run repeatedly with controlled variability.

**Installed dependencies:** this performance testing solution requires many things to be installed, key among them JMeter, InfluxDB and Grafana software. Docker containers, which package all the software and dependencies together, seem to be a good fit.

## Why a multi-container Docker solution?
Splitting out the various applications into individual containers is the recommended approach, as opposed to running multiple services in a single Docker container.  The multi-container approach makes it much easier to update/scale individual components.

**Scalability:** the multi-container approach allows us to define a different number of containers for each service, something that cannot be easily accomplished if the services are all running inside a single container.  The multi-container approach allows you to isolate when updating or restarting a service, rather than having to restart the entire application / container.

## Why use Docker Compose?
Docker Compose is a tool used to configure the services in a multi-container Docker application. When using Docker Compose, we still need Dockerfiles for each of the individual containers, and on top of that we also need to write a Compose file in which we configure all of the application’s services, for instance specifying the port-forwarding, build arguments and environment variables of each service. 

Using Docker Compose to manage the containers instead of managing all the containers manually has many advantages, including:
- ability to build, start and stop all the services of the multi-container application with one command, instead of manually starting/stopping each container
- ability able to configure all of the application’s services in one place, via the Docker Compose file. This provides a better overview of the entire application and how each service is configured.  Also, thanks to the Docker Compose file we don't need to pass these configurations in as runtime arguments each time we want to run the service / app / containers, which is more tedious and more error-prone

## Sample docker-compose.yml
```shell
version: '3'
  services:        
    influxdb:                
      image: influxdb:1.7.9-alpine                
      environment:                        
        - INFLUXDB_DB=jmeter        
    jmeter:                
      build:                         
        context: "jmeter"                        
        args:                                
          - jmeterVersion=5.2.1                
      environment:                
        - JMETER_TEST=performance_test_plan.jmx             
      depends_on:                        
        - influxdb        
    grafana:                
      build:                        
        context: "grafana"                
      ports:                        
        - "3000:3000"                
      depends_on:                        
        - influxdb                        
        - jmeter
```

- Docker Compose creates a single network for all of the application’s service’s containers by default, and containers are able to discover other containers by their container name and talk to other containers via this network. This is extremely convenient in this use case because the JMeter container needs to talk to the InfluxDB container to pass test execution result data to the database and Grafana needs to talk to the InfluxDB container to retrieve test results to display. 
- With the default network created by Docker Compose, we will be able to configure JMeter to send data to InfluxDB by adding a InfluxDBBackendListenerClient to JMeter and specifying the InfluxDB URL as http://influxdb:8086/write?db=jmeter
- and configure Grafana to retrieve data from InfluxDB by adding an InfluxDB datasource with URL: http://influxdb:8086. Note that we don’t need to know what the ip address of the InfluxDB container is, and can directly send requests to http://influxdb, since by default Docker Compose creates each container with a hostname identical to their container name. This is great news, we can just use the default network created by Docker Compose instead of having to play around with Docker networking

In the next section, we will demonstrate how to build and run this application with and with Docker Compose

*With Docker Compose:*
- *docker-compose up* to build and run all the services
- *docker-compose down* to stop all the services

Docker Compose is the option that best satisfies the our requirements, simple commands, the Docker Compose file and default network creation functionality also makes it easier for us to work with.

## How do we choose the version of the images?
Most Dockerfiles define a parent image.  The final image that is built from the Dockerfile is the parent image after it has been modified by the instructions in the Dockerfile. There are many images hosted on Docker Hub that can be used as parent images to build on top of.
For InfluxDB and Grafana, we should chose the official InfluxDB and Grafana Docker images, so that we won't need to install these in the Dockerfile, and since official images are generally better maintained and more secure. 

We should chose the InfluxDB image tagged with alpine in particular because Alpine image variations are more lightweight, and the influxdb:alpine image provided all the functionality we will needed.

There is no official JMeter image on Docker Hub, so we will chose Alpine Linux as the base image, in order to start with the smallest image possible, and manually installed JMeter in the Dockerfile.

## Performance Tester Configuration
There are typically many tunables that a performance tester might want to configure when running tests.  The most common examples are URL, number of concurrent users, test duration.  

Sample user.properties
```shell
# time it takes for jmeter to start all the threads in seconds
rampUp=0
# total number of threads to be executed
threads=5
# duration of the running test (in seconds)
duration=360 
# url of the web app to put load on
webUrl=viney.ca
```

example line in default-test-plan.jmx file with JMeter property function reading value of duration from user.properties:
```shell
<stringProp name="ThreadGroup.duration">${__P(duration,1)}</stringProp>
```

## How to use the rig (MVP)

This is meant to be simple.  In order to try this setup for yourself, follow the steps below.  You can also check out the following demo of the steps below:  https://youtu.be/r5v8jYnwozw

* git clone - pull down the source
* docker-compose up - run it

Let's elaborate on each step in the next section.

### Give it a try

```shell
git clone https://github.com/oviney/performance-rig-as-code.git
```

What am I running?  The test is a basic HTTP test, hitting the endpoint as configured in the user.properties file (jmeter/test/).  The test will run using the properties defined in this file.  
```shell
docker-compose up
```

There should be load sent to the URL you specified, you should be able to view the results in Grafana at:
```shell
http://localhost:3000
```

You should see something like the following.

![Sample Grafana JMeter Dashboard](/images/jmeter-grafana-dashboard.png)

### Extending & Scaling it

This solution is also configurable.  You can use this setup to execute your own JMeter test plan(S) by copying test-plan.jmx into jmeter/test/ and replacing the value of the JMETER_TEST environment variable, which should be *‘default-test-plan.jmx’*, with the name of your new test (*‘your-test-plan.jmx’* in this example) in *docker-compose.yml*.

You will notice that the default report is modeled to give you what LoadRunner Controller by default provides.  This is a good standard to start with, but you can easily add any additional metrics.  Just checkout the Grafana docs, and then export your changes (JSON) to grafana/dashboards/.  This is the approach that I followed to tweak the dashboard that I downloaded from https://grafana.com/grafana/dashboards?search=jmeter.

# Reference Links
- Useful article about JMeter and Docker from Blazemeter - https://www.blazemeter.com/blog/make-use-of-docker-with-jmeter-learn-how/
- Useful article about Docker Containers options for JMeter Users - https://www.blazemeter.com/blog/top-6-docker-images-for-jmeter-users-and-performance-testers/
- How to Compose the Most Effective Apache JMeter Test Plan - https://dzone.com/articles/tips-and-tricks-to-compose-the-most-effective-apac
- Various types of Performance Tests and where to apply them - https://smartbear.com/blog/test-and-monitor/7-types-of-web-performance-tests-and-how-they-fit/
- https://dzone.com/articles/improving-app-performance-with-isolated-component
- https://ieeexplore.ieee.org/document/7302490
- https://martinfowler.com/articles/microservice-testing/#testing-component-introduction
- Learn JMeter - https://www.blazemeter.com/blog/learn-jmeter-for-free-launching-the-jmeter-academy/
- JMeter Academy - https://www.blazemeter.com/blog/jmeter-academy-advanced-jmeter-course-now-open/
- JMeter Training - https://www.blazemeter.com/jmeter-training/ 
- Pre-built Grafana dashboards for JMeter - https://grafana.com/grafana/dashboards/1152