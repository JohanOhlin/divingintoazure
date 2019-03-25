---
layout: post
title: "Custom themes in Office UI Fabric for React"
date: 2018-08-01
tags: react office-ui-fabric
---
<p class="intro"><span class="dropcap">I</span>n a recent project, I've started to use Office UI Fabric as component library for a React application. This is an interesting piece of Microsoft work - who knew they would ever create open source React components?</p> 

The components are well made, but the documentation is lagging behind in good examples, a bit like the MSDN documentation - full of property specifications, but few fully fledged examples.

One neat piece of functionality when working with themes for Office UI Fabric is the [Theme Generator](https://developer.microsoft.com/en-us/fabric#/styles/themegenerator). It gives you the possibility to specify your colours of choice and see the theme generated shades.

{% 
  include image.html 
  url="2018-08-01-custom-themes/theme-generator-selection.png" 
  alt="Selecting your own colours in the Theme" 
  description="You can select your own colours in the Theme" 
%}

If I were to wish for something here it would be the possibility to define a secondary colour as well to create a complementary matching pair based on the colour wheel. However, that's not how this theme works and you're stuck with a monochromatic solution. The focus of this component library is to be functional and simplistic - something it does succeed well with.

The colours you've chosen are also analysed based upon how well they work together. A contrast ratio of less than 4.5 is pointed out and you are recommended to go back to the colour selection to pick other colours.

{% 
  include image.html 
  url="2018-08-01-custom-themes/theme-generator-accessibility.png" 
  alt="Your colour choice is analysed for accessibility" 
  description="Your colour choice is analysed for accessibility" 
%}

The theme generator produces different outputs for you to use, but for embedding themes into React applications the JSON option is the best. In the official [documentation](https://github.com/OfficeDev/office-ui-fabric-react/tree/master/packages/styling#usage-via-javascript-styling-libraries-glamor-aphrodite) you can read about how themes can be imported and used. However, something is wrong in their examples, it just doesn't work. Instead I had to do in the following way.

{% highlight csharp %}
import { getTheme, loadTheme } from '@uifabric/styling';
import * as React from 'react';

loadTheme(
  {
    palette: {
      "themePrimary": "#1490ef",
      "themeLighterAlt": "#f5fafe",
      "themeLighter": "#d7ecfd"
    }
  }
);

class App extends React.Component {
  private theme = getTheme();

  public render() {
    return (
      <div className="App">
        <h1 style={\{color: this.theme.palette.themePrimary} }>It works</h1>
      </div>
    );
  }
}
export default App;
{% endhighlight %}

One important step here is that <code class="code">getTheme()</code> is called inside a component. It doesn't necessarily have to, but the order in which things are called make a difference and <code class="code">loadTheme()</code> has to be called before you call <code class="code">getTheme()</code>. When all files are merged together, it's not easy to know in what order things are being loaded. But when you put the call inside the component, then you know for sure it'll happen after <code class="code">loadTheme()</code> has been called.

These theme details, including other Office UI Fabric configurations, can then also be used if you need to integrate with styling frameworks like [Glamor](https://github.com/threepointone/glamor) or [Aphrodite](https://github.com/Khan/aphrodite).

The example below shows how an Aphrodite style can include pre-defined font styles as well as theme colours. Notice how the font is a collection of settings and has to be added using the spread operator. 

{% highlight csharp %}
import * as React from 'react';
import { getTheme } from "@uifabric/styling";
import { StyleSheet, css } from 'aphrodite';

export const createStyles = () =>
{
    const theme = getTheme();

    return StyleSheet.create({
        header: {
            ...(theme.fonts.xxLarge as any),
            backgroundColor: theme.palette.themePrimary,
            color: theme.palette.white,
            padding: '20px'
        }
    })
}

export class HeaderComponent extends React.Component {
    private styles = createStyles();
    public render() {
        return (
            <header className={css(this.styles.header)}>
                Header for page
            </header>
        );
    }
}
{% endhighlight %}

There are many ways to structure your embedded CSS styles. By creating a separate <code class="code">createStyles()</code> function you can easily separate it into its own style file, if you prefer such structure.