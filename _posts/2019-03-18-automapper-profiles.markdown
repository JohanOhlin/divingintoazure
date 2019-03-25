---
layout: post
title: "AutoMapper profile injections"
date: 2019-03-18
tags: automapper .net-core
---

<p class="intro"><span class="dropcap">I</span> am a big fan of Jimmy Bogard's AutoMapper. It helps you focus on solving your problem. But once in a while you encounter mapping scenarios that just are a bit more complex and can't be solved out of the box.</p>

I have a scenario in a system with subscriptions where each subscription can have different levels. In the domain model, I only store the subscription level id, but in the view model I'd like that to be translated into the actual name for the level.

For simplicity, I've only included the relevant properties in the models and code below.

{% highlight csharp %}
public class SubscriptionDomainModel
{
  public int Id { get; set; }
  public int SubscriptionLevelId { get; set; }
}

public class SubscriptionViewModel
{
  public int Id { get; set; }
  public string SubscriptionLevelName { get; set; }
}
{% endhighlight %}

The default AutoMapper profile can handle all properties having the same name and type

{% highlight csharp %}
public class SubscriptionViewModelProfile : Profile
{
  public SubscriptionViewModelProfile()
  {
    CreateMap<SubscriptionDomainModel, SubscriptionViewModel>();
  }
}
{% endhighlight %}

But the <code>SubscriptionLevelId</code> property needs custom code to be mapped - a resolver. Here are two different approaches to how that can be solved.

#### Resolving a static list

If the list of subscription levels is contained in code and accessible through a synchronous call then you can add an AutoMapper resolver and inject the class containing the list directly into it.

Here's the simplified service containing the levels.

{% highlight csharp %}
public interface ISubscriptionLevelService
{
  List<SubscriptionLevelDomainModel> GetSubscriptionLevels();
}
public class SubscriptionLevelService : ISubscriptionLevelService
{
  public List<SubscriptionLevelDomainModel> GetSubscriptionLevels()
  {
    return new List<SubscriptionLevelDomainModel> {
      new SubscriptionLevelDomainModel {
        Id = 1,
        Title = "Basic subscription"
      },
      new SubscriptionLevelDomainModel {
        Id = 2,
        Title = "Premium subscription"
      },
    };
  }
}
{% endhighlight %}

We can now create a resolver and inject the service into it for object lookup

{% highlight csharp %}
public class SubscriptionLevelResolver : IValueResolver<
  SubscriptionDomainModel, 
  SubscriptionViewModel, 
  string
>
{
  private readonly ISubscriptionLevelService _service;
  public SubscriptionLevelResolver(ISubscriptionLevelService service)
  {
    _service = service;
  }

  public string Resolve(SubscriptionDomainModel source, 
      SubscriptionViewModel destination, 
      string destMember, 
      ResolutionContext context
  )
  {
    var levels = _service.GetSubscriptionLevels();
    if (levels.Any(x => x.Id == source.SubscriptionLevelId))
    {
      return levels.Single(x => x.Id == source.SubscriptionLevelId).Title;
    }
    return $"Unknown level ({source.SubscriptionLevelId})";
  }
}
{% endhighlight %}

Finally, in the mapper profile we can now specify that the SubscriptionLevelId should be resolved using the new resolver.

{% highlight csharp %}
CreateMap<SubscriptionDomainModel, SubscriptionViewModel>()
  .ForMember(d => d.SubscriptionLevel, 
           opt => opt.MapFrom<SubscriptionLevelResolver>());
{% endhighlight %}

#### Resolving an async service

So far so good. But what if your service makes an async call to database or cache to fetch a list of subscription levels? [AutoMapper has no support for async calls](https://github.com/AutoMapper/AutoMapper/issues/2615#issuecomment-385745684) due to the complexity that would bring to the usage of reflection and expression trees. You therefore have to solve this another way. My suggested solution is to inject the subscription level list when you do the mapping.

Injection of values is done by using the AutoMapper context. In the resolver, you can use the Items dictionary inside the context to send variables. In our solution, it could look like this:

{% highlight csharp %}
public class SubscriptionLevelResolver : IValueResolver<
  SubscriptionDomainModel, 
  SubscriptionViewModel, 
  string
>
{
  public string Resolve(
    SubscriptionDomainModel source, 
    SubscriptionViewModel destination, 
    string destMember, 
    ResolutionContext context)
  {
    var levels = context.Items["SubscriptionLevels"]
      as List<SubscriptionLevelDomainModel>;
    if (levels != null && levels.Any(x => 
      x.Id == source.SubscriptionLevelId))
    {
      return levels.FirstOrDefault(x => 
        x.Id == source.SubscriptionLevelId)?.Title;
    }
    return $"Unknown level ({source.SubscriptionLevelId})";
  }
}
{% endhighlight %}

When requesting the mapping you now have to pass along the list of subscription levels.

{% highlight csharp %}
var subscriptionViewModel = _mapper.Map<SubscriptionViewModel>(
  subscription, 
  opt => opt.Items["SubscriptionLevels"] = subscriptionLevels
);
{% endhighlight %}

This works fine, but I'm not really happy with the way the list is passed in - it's quite easy to forget to pass in the list and it's also possible to misspell the key in the dictionary. To make it a bit more visible, you can add an extension method to the mapper that asks for a list when you try to do this specific mapping.

{% highlight csharp %}
namespace AutoMapper
{
  public static class AutoMapperExtensions
  {
    public static SubscriptionViewModel Map<SubscriptionViewModel>(
        this IMapper map, 
        SubscriptionDomainModel domainModel, 
        List<SubscriptionLevelDomainModel> subscriptionLevels)
    {
      return map.Map<SubscriptionViewModel>(domainModel, opt => 
        opt.Items["SubscriptionLevels"] = subscriptionLevels);
    }
  }
}
{% endhighlight %}

The extension is added to the <code>AutoMapper</code> namespace so it'll be available each time you use the <code>Map</code> function. This is how you now can call the function, and it even comes up as one of the suggested ways.

{% highlight csharp %}
var subscriptionViewModel = _mapper.Map<SubscriptionViewModel>(
  subscription, 
  subscriptionLevels
);
{% endhighlight %}

#### Summary
AutoMapper is a really helpful tool when you need to map different data classes. The resolver makes it possible to convert complex scenarios and you can even inject services to use when converting. However, async is not supported in the resolver so you have to call async functions before and pass in the values you need.
