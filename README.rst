=======
automon
=======

/var/log/messages monitoring tool

This tool will monitor /var/log/messages file
and send alerts via Telegram if detects any anomalies.

Installation
------------

 - ``cd /opt``
 - ``git clone https://github.com/makhomed/automon.git automon``

Upgrade
-------

 - ``cd /opt/automon``
 - ``git pull``


Configuration
-------------

  - ``vim /opt/automon/automon.conf``
  - write to config something like this:

.. code-block:: none

    host localhost
    host example.com one-line description of this host

Configuration file allow comments, from symbol ``#`` to end of line.

Configuration file has only four directives:
``host``, ``log``, ``alert`` and ``delay``.

``host`` directive has syntax: ``host <hostname>[:port] [description]``.
``<hostname>`` part is requred, it may be domain name or ip address.
``port`` is optional, by default used port 22. ``description`` also optional.
If hostname is ``localhost`` or ``127.0.0.1`` - direct access to log files will be used,
else log files will be acessed via ssh.

``log`` directive has syntax ``log </path/to/logfile>``. Default value of ``log``
directive is ``/var/log/messages``. It can be redefined to any other value, for example,
``/var/log/syslog``. Value of ``log`` directive will be used for all below host declarations.
For example:

.. code-block:: none

    host centos1
    host centos2

    log var/log/syslog

    host debian1
    host debian2

``alert`` directive defines path to alert program, default value is
``/opt/automon/bin/alert-via-telegram``. Program ``/opt/automon/bin/alert-via-telegram``
included in automon and send alerts to Telegram via https://pypi.python.org/pypi/telegram-send script.
``alert`` program receive one argument - full name of file with generated alert text. See source
of ``/opt/automon/bin/alert-via-telegram`` program for details. Using ``/opt/automon/bin/alert-via-telegram``
as example you can write own alert program for sending alerts via email or SMS or via any other way.

``delay`` directive defines delay between two automon scans is daemon mode. By default delay is 600 seconds.

Global ignore patterns defined if files in directory ``/opt/automon/ignore.d``. This directory included in automon repository.
Local ignore patterns should be defined in files in directory ``/opt/automon/local-ignore.d``.
This directory is not included in automon repository and should be created manually.
Host-specific ignore paterns should be defined in files in subdirectories named as host name + ".d".
For example, for host ``localhost`` ignore patterns should be defined in files localed inside directory
``/opt/automon/local-ignore.d/localhost.d``, for host ``example.com`` ignore patterns should be defined
in directory ``/opt/automon/local-ignore.d/example.com.d``.

Each line in ignore file should be python regular expression, symbols ``^`` at start and ``$`` at end will be added automatically.
If first non-whitespace symbol of line is ``#`` - such line considered as comment and will be ignored in pattern matching.

Command line arguments
----------------------

.. code-block:: none

    automon [-c /path/to/configuration/file.conf] [mode]

``automon`` has optional command line agrument ``-c </path/to/configuration/file.conf>``.
If agrument ``-c`` not defined - by default will be used config ``/opt/automon/automon.conf``.

``automon`` also has ohe optional positional argument ``mode``. Allowed values are ``daemon``, ``once`` and ``debug``.
``daemon`` mode useful for running ``automon`` as systemd service. In this mode ``automon`` will be run forever with
``delay`` seconds delay between two scans of hosts defined in configuration.
``once`` mode is useful for running ``automon`` from cron. In ``once`` mode ``automon`` run once and exit.
``debug`` mode useful for debug, in this mode no alerts will be send and no logscan state will be saved.
In ``debug`` mode alert will be printed to stdout and ``automon`` will exit. In ``daemon`` and ``once`` modes
alerts will be send to system administrator via alert program.

Before first run
----------------

Before first run you need to create Telegram bot and configure telegram-send script.
Detalis see in https://pypi.python.org/pypi/telegram-send documentation.

Secure Shell
------------

For work you need to generate private ssh key on ``automon`` server
with comamnd ``ssh-keygen -t rsa`` and copy public key from ``/root/.ssh/id_rsa.pub``
to ``/root/.ssh/authorized_keys`` on monitored servers. Also you need to check connection
with monitored server with command ``ssh example.com`` and answer ``yes`` to ssh question:

.. code-block:: none

    # ssh .example.com
    The authenticity of host 'example.com' can't be established.
    ECDSA key fingerprint is SHA256:/cYI0bJzEX+CF3DhGEUQ+ZeGFmMzEJYAt3C15450zKs.
    ECDSA key fingerprint is MD5:44:20:bd:f5:aa:a7:52:ac:c5:19:e5:e0:28:2b:90:49.
    Are you sure you want to continue connecting (yes/no)? yes

Automation via cron
-------------------

Create configuration file ``/opt/automon/cron.conf`` and define hosts to check inside it.
After it configure cron job, for example, in file ``/etc/cron.d/automon``:

.. code-block:: none

    0 * * * * root /opt/automon/automon -c /opt/automon/cron.conf once

Automation via systemd service
------------------------------

Create configuration file ``/opt/automon/service.conf`` and define hosts to check inside it.
After it create systemd service, for example, in file ``/etc/systemd/system/automon.service``:

.. code-block:: none

    [Unit]
    Description=automon
    After=network-online.target

    [Service]
    ExecStart=/opt/automon/automon -c /opt/automon/service.conf daemon
    Restart=always

    [Install]
    WantedBy=multi-user.target

After this you need to start service:

  - ``systemctl daemon-reload``
  - ``systemctl enable automon``
  - ``systemctl start automon``
  - ``systemctl status automon``

If all ok you will see what service is enabled and running.

Automation via multiple systemd services
----------------------------------------

Create multiple configuration file ``/opt/automon/service1.conf``, ``/opt/automon/service2.conf``, ...
and define hosts to check inside it. After it create systemd service,
for example, in file ``/etc/systemd/system/automon@.service``:

.. code-block:: none

    [Unit]
    Description=automon %I
    After=network-online.target

    [Service]
    ExecStart=/opt/automon/automon -c /opt/automon/%i.conf daemon
    Restart=always

    [Install]
    WantedBy=multi-user.target

After this you need to start services:

  - ``systemctl daemon-reload``
  - ``systemctl enable automon@service1``
  - ``systemctl enable automon@service2``
  - ...
  - ``systemctl start automon@service1``
  - ``systemctl start automon@service2``
  - ...
  - ``systemctl status automon@service1``
  - ``systemctl status automon@service2``
  - ...

If all ok you will see what ``automon`` services are enabled and running.

