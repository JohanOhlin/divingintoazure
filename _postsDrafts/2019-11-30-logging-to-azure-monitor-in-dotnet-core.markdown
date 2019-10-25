---
layout: post
title: "Logging to Azure Monitor in .NET Core"
date: 2019-10-01
tags: c# azure .net-core application-insights azure-monitor
blog_serie: logging_with_azure_monitor
---

<p class="intro"><span class="dropcap">L</span>ogging isn't the first thing that comes to mind when you create a new project. But it's still vitally important to be able to track the health of an application. In this post we'll go through some recommendations on how to handle logging in .NET Core.</p>

{%
  include blog_serie.html
  page=page
%}

### What data do you need?

<p>There are many logging frameworks available to choose from.</p>

### Application Insights

If you want to log all the data and traces from the app as well as from the instance it's running on, then Application Insights might be a good choice for you. You pay for the amount of logging/telemetry data your app is sending over to Azure Monitor, currently at a [cost of $2.76/GB with the first 5GB being free](https://azure.microsoft.com/en-gb/pricing/details/monitor/). How much data capacity do you need? I created a test app logging a few traces, warnings and errors every second, and the estimated data usage was 9GB/month according to Azure Monitor.

#### Using operations

<p>By wrapping your code in telemetry operations you can group all logging and trace calls by operation name and link different types of events. All events has to be triggered inside the operation scope. If an exception is thrown inside of an operation but is caught outside of it, then the error logging will be without the operation information</p>

<p>In the example below we start an operation and make two log calls</p>

{% highlight csharp linenos %}
using (_telemetryClient.StartOperation<RequestTelemetry>("Create invoice for customer"))
{
    _logger.LogInformation("Invoice created for customer 123");
    _telemetryClient.TrackEvent("Customer 123 has a lot to pay now");
}
{% endhighlight %}

<p>You can't wrap several operations inside one another. If you do, the inner ones won't be used.</p>

### Azure Monitor

<p>All the events from Application Insights can be found in the Azure Monitor Logs section. Data here is searched using the Kusto Query Language (KQL) which is somewhat similar to SQL, but also has a lot of unique features to make searches easier.</p>

<p>When you expand the worker node you'll find a number of data sets containing your data:</p>

<table>
<thead>
  <tr><th scope="col">Data set</th><th scope="col">Content</th></tr>
</thead>
<tbody>
  <tr><td>traces</td><td>All events from the ILogger component, but not LogError containing an exception</td></tr>
  <tr><td>customEvents</td><td>Calling TrackEvent on the TelemetryClient</td></tr>
  <tr><td>requests</td><td>Requests made by HttpClient</td></tr>
  <tr><td>dependencies</td><td>I.e. URL's called by HttpClient, or database calls</td></tr>
  <tr><td>exceptions</td><td>LogError events from ILogger containing an exception</td></tr>
  <tr><td>customMetrics</td><td>I.e. System.Runtime data about the app</td></tr>
  <tr><td>performanceCounters</td><td>CPU and Memory info from Worker Service</td></tr>
</tbody>
</table>

<p>Un-caught exceptions are not reported back to Application Insights so you need to catch and log them yourself.</p>

{% highlight csharp linenos %}
try {

} catch (Exception ex) {
  _logger.LogError(ex, "Failed to create invoice");
}
{% endhighlight %}

<p></p>

{% highlight bash linenos %}

{% endhighlight %}

<p>KQL is the query language you use to extract data from all the logs being added to Azure Monitor Logs. It has some similarities to SQL, but also allows you to divide your queries into sections in a very easy way.</p>

<p>This example KQL script fetches data from three sources (customEvents, traces and exceptions) and counts the severity events per hour</p>

{% highlight kql linenos %}
let selectedAppName = "WorkerService";
let dataStartDate = ago(24h);
let getSeverity = (severityLevel:int) {
    case(severityLevel <= 1, 'Info'
       , severityLevel == 2, 'Warning'
       , severityLevel >= 3, 'Error'
       , 'Unknown')
};
let customEventsData = customEvents
| where appName == selectedAppName
| where timestamp >= dataStartDate
| project hourOfDay = datetime_part("hour", timestamp)
        , severityLevel = toint(1);
let traceData = traces
| where timestamp >= dataStartDate
| project hourOfDay = datetime_part("hour", timestamp)
        , severityLevel;
let exceptionData = exceptions
| where timestamp >= dataStartDate
| project hourOfDay = datetime_part("hour", timestamp) 
        , severityLevel;
union kind=outer customEventsData, traceData, exceptionData
| project hourOfDay, severity = getSeverity(severityLevel)
| evaluate pivot(severity)
| sort by hourOfDay asc
{% endhighlight %}

<p>The different severity levels have a pivot applied to them so they become columns. The end result looks something like this:</p>

<table>
<thead><tr><th>hourOfDay</th><th>Warning</th><th>Info</th><th>Error</th></tr></thead>
<tbody>
<tr><td>14</td><td>4</td><td>463</td><td>1</td></tr>
<tr><td>15</td><td>10</td><td>839</td><td>3</td></tr>
<tr><td>16</td><td>5</td><td>738</td><td>0</td></tr>
<tr><td>17</td><td>2</td><td>193</td><td>0</td></tr>
<tr><td>18</td><td>0</td><td>378</td><td>1</td></tr>
<tr><td>19</td><td>18</td><td>1062</td><td>2</td></tr>
</tbody>
</table>

### Structured logging

<p>Azure Monitor can harvest huge amounts of data from many different sources. This is great! But also a big pain point if you can't find the relevant data due to vast amount of non-relevant data. Structured logging is not a new concept but has been around for a number of years. It basically gives you the option to add metadata to your logging, metadata that then can be searched on in Azure Monitor.</p>

<p>Take this simple example</p>

{% highlight csharp linenos %}
_logger.LogWarning($"Customer {Customer.Id} has several unpaid invoices");
{% endhighlight %}

<p>In this example, the customerId field will be replaced with the id of the customer and the new processed string will be sent to the logging framework. But with structured logging we can send both the unformatted string and the property separately. The logging framework will format the string correctly and also store the property as metadata. We do that by simply changing the logging line above to something like this:</p>

{% highlight csharp linenos %}
_logger.LogWarning("Customer {customerId} has several unpaid invoices", Customer.Id);
{% endhighlight %}

<p>Notice how we removed the leading <code class="code">$</code> before the string. The result in Azure Monitor is an object named <code class="code">customDimensions</code> next to the rest of the logging data, structured like this.</p>

{% highlight json linenos %}
{
	"{OriginalFormat}": "Customer {customerId} has several unpaid invoices",
	"customerId": "12345"
}
{% endhighlight %}

<p>We can then change in the previous KQL query to filter by a single customer</p>

{% highlight kql linenos %}
let selectedAppName = "WorkerService";
let dataStartDate = ago(24h);
let customerId = 12345;
let getSeverity = (severityLevel:int) {
    case(severityLevel <= 1, 'Info'
       , severityLevel == 2, 'Warning'
       , severityLevel >= 3, 'Error'
       , 'Unknown')
};
let customEventsData = customEvents
| where appName == selectedAppName
| where customDimensions.customerId == customerId
| where timestamp >= dataStartDate
| project hourOfDay = datetime_part("hour", timestamp)
        , severityLevel = toint(1);
let traceData = traces
| where timestamp >= dataStartDate
| where customDimensions.customerId == customerId
| project hourOfDay = datetime_part("hour", timestamp)
        , severityLevel;
let exceptionData = exceptions
| where timestamp >= dataStartDate
| where customDimensions.customerId == customerId
| project hourOfDay = datetime_part("hour", timestamp) 
        , severityLevel;
union kind=outer customEventsData, traceData, exceptionData
| project hourOfDay, severity = getSeverity(severityLevel)
| evaluate pivot(severity)
| sort by hourOfDay asc
{% endhighlight %}
