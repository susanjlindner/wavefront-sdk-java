# wavefront-sdk-java [![travis build status](https://travis-ci.com/wavefrontHQ/wavefront-sdk-java.svg?branch=master)](https://travis-ci.com/wavefrontHQ/wavefront-sdk-java)

Wavefront by VMware SDK for Java is a library that provides support for sending metrics, histograms and opentracing spans from your Java application to Wavefront via WavefrontSender interface.

## Usage
If you are using Maven, add following maven dependency to your pom.xml
```
<dependency>
    <groupId>com.wavefront</groupId>
    <artifactId>wavefront-sdk-java</artifactId>
    <version>0.9.0</version>
</dependency>
```

### WavefrontSender
The WavefrontSender interface has two implementations.<br/>
1) WavefrontProxyClient
2) WavefrontDirectIngestionClient

### WavefrontProxyClient
```java
  /*
   * Assume you have a running Wavefront proxy listening on at least one of 
   * metrics/direct-distribution/tracing ports and you know the proxy hostname
   */
  WavefrontProxyClient.Builder builder = new WavefrontProxyClient.Builder(proxyHost);
 
  /* set this (Example - 2878) if you want to send metrics to Wavefront */
  builder.metricsPort(metricsPort);
 
  /* set this (Example - 40,000) if you want to send histograms to Wavefront */
  builder.distributionPort(distributionPort);
 
  /* set this (Example - 30,000) if you want to send opentracing spans to Wavefront */
  builder.tracingPort(tracingPort);
 
  /* set this if you want to override the default SocketFactory */
  builder.socketFactory(<SocketFactory>);
  
  /* The flushing interval controls how often the WavefrontSender sends data to the Wavefront service.
   * Set this if you want to change the default flush interval of 5 seconds. 
   */
  builder.flushIntervalSeconds(2);
  
  WavefrontProxyClient wavefrontProxyClient = builder.build();
 ```
 
### WavefrontDirectIngestionClient
```java
  /*
   * Assume you have a running Wavefront cluster and you know the 
   * server URL (example - https://mydomain.wavefront.com) and the API token
   */
  WavefrontDirectIngestionClient.Builder builder = 
  new WavefrontDirectIngestionClient.Builder(wavefrontServer, token);
 
  /* set this if you want to change the defualt max queue size of 50,000 */
  builder.maxQueueSize(100_000);
 
  /* set this if you want to change the default batch size of 10,000 */
  builder.batchSize(20_000);
 
  /* The flushing interval controls how often the WavefrontSender sends data to the Wavefront service.
   * Set this if you want to change the default flush interval value of 1 seconds 
   */
  builder.flushIntervalSeconds(2);
   
  WavefrontDirectIngestionClient wavefrontDirectIngestionClient = builder.build();
 ```
 
 ### Send data to Wavefront via WavefrontSender
 
 ```java
  // 1) Send Metric to Wavefront
  /*
   * Wavefront Metrics Data format
   * <metricName> <metricValue> [<timestamp>] source=<source> [pointTags]
   *
   * Example: "new-york.power.usage 42422 1533529977 source=localhost datacenter=dc1"
   */
  wavefrontSender.sendMetric("new-york.power.usage", 42422.0, 1533529977L,
        "localhost", ImmutableMap.<String, String>builder().put("datacenter", "dc1").build());

  // 2) Send Delta Counter to Wavefront     
  /*
   * Wavefront Delta Counter format
   * <metricName> <metricValue> source=<source> [pointTags]
   *
   * Example: "lambda.thumbnail.generate 10 source=lambda_thumbnail_service image-format=jpeg"
   */
   wavefrontSender.sendDeltaCounter("lambda.thumbnail.generate", 10,
        "lambda_thumbnail_service",
        ImmutableMap.<String, String>builder().put("image-format", "jpeg").build());

  // 3) Send Direct Distribution (Histogram) to Wavefront
  /*
   * Wavefront Histogram Data format
   * {!M | !H | !D} [<timestamp>] #<count> <mean> [centroids] <histogramName> source=<source> 
   * [pointTags]
   *
   * Example: You can choose to send to atmost 3 bins - Minute/Hour/Day
   * 1) Send to minute bin    =>    
   *    "!M 1533529977 #20 30.0 #10 5.1 request.latency source=appServer1 region=us-west"
   * 2) Send to hour bin      =>    
   *    "!H 1533529977 #20 30.0 #10 5.1 request.latency source=appServer1 region=us-west"
   * 3) Send to day bin       =>    
   *    "!D 1533529977 #20 30.0 #10 5.1 request.latency source=appServer1 region=us-west"
   */
  wavefrontSender.sendDistribution("request.latency", 
        ImmutableList.<Pair<Double, Integer>>builder().add(new Pair<>(30.0, 20)).
        add(new Pair<>(5.1, 10)).build(),
        ImmutableSet.<HistogramGranularity>builder().add(HistogramGranularity.MINUTE).
            add(HistogramGranularity.HOUR).
            add(HistogramGranularity.DAY).build(), 
        1533529977L, "appServer1",
        ImmutableMap.<String, String>builder().put("region", "us-west").build());

  // 4) Send OpenTracing Span to Wavefront
  /*
   * Wavefront Tracing Span Data format
   * <tracingSpanName> source=<source> [pointTags] <start_millis> <duration_milliseconds>
   *
   * Example: "getAllUsers source=localhost
   *           traceId=7b3bf470-9456-11e8-9eb6-529269fb1459
   *           spanId=0313bafe-9457-11e8-9eb6-529269fb1459
   *           parent=2f64e538-9457-11e8-9eb6-529269fb1459
   *           application=Wavefront http.method=GET
   *           1533529977 343500"
   */
  wavefrontSender.sendSpan("getAllUsers",1533529977L, 343500L, "localhost",
        UUID.fromString("7b3bf470-9456-11e8-9eb6-529269fb1459"),
        UUID.fromString("0313bafe-9457-11e8-9eb6-529269fb1459"),
        ImmutableList.<UUID>builder().add(UUID.fromString(
            "2f64e538-9457-11e8-9eb6-529269fb1459")).build(), null,
        ImmutableList.<Pair<String, String>>builder().
            add(new Pair<>("application", "Wavefront")).
            add(new Pair<>("http.method", "GET")).build(), null);

  /*
   * If there are any failures observed while sending metrics/histograms/tracing-spans above, 
   * you get the total failure count using the below API
   */
  int totalFailures = wavefrontSender.getFailureCount();
  
  /* on-demand buffer flush (might want to do this if you are shutting down your JVM) */
  wavefrontSender.flush();
  
  /*
   * close connection before shutting down JVM 
   * (this will flush in-flight buffer and close connection)
   */
  wavefrontSender.close();
```
