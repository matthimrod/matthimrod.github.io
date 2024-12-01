---
layout: post
title: Python Context Managers
tags: [python, code]
author: Matt Himrod
---

One of my current projects involves some improvements to code that I wrote last year -- specifically enhancing the "Patient Identity Fixer", which is my pet project of sorts. I plan on adding support for a new API that was recently released, but an improvement that I was looking forward to the most was one to boost performance by adding HTTP sessions and connection pooling. A patient with a single MRN in a single system involves at least 11 calls to the EMPI API. Each additional MRN adds 5 more calls, and some patients have a dozen MRNs from the various facilities in the Health System. Creating an HTTPS session for a single API call was taking between 600 and 1000 milliseconds -- almost a full second for each API call! You can see how this adds up quickly! 

Using connection pooling with the Python `requests` library isn't difficult at all, but it involves initializing a `Session` object and closing it at the termination of the program. I'm sure there's a measure of idiot-proofing within Python that will terminate any open network connections when a given program terminates, but to be a good developer, we should do our housekeeping. The easiest way to do that is with [Context Managers](https://docs.python.org/3/reference/datamodel.html#context-managers).

What's a context manager? If you've opened a File Handle object, you've used a Connection Manager. What's a File Handle? Ok, let me give an example so we're all on the same page... this `with` block is a Context Manager:

```
with open('my_text_file') as file:
    contents = file.read()
    
print(contents)
```

We use these so that we don't have to worry about closing our file when we exit the loop, and they're incredibly easy to implement in classes that involve things like clients. In fact, I think it was the `requests` library that inspired me to make this change.

Adding a context manager involves adding two "special" instance methods to a class similar to the `__init__` constructor method that we're already familiar with. The first is `__enter__`.  It cannot take any parameters - as there isn't anywhere to specify any, and its return value will be bound to the variable specified in the `as` clause - that is, `file` in the example above. Unless you need to do something specific when inside a context manager, this is likely just going to be a simple `return self` without much else. When the context manager code block begins, the constructor is called as it would if we were assigning the part after the keyword `with` to the variable defined after the keyword `as`. 

The function that's likely to have more to it is the `__exit__` method. In simple terms, this method is called when code execution leaves the Context Manager, so this is where you call your `.close()` methods on your objects so that files can be closed, connections can be gracefully terminated, and any needed housekeeping tasks can be tidily handled. In more elaborate terms, the `__exit__` method is also called when an exception occurs in the Context Manager and can opt to suppress the exception. The three required parameters to the `__exit__` method are the exception's type, value, and traceback. The `__exit__` method's return type is `bool`. If the context exited without causing an exception, the value is ignored. Otherwise, a `true` value indicates that the exception should be suppressed.

While there are a few more nuances, the gist is that we could also write our operation this way:

```
try:
    file = open('my_text_file')
    contents = file.read()
except:
    file.close()

print(contents)
```

Back to my project and context managers... keeping the `Session` in a class variable and initializing it in the constructor makes sense. Then all I needed to do was search-and-replace all of my calls to `requests.post()` to use `self.session.post()` and so on. 

Bringing everything together, the class definition and special methods look like this:

```
class EmpiClient:
    """
    Enterprise Master Patient Index (EMPI) REST API Client
    """
    logger: logging.Logger
    config: EmpiConfig
    auth: HTTPBasicAuth
    session: requests.Session

    def __init__(self, config: EmpiConfig) -> None:
        self.logger = logging.getLogger(__name__)
        self.config = config
        self.session = requests.Session()
        self.session.auth = HTTPBasicAuth(self.config.user, self.config.password)
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.session.close()
```

And a snippet of the code where I use this in a different class:

```
    with EmpiClient(self.configuration.empi) as empi_client:
        while patient_list:
            this_source, this_id = patient_list.pop().split(':', maxsplit=1)
            empi_id, corrected_patient = self.build_patient(empi_client, this_source, this_id)
```
