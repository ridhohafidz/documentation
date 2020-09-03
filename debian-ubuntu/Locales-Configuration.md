# Locales Configuration #


The following error may occur if the locale settings have not been configured correctly:

 ```bash
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
    LANGUAGE = (unset),
    LC_ALL = (unset),
    LANG = "en_US.UTF-8"
are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
```

Try the following commands to fix it:
```bash
$ sudo locale-gen
$ sudo dpkg-reconfigure locales
```

If it still occur, try this:
```bash
$ sudo apt-get purge locales
$ sudo apt install locales
$ sudo dpkg-reconfigure locales
```

If that doesn't fix this, try this workaround:
```bash
$ cat /etc/default/locale
...
LANGUAGE=en_US.UTF-8
LC_ALL=en_US.UTF-8
LANG=en_US.UTF-8
LC_CTYPE=en_US.UTF-8
```

Log out of SSH session and log in again.