Installation
============

This document describes the necessary steps to configure Patchwork in a
production environment. This requires a significantly "harder" deployment than
the one used for development. If you are interested in developing Patchwork,
refer to the :doc:`development guide <../development/installation>` instead.

This document describes a single-node installation of Patchwork, which will
handle the database, server, and application. It is possible to split this into
multiple servers, which would provide additional scalability and availability,
but this is is out of scope for this document.

Deployment Guides, Provisioning Tools and Platform-as-a-Service
---------------------------------------------------------------

Before continuing, it's worth noting that Patchwork is a Django application.
With the exception of the handling of incoming mail (described below), it can
be deployed like any other Django application. This means there are tens, if
not hundreds, of existing articles and blogs detailing how to deploy an
application like this. As such, if any of the below information is unclear then
we'd suggest you go search for "Django deployment guide" or similar, deploy
your application, and submit a patch for this guide to clear up that confusion
for others.

You'll also find that the same search reveals a significant number of existing
deployment tools aimed at Django. These tools, be they written in Ansible,
Puppet, Chef or something else entirely, can be used to avoid much of the
manual configuration described below. If possible, embrace these tools to make
your life easier.

Finally, many Platform-as-a-Service (PaaS) providers and tools support
deployment of Django applications with minimal effort. Should you wish to avoid
much of the manual configuration, we suggest you investigate the many options
available to find one that best suits your requirements. The only issue here
will likely be the handling of incoming mail - something which many of these
providers don't support. We address this in the appropriate section below.

Requirements
------------

For the purpose of this guide, we will assume an *Ubuntu 16.04 host*: commands,
package names and/or package versions will likely change if using a different
distro or release. Similarly, usage of different package versions to the ones
suggested may require slightly different configuration. For example, this guide
describes configuration with *Python 3* and using Python 2 will require
different packages and some minor changes to configuration files.

Before beginning, you should update this system:

.. code-block:: shell

   $ sudo apt-get update
   $ sudo apt-get upgrade

We also need to configure some environment variables to ease deployment:

``DATABASE_NAME=patchwork``

  Name of the database. We'll name this after the application itself.

``DATABASE_USER=www-data``

  Username that the Patchwork web application will access the database with. We
  will use ``www-data``, for reasons described below.

``DATABASE_PASS=``

  Password that the Patchwork web application will access the database with. As
  we're going to use ident authentication (more on this later), this will be
  unset.

``DATABASE_HOST=``

  IP or hostname of the database host. As we're hosting the application on the
  same host as the database and hoping to use ident authentication, this will
  be unset.

``DATABASE_PORT=``

  Port of the database host. As we're hosting the application on the same host
  as the database and using the default configuration, this will be unset.

The remainder of the requirements are listed as we install and configure the
various components required.

Database
--------

Install Requirements
~~~~~~~~~~~~~~~~~~~~

We're going to rely on PostgreSQL, though MySQL is also supported:

.. code-block:: shell

   $ sudo apt-get install -y postgresql postgresql-contrib

Configure Database
~~~~~~~~~~~~~~~~~~

We need to create a database for the system using the database name above. In
addition, we need to add accounts for two system users, the web user (the user
that the web server runs as) and the mail user (the user that the mail server
runs as). On Ubuntu these are ``www-data`` and ``nobody``, respectively.
PostgreSQL supports ident-based authentication, which uses the standard UNIX
authentication method as a backend. This means no database-specific passwords
need to be configured.

PostgreSQL created a user account called ``postgres``; you will need to run
commands as this user.

.. code-block:: shell

   $ sudo -u postgres createdb $DATABASE_NAME
   $ sudo -u postgres createuser $DATABASE_USER
   $ sudo -u postgres createuser nobody

We will also need to apply permissions to the tables in this database but
seeing as the tables haven't actually been created yet this will have to be
done later.

Finally, we should enable ``trust`` authentication. This will allow us to use
the local ``www-data`` user without having to set a password for a daemon
account. Replace the following line in
``/etc/postgresql/9.6/main/pg_hba.conf``::

    local   all             all                                     ident

with::

    local   all             all                                     trust

Patchwork
---------

Install Requirements
~~~~~~~~~~~~~~~~~~~~

The first requirement is Patchwork itself. It can be downloaded like so:

.. code-block:: shell

   $ wget https://github.com/getpatchwork/patchwork/archive/v2.0.0.tar.gz

We will install this under `/opt`, though this is only a suggestion:

.. code-block:: shell

   $ tar -xvzf v2.0.0.tar.gz
   $ sudo mv v2.0.0 /opt/patchwork

.. important::

   Per the `Django documentation`__, source code should not be placed in your
   web server's document root as this risks the possibility that people may be
   able to view your code over the Web. This is a security risk.

__ https://docs.djangoproject.com/en/dev/intro/tutorial01/#creating-a-project

Next we require Python. If not already installed, then you should do so now.
Patchwork supports both Python 2.7 and Python 3.3+, though we're going to use
the latter to ease future upgrades. Python 3 is installed by default, but you
should validate this now:

.. code-block:: shell

   $ sudo apt-get install -y python3

We also need to install the various requirements. Let's use system packages for
this also:

.. code-block:: shell

   $ sudo apt-get install -y python3-django python3-psycopg2 \
       python3-djangorestframework python3-django-filters

.. tip::

   The `pkgs.org <https://pkgs.org/>`__ website provides a great reference for
   identifying the name of these dependencies.

You can also install requirements using `pip`. If using this method, you can
install requirements like so:

.. code-block:: shell

   $ sudo pip install -r /opt/patchwork/requirements-prod.txt

.. _deployment-settings:

Configure Patchwork
~~~~~~~~~~~~~~~~~~~

You will also need to configure a `settings file`__ for Django. A sample
settings file is provided that defines default settings for Patchwork. You'll
need to configure settings for your own setup and save this as
``production.py``.

.. code-block:: shell

   $ cd /opt/patchwork
   $ cp patchwork/settings/production{.example,}.py

Alternatively, you can override the ``DJANGO_SETTINGS_MODULE`` environment
variable and provide a completely custom settings file.

The provided ``production.example.py`` settings file is configured to read
configuration from environment variables. We're not actually going to use this
here, preferring to hard code settings instead. If you wish to use environment
variables, you should export each setting using the appropriate name, e.g.
``DJANGO_SECRET_KEY``, ``DATABASE_NAME``, ``EMAIL_HOST``, etc.

.. important::

   You should not include shell variables in settings but rather hardcoded
   values. These settings files are evaluated in Python - not a shell. Load any
   required environment variables using ``os.environ``.

__ https://docs.djangoproject.com/en/1.8/ref/settings/

Databases
^^^^^^^^^

As described previously, we're going to modify the ``production.py`` settings
file we created earlier to hard code our settings. Replace the ``DATABASE``
setting with the below to use the database configuration we're described in the
introduction:

.. code-block:: python

   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.postgresql_psycopg2',
           'NAME': 'patchwork',
           'USER': 'www-data',
           'PASSWORD': '',
           'HOST': '',
           'PORT': '',
           'TEST': {
               'CHARSET': 'utf8',
           },
       },
   }

.. note::

  `TEST/CHARSET` is used when creating tables for the test suite.  Without it,
  tests checking for the correct handling of non-ASCII characters fail. It is
  not necessary if you don't plan to run tests, however.

Static Files
^^^^^^^^^^^^

While we have not yet configured our proxy server, we need to configure the
location that these files will be stored in. We will install these under
``/var/www/patchwork``, though this is only a suggestion and can be changed.

.. code-block:: shell

   $ sudo mkdir -p /var/www/patchwork

You can configure this by overriding the ``STATIC_ROOT`` variable with the
below:

.. code-block:: shell

   STATIC_ROOT = '/var/www/patchwork'

Other Options
^^^^^^^^^^^^^

Finally, the following settings need to be configured and the appropriate
setting overridden. The purpose of many of these variables is described in
:doc:`configuration`.

* ``SECRET_KEY``
* ``ADMINS``
* ``TIME_ZONE``
* ``LANGUAGE_CODE``
* ``DEFAULT_FROM_EMAIL``
* ``NOTIFICATION_FROM_EMAIL``

You can generate the ``SECRET_KEY`` with the following Python code:

.. code-block:: python

   import string, random
   chars = string.ascii_letters + string.digits + string.punctuation
   print(repr("".join([random.choice(chars) for i in range(0,50)])))

If you wish to enable the XML-RPC API, you should add the following:

.. code-block:: python

   ENABLE_XMLRPC = True

Finally, should you wish to disable the REST API, you should add the following:

.. code-block:: python

   ENABLE_REST_API = False

Final Steps
~~~~~~~~~~~

Once done, we should be able to check that all requirements are met using the
``check`` command of the ``manage.py`` executable:

.. code-block:: shell

    $ python3 manage.py check

We should also take this opportunity to both configure the database and static
files:

.. code-block:: shell

   $ python3 manage.py migrate
   $ sudo python3 manage.py collectstatic
   $ python3 manage.py loaddata default_tags default_states

.. note::

   The above ``default_tags`` and ``default_states`` fixtures above are just
   that: defaults. You can modify these to fit your own requirements.

Finally, it may be helpful to start the development server quickly to ensure
you can see *something*. For this to function, you will need to add the
``ALLOWED_HOSTS`` and ``DEBUG`` settings to your settings file.

.. code-block:: python

   ALLOWED_HOSTS = ['*']
   DEBUG = True

Now, run the server.

.. code-block:: shell

   $ python3 manage.py runserver 0.0.0.0:8000

Browse this instance at ``http://[your_server_ip]:8000``. If everything is
working, kill the development server using :kbd:`Control-c` and remove
``ALLOWED_HOSTS`` and ``DEBUG``.

Reverse Proxy and WSGI HTTP Servers
-----------------------------------

Install Packages
~~~~~~~~~~~~~~~~

We will use `nginx` and `uWSGI` to deploy Patchwork, acting as reverse proxy
server and WSGI HTTP server respectively. Other options are available, such as
`Apache` with the `mod_wsgi` module, or `nginx` with the `Gunicorn` WSGI HTTP
server. While we don't document these, sample configuration files for the
former case are provided in `lib/apache2/`.

.. code-block:: shell

   $ sudo apt-get install -y nginx-full uwsgi uwsgi-plugin-python3

Configure nginx and uWSGI
~~~~~~~~~~~~~~~~~~~~~~~~~

Configuration files for `nginx` and `uWSGI` are provided in the `lib`
subdirectory of the Patchwork source code. These can be modified as necessary,
but for now we will simply copy them.

First, let's load the provided configuration for `nginx`:

.. code-block:: shell

   $ sudo cp /opt/patchwork/lib/nginx/patchwork.conf \
       /etc/nginx/sites-available/

If you wish to modify this configuration, now is the time to do so. Once done,
validate and enable your configuration:

.. code-block:: shell

   $ sudo ln -s /etc/nginx/sites-available/patchwork.conf \
       /etc/nginx/sites-enabled/patchwork.conf
   $ sudo nginx -t

If you see a "duplicate default server" error message, You may need to disable
the ``default`` application at this point:

.. code-block:: shell

   $ sudo unlink /etc/nginx/sites-enabled/default
   $ sudo nginx -t

Now, use the provided configuration for `uWSGI`:

.. code-block:: shell

   $ sudo mkdir -p /etc/uwsgi/sites
   $ sudo cp /opt/patchwork/lib/uwsgi/patchwork.ini \
       /etc/uwsgi/sites/patchwork.ini

.. note::

   We created the ``/etc/uwsgi`` directory above because we're going to run
   `uWSGI` in `emperor mode`__. This has benefits for multi-app deployments.

__ https://uwsgi-docs.readthedocs.io/en/latest/Emperor.html

Create systemd Unit File
~~~~~~~~~~~~~~~~~~~~~~~~

As things stand, `uWSGI` will need to be started manually every time the system
boots, in addition to any time it may fail. We can automate this process using
`systemd`. To this end a `systemd unit file`__ should be created to start
`uWSGI` at boot:

.. code-block:: shell

   $ sudo cat << EOF > /etc/systemd/system/uwsgi.service
   [Unit]
   Description=uWSGI Emperor service

   [Service]
   ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown www-data:www-data /run/uwsgi'
   ExecStart=/usr/bin/uwsgi --emperor /etc/uwsgi/sites
   Restart=always
   KillSignal=SIGQUIT
   Type=notify
   NotifyAccess=all

   [Install]
   WantedBy=multi-user.target
   EOF

You should also delete the default service file found in ``/etc/init.d`` to
ensure the unit file defined above is used.

.. code-block:: shell

   sudo rm /etc/init.d/uwsgi
   sudo systemctl daemon-reload

__ https://uwsgi-docs.readthedocs.io/en/latest/Systemd.html

.. _deployment-final-steps:

Final Steps
~~~~~~~~~~~

Start the `uWSGI` service we created above:

.. code-block:: shell

   $ sudo systemctl restart uwsgi
   $ sudo systemctl status uwsgi
   $ sudo systemctl enable uwsgi

Next up, restart the `nginx` service:

.. code-block:: shell

   $ sudo systemctl restart nginx
   $ sudo systemctl status nginx
   $ sudo systemctl enable nginx

Finally, browse to the instance using your browser of choice. You may wish to
take this opportunity to setup your projects and configure your website address
(in the Sites section of the admin console, found at `/admin`).

If there are issues with the instance, you can check the logs for `nginx` and
`uWSGI`. There are a couple of commands listed below which can help:

- ``sudo systemctl status uwsgi``, ``sudo systemctl status nginx``

  To ensure the services have correctly started

- ``sudo cat /var/log/nginx/error.log``

  To check for issues with `nginx`

- ``sudo cat /var/log/patchwork.log``

  To check for issues with `uWSGI`. This is the default log location set by the
  ``daemonize``  setting in the `uWSGI` configuration file.

Django administrative console
-----------------------------

In order to access the administrative console at `/admin`, you need at least
one user account to be registered and configured as a super user or staff
account to access the Django administrative console.  This can be achieved by
doing the following:

.. code-block:: shell

   $ python3 manage.py createsuperuser

Once the administrative console is accessible, you would want to configure your
different sites and their corresponding domain names, which is required for the
different emails sent by Patchwork (registration, password recovery) as well as
the sample `pwclientrc` files provided by your project's page.

.. _deployment-parsemail:

Incoming Email
--------------

Patchwork is designed to parse incoming mails which means you need an address
to receive email at. This is a problem that has been solved for many web apps,
thus there are many ways to go about this. Some of these ways are discussed
below.

IMAP/POP3
~~~~~~~~~

The easiest option for getting mail into Patchwork is to use an existing email
address in combination with a mail retriever like `getmail`__, which will
download mails from your inbox and pass them to Patchwork for processing.
getmail is easy to set up and configure: to begin, you need to install it:

.. code-block:: shell

   $ sudo apt-get install -y getmail4

Once installed, you should configure it, substituting your own configuration
details where required below:

.. code-block:: shell

   $ sudo cat << EOF > /etc/getmail/user@example.com/getmailrc
   [retriever]
   type = SimpleIMAPSSLRetriever
   server = imap.example.com
   port = 993
   username = XXX
   password = XXX
   mailboxes = ALL

   [destination]
   # we configure Patchwork as a "mail delivery agent", in that it will
   # handle our mails
   type = MDA_external
   path = /opt/patchwork/patchwork/bin/parsemail.sh

   [options]
   # retrieve only new emails
   read_all = false
   # do not add a Delivered-To: header field
   delivered_to = false
   # do not add a Received: header field
   received = false
   EOF

Validate that this works as expected by starting `getmail`:

.. code-block:: shell

   $ getmail --getmaildir=/etc/getmail/user@example.com --idle INBOX

If everything works as expected, you can create a `systemd` script to ensure
this starts on boot:

.. code-block:: shell

   $ sudo cat << EOF > /etc/systemd/system/getmail.service
   [Unit]
   Description=Getmail for user@example.com

   [Service]
   User=nobody
   ExecStart=/usr/bin/getmail --getmaildir=/etc/getmail/user@example.com --idle INBOX
   Restart=always

   [Install]
   WantedBy=multi-user.target
   EOF

And start the service:

.. code-block:: shell

   $ sudo systemctl start getmail
   $ sudo systemctl status getmail
   $ sudo systemctl enable getmail

__ http://pyropus.ca/software/getmail/

Mail Transfer Agent (MTA)
~~~~~~~~~~~~~~~~~~~~~~~~~

The most flexible option is to configure our own mail transfer agent (MTA) or
"email server". There are many options, of which `Postfix`__ is one.  While we
don't cover setting up Postfix here (it's complicated and there are many guides
already available), Patchwork does include a script to take received mails and
create the relevant entries in Patchwork for you. To use this, you should
configure your system to forward all emails to a given localpart (the bit
before the `@`) to this script. Using the `patchwork` localpart (e.g.
`patchwork@example.com`) you can do this like so:

.. code-block:: shell

   $ sudo cat << EOF > /etc/aliases
   patchwork: "|/opt/patchwork/patchwork/bin/parsemail.sh"
   EOF

You should ensure the appropriate user is created in PostgreSQL and that it has
(minimal) access to the database. Patchwork provides scripts for the latter and
they can be loaded as seen below:

.. code-block:: shell

   $ sudo -u postgres createuser nobody
   $ sudo -u postgre psql -f \
       /opt/patchwork/lib/sql/grant-all.postgres.sql patchwork

.. note::

   This assumes your Postfix process is running as the `nobody` user.  If this
   is not correct (use of `postfix` user is also common), you should change
   both the username in the `createuser` command above and substitute the
   username in the `grant-all-postgres.sql` script with the appropriate
   alternative.

__ http://www.postfix.org/

Use a Email-as-a-Service Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Setting up an email server can be a difficult task and, in the case of
deployment on PaaS provider, may not even be an option. In this case, there
are a variety of web services available that offer "Email-as-as-Service".
These services typically convert received emails into HTTP POST requests to
your endpoint of choice, allowing you to sidestep configuration issues. We
don't cover this here, but a simple wrapper script coupled with one of these
services can be more than to get email into Patchwork.

You can also create such as service yourself using a PaaS provider that
supports incoming mail and writing a little web app.

.. _deployment-vcs:

(Optional) Configure your VCS to Automatically Update Patches
-------------------------------------------------------------

The `tools` directory of the Patchwork distribution contains a file named
`post-receive.hook` which is a sample Git hook that can be used to
automatically update patches to the `Accepted` state when corresponding
commits are pushed via Git.

To install this hook, simply copy it to the `.git/hooks` directory on your
server, name it `post-receive`, and make it executable.

This sample hook has support to update patches to different states depending
on which branch is being pushed to. See the `STATE_MAP` setting in that file.

If you are using a system other than Git, you can likely write a similar hook
using `pwclient` to update patch state. If you do write one, please contribute
it.

.. _deployment-cron:

(Optional) Configure the Patchwork Cron Job
-------------------------------------------

Patchwork can send notifications of patch changes. Patchwork uses a cron
management command - ``manage.py cron`` - to send these notifications and to
clean up expired registrations. To enable this functionality, add the following
to your crontab::

   # m h  dom mon dow   command
   */10 * * * * cd patchwork; python3 ./manage.py cron

.. note::

   The frequency should be the same as the ``NOTIFICATION_DELAY_MINUTES``
   setting, which defaults to 10 minutes. Refer to the :doc:`configuration
   guide <configuration>` for mor information.
