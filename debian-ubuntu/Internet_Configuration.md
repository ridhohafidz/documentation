# No Internet on Debian Server #

When you execute `wget` and it just hangs, it could be a proxy problem (since the servers are mostly internal anyway).
![Before proxy setup](../images/before_proxy.png)


A non-permanent solution would be like this:
![Add set_proxy_function](../images/proxy_function.png)
Creating a custom setproxy() function in the `.bashrc` file and execute it when needed.


While a more permanent solution would be modifiying `/etc/wgetrc`:
![Modify /etc/wgetrc](../images/proxy_setup.png)

Try `wget`-ing again, and it should whoosh right away :)

![After proxy setup](../images/after_proxy.png)
