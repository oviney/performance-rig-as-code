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
