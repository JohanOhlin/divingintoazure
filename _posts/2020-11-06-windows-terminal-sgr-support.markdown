---
layout: post
title: "Using SGR5 in Windows Terminal"
date: 2020-11-06
tags: windows-terminal
twitter-title: "Using SGR5 in Windows Terminal"
description: "With the release of Windows Terminal 1.4 you now has the possibility to add blinking colourful text to your terminal text. I'll show you how it's done."
image: 2019-11-19-deploying-worker-service-to-kubernetes/cargo-ship.jpg
---

<p class="intro"><span class="dropcap">W</span>ith the release of (Windows Terminal 1.4)[https://devblogs.microsoft.com/commandline/windows-terminal-preview-1-4-release/] support for SGR 5 parameters were added. SGR (Select Graphic Rendition) parameters sets display attributes when you're showing text on a terminal. These attributes can alter things like colour of the text and now also, and with this Windows Terminal release, you can make the text blink.</p>

The easiest way to to get started is pick a ready made ASCII art and then add some blinking colour to it. For this illustration I picked a [lighthouse](https://www.asciiart.eu/buildings-and-places/lighthouses).

To get right to it, create a new bash file `lighthouse.sh` and add the following code to the file.

{% highlight bash %}
echo -e ""
echo -e " + \033[5;1;33m/\033[0m"
echo -e "\033[5;1;33m\\ \033[0m | \033[5;1;33m/\033[0m"
echo -e " \033[5;1;33m\\ \033[0m | \033[5;1;33m/\033[0m"
echo -e " \033[5;1;33m\\ \033[0m / \ \033[5;1;33m/\033[0m"
echo -e " \033[5;1;33m\\ \033[0m /**\_**\ \033[5;1;33m/\033[0m"
echo -e " \033[5;1;33m/\033[0m |\033[5;1;33m\u2588\u2588\033[0m|\033[5;1;33m\u2588\u2588\033[0m| \033[5;1;33m\\ \033[0m"
echo -e " \033[5;1;33m/\033[0m |;| |;| \033[5;1;33m\\ \033[0m"
echo -e " \033[5;1;33m/\033[0m \\. . / \033[5;1;33m\\ \033[0m"
echo -e "\033[5;1;33m/\033[0m ||: . | \033[5;1;33m\\ \033[0m"
echo -e " ||: | \033[5;1;33m\\ \033[0m"
echo -e " ||:. | \033[5;1;33m\\ \033[0m"
echo -e " ||: .|"
echo -e " ||: , | /'\\"
echo -e " ||: | /'\\"
echo -e " ||: _ . | /'\\"
echo -e " _||_| |**| \033[94m\_\_**"
echo -e " \_\_\_\_~\033[0m |_| |**\_ \033[94m**-----~ ~'---,** \_**"
echo -e "-~ ~---**\_,--~' ~~----\_\_\_**-~'"
echo -e "~----,\_\_\_\_\033[0m -Niall-"
echo -e ""
echo -e ""
{% endhighlight %}

Add a command to your startup script to always run this file at login.

And, voila, you have a pimped login screen for your WSL terminal.

So how does it work? The key is the escape character `\033`. It's not specific to bash but is rather interpreted by Windows Terminal.

The SGR codes used in the example are

| Code | Effect          |
| ---- | --------------- |
| 0    | Reset to normal |
| 1    | Bold            |
| 5    | Slow blink      |
| 33   | Yellow          |
| 94   | Bright blue     |

You can specify several SGR codes at once after the escape character by starting with `[` and ending with `m`. Some examples:

Change terminal output to slow blinking, bold and yellow
{% highlight bash %}
\033[5;1;33m
{% endhighlight %}

Change terminal output to bright blue
{% highlight bash %}
\033[94m
{% endhighlight %}

Reset terminal output to normal
{% highlight bash %}
\033[0m
{% endhighlight %}

By alternating these three examples, we're able to create a blinking effect in the ASCII art making the light pulsate and the water turn blue.

There are a lot more things you can do as shown [here](https://en.wikipedia.org/wiki/ANSI_escape_code#SGR_parameters). If you want to build more advanced things you can also use blocks. I had to change the windows of the lighthouse to use unicode [block elements](https://en.wikipedia.org/wiki/Block_Elements) instead to make the colour fill up the whole window.

You might think that blinking text isn't that useful - and you're right. But it's fun, you have to admit that.
