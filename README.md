# Performance Testing Infrastructure as Code
An example of how to build performance test infra on demand.

In this workshop, we'll explore how to use a popular open source performance testing solution JMeter + InfluxDB + Grafana combo, with JMeter executing the performance tests, InfluxDB storing the test results and Grafana visualizing the results.

Problem Statement
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
