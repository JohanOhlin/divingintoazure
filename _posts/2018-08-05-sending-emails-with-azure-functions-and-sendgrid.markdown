---
layout: post
title: "Sending emails with web hooks in Azure Function and Sendgrid"
date: 2018-08-05
tags: azure azure-functions sendgrid
---

<p class="intro"><span class="dropcap">S</span>ometimes you might need to send emails from an Azure Function, maybe a status email to admin or a notification to a user. In this example, I show you how to create an async function that can send SendGrid emails. By letting the function be triggered by an HTTP call you can easily connect other Azure function or flows and send emails as needed.</p>

#### Create function

Create a new C# function with an HTTP trigger. Authorization level can be specified to <code class="code">function</code> so it's protected from unwanted calls. The function is also protected by HTTPS by default.

Replace the default code with the following snippet.

{% highlight csharp linenos %}
#r "SendGrid"
using System;
using System.Net;
using SendGrid.Helpers.Mail;

public static async Task<HttpResponseMessage> Run(
  HttpRequestMessage req, 
  IAsyncCollector<Mail> mailCollection, 
  TraceWriter log
)
{
  try 
  {
    var email = await req.Content.ReadAsAsync<Email>();
    var message = new Mail
    {        
        Subject = email.Subject          
    };

    var personalization = new Personalization();
    personalization.AddTo(new Email(email.To));   

    Content content = new Content
    {
        Type = "text/html",
        Value = email.Body
    };
    message.AddContent(content);
    message.AddPersonalization(personalization);
    await mailCollection.AddAsync(message);

    string msg = $"Sent email with subject '{email.Subject}' " +
                  "to Administrator"; 
    log.Info(msg);
    return req.CreateResponse(HttpStatusCode.OK, msg);

  } 
  catch (Exception e) 
  {
    string msg = $"Failed to send email with subject " + 
                  "'{email.Subject}' to Administrator"; 
    log.Error(msg);
    return req.CreateResponse(HttpStatusCode.BadRequest, msg);
  }
}

public class EmailStructure
{
  public string To { get; set; }
  public string Subject { get; set; }
  public string Body { get; set; }
}
{% endhighlight %}

#### Update bindings

Click on <code class="code">View Files</code> to the right and open the <code class="code">function.json</code> file. Update the bindings to include the SendGrid configuration as in the example below. Note that the field <code class="code">apiKey</code> should contain the text <code class="code">SendGridAttribute.ApiKey</code> because it's referring to an app setting for the function rather than the actual api key itself.

{% highlight csharp %}
{
  "disabled": false,
  "bindings": [
    {
      "authLevel": "function",
      "name": "req",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "post"
      ]
    },
    {
      "name": "$return",
      "type": "http",
      "direction": "out"
    },
    {
      "name": "mailCollection",
      "type": "sendGrid",
      "direction": "out",
      "apiKey": "SendGridAttribute.ApiKey",
      "from": "sender@mycompany.com"
    }
  ]
}
{% endhighlight %}

#### Configure SendGrid

Configure a SendGrid service in Azure. The free tier allows you to send 25 000 emails a months which is more than enough for this sample application. After you've setup SendGrid in Azure you need to login to the SendGrid portal to create an API key. You can access this portal by clicking on the <code class="code">Manage</code> button inside your newly created Azure SendGrid account.

#### Add application setting

Open Application Settings for the Function App (not the individual function, since many functions can run in a Function App). Add the application setting <code class="code">SendGridAttribute.ApiKey</code> with the key you've created in the SendGrid system. 

#### Get URL and send test email

Open the newly created Azure function and click on the <code class="code">Get function URL</code> button. Select <code class="code">Function key</code> and copy the URL created. Now you can send POST message with the following JSON data structure.

{% highlight csharp %}
{
    "To": "some.person@some.email.com",
    "Subject": "Email alert",
    "Body": "Someone is sending you emails!"
}
{% endhighlight %}
