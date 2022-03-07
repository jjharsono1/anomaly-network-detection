<!--- Live website link: https://ctran0615.github.io/anomaly-network-detection/ --->
# Table of Contents
- [Introduction](#introduction)
- [Methodology](#methodology)
    - [Prior Work](#prior-work)
    - [The Data](#the-data)
    - [Definition of an Anomaly](#definition-of-an-anomaly)
    - [EDA](#eda)
    - [Feature Engineering](#feature-engineering)
    - [Anomaly Detection Methods and Models](#exploration-of-anomaly-detection-methods)
- [Results](#results)
    - [Performance Measures](#performance-measures)
    - [Performance](#performance)
- [Future Work](#future-work)
    - [Bigger Questions](#bigger-questions)
    - [Improvements](#improvements)
- [References](#references)
- [Appendix](#appendix)
# Introduction
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Viasat Inc. is an American communications company that provides high-speed satellite broadband services and secure networking systems covering military and commercial markets. Therefore, ensuring high quality network performance is crucial for maintaining customer satisfaction. Fundamental factors that affect network performance are packet loss and latency. </p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Small units of data that are sent and recieved across a network are called packets, and packet loss occurs when one or more of these packets fails to reach its intended destination. Latency is the delay between a user's action and a web application's response to that action. High quantities of each of these often results in slow service and network disruptions for the user.</p> 
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; The problem we are trying to explore is: can we detect anomalies within network traffic, whether it be an increase in the packet loss rate, larger latency, or both? Packet loss can be caused by a number of issues, including: network congestion, problems with network hardware, software bugs and security threats. On the other hand, main factors that affect network latency are: the transmission medium (copper cable-based networks vs. modern optic fibers), propagation (how far apart the networks are), efficiency of routers and storage delays. </p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; The ability to detect anomalies on a data stream would be extremely useful as it can be an effective monitoring & alerting system for poor network conditions. If an anomaly is predicted, our hope is that a proper recovery protocol can be put in place to stabilize network performance, creating a better experience for the end user.</p> 
# Methodology
### The Data

We will be using data generated from DANE (Data Automation and Network Emulation Tool), a tool that automatically collects network traffic datasets in a parallelized manner without background noise, and emulates a diverse range of network conditions representative of the real world. Using DANE, we are able to configure parameters such as latency and the packet loss ratio ourselves. More importantly, we are able to simulate an anomaly by changing the configuration settings of the packet loss rate and latency mid run. The data collected is already aggregated per second with these fields:

| DANE Features | Description | 
|--------------- | ----------- | 
| Time | Broken down into seconds(epoch) | 
| Port 1 & 2 | Identifier of where the packets are being sent to and from.  Can originate from either | 
| IP | IP address of the ports in comunication |
| Proto | IP Protocol number that identifies the transport protocol for the packet | 
| Port 1->2 Bytes | Total bytes sent from port 1 to 2 in given second| 
| Port 2->1 Bytes | Total bytes sent from port 2 to 1 in given second |
| Port 1->2 Packets | Total packets sent from port 1 to 2 in given second| 
| Port 2->1 Packets | Total packets sent from port 2 to 1 in given second | 
| Packet_times | Time in milliseconds that the packet arrived |
| Packet_size | Size of the bytes of the packet | 
| Packet_dirs | Which endpoint was the source of each packet that arrived | 
<br>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  We ran only two scenarios at once to prevent overloading our CPU by running too many DANE scenarios concurrently. Each DANE run is 5 minutes long.  The configuration change happens at the 180 second mark and what we configure for our packet loss rate determines the likelihood of each packet being dropped. This way, we are able to simulate an anomaly within our data and are able to simulate packet loss in a more realistic manner. Our steady state had a packet loss ratio of 1/5,000 and a latency of 40 ms. We are focused on identifying changes of a factor of 4 and above. In our case a packet loss ratio of 1/1,250  and a latency of 160 ms or greater or a packet loss ratio of 1/20,000 and latency of 10 ms. </p>
### Definition of an Anomaly
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; The type of behavior we are looking for is different from what a conventional anomaly would behave like. Typical anomalies are characterized by significantly large spikes or drops in a feature. The behavior we are looking for is more closely related to anomalous regions where the degradation in the network continues; these anomalies resemble shifts. As we will show in the next section, spikes in our features that would normally resemble anomalous behavior are perfectly random and not caused by any change in network quality. These anomalies would be considered false positives if detected.</p>
### EDA & Feature Engineering
![](EDA.png?raw=true)
<br>
**Figure 1** The packets per second sent from machine 1 → 2 with conditions of 50 ms latency and 1/2500 packets expected to be dropped with the conditions shifting to 300 ms latency and 1/1250 expected packet drops at 180 seconds.
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Comparing last quarter’s deterministic packet drops data with this quarter’s random packet drops data, we can see that the correlation between packets per second and packet loss is still there. Since packet loss and latency are our only ways of generating an anomaly, features such as packets per second and other features correlated with packet loss and latency will be our basis for determining whether there is an anomaly or not.</p>
    
**Exploring Packets Per Second Feature**

|First 180 Seconds| Last 120 Seconds|
|-|----------------|-----------------|
|Mean|1783.72|428.43|
|Standard Deviation|710.10|260.22|
|Max|4404|1548|
|Min|680|152|

### Exploration of Anomaly Detection Methods
**I.** Forecasting
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Since we are dealing with time series data, we can create an anomaly detection model through the use of forecasting techniques. The basic concept is that we will pick a feature, in this case total packets sent per second (volume of traffic) and build a forecast. If the expected value is outside of our prediction interval (threshold) we will flag it as an anomaly. We are employing a multivariate time series forecast because we are using predictors other than the series (a.k.a exogenous variables).</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; We will be focusing on building an ARIMA (Auto Regressive Integrated Moving Average) model. This model is actually a class of models that ‘explains’ a given time series based on its own past values, that is, its own lags and the lagged forecast errors, so that equation can be used to forecast future values.</p> 
An ARIMA model can be characterized by 3 terms:
- P: Order of the "AR" term which refers to the number of lags of Y to be used as predictors
    - A pure Auto Regressive (AR only) model is one where Yt depends only on its own lags. That is, Yt is a function of the ‘lags of Yt’. 
    ![](ar-formula.png?raw=true)
- Q: Order of the "MA" term which refers to the number of lagged forecast errors that should go into the model
    - Likewise a pure Moving Average (MA only) model is one where Yt depends only on the lagged forecast errors.
    ![](ma-formula.png?raw=true)
- D: Number of differencing required to make the time series stationary

An ARIMA model is one where the time series was differenced at least once to make it stationary and you combine the AR and the MA terms. So the equation becomes:
![](arima-formula.png?raw=true)
# Results
### Performance Measures
We chose to use **F1 score** as our metric but also considered **precision** in our metrics. Our motivation behind choosing F1 score and precision is because anomalous regions cause by network degradation are assumed to be rare. Since the intended use for this ensemble model is to alert ISP employees what connections are degraded if too many false positives are being flagged then there would not be much use in the alarm.
<br/><br/>
<p> Our goal is to have the users trust that if there is an alarm they can be sure that there is network degradation, hence why precision is considered. We also do not want our model to always predict negative, making it trivial so F1 score was included. </p>

|F1 Score Formula| Precision Formula|
| ---------------| -----------------|
|![](f1-formula.png?raw=true)| ![](precision-formula.png?raw=true)|

### Performance

![](ensemble-performance.png?raw=true) 
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Our method to combine our ARIMA and MAD model followed a simple logic. We ran both models independently and compared the results of each model. We used an AND operator to see where both models detected an anomaly and labeled it as an anomaly. Otherwise, if the condition is not met, then we do not label it as an anomaly. </p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Using the ensemble logic described above, our results show that we are able to detect anomalies early on when they occur. However, we have one false negative where no anomaly appears due to the fact that our ARIMA model detects the anomalies early on but the MAD model detects the anomalies later, so there is no overlap for our ensemble logic to agree on an anomaly. </p>
# Future Work
### Bigger Questions
### Improvements
# References
# Appendix
