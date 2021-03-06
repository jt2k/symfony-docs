.. index::
   single: Emails; Spooling

How to Spool Emails
===================

The default behavior of the Symfony mailer is to send the email messages
immediately. You may, however, want to avoid the performance hit of the
communication to the email server, which could cause the user to wait for the
next page to load while the email is sending. This can be avoided by choosing to
"spool" the emails instead of sending them directly.

This makes the mailer to not attempt to send the email message but instead save
it somewhere such as a file. Another process can then read from the spool and
take care of sending the emails in the spool. Currently only spooling to file or
memory is supported.

.. _email-spool-memory:

Spool Using Memory
------------------

When you use spooling to store the emails to memory, they will get sent right
before the kernel terminates. This means the email only gets sent if the whole
request got executed without any unhandled exception or any errors. To configure
this spool, use the following configuration:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/swiftmailer.yaml
        swiftmailer:
            # ...
            spool: { type: memory }

    .. code-block:: xml

        <!-- config/packages/swiftmailer.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:swiftmailer="http://symfony.com/schema/dic/swiftmailer"
            xsi:schemaLocation="http://symfony.com/schema/dic/services https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/swiftmailer https://symfony.com/schema/dic/swiftmailer/swiftmailer-1.0.xsd">

            <swiftmailer:config>
                <swiftmailer:spool type="memory" />
            </swiftmailer:config>
        </container>

    .. code-block:: php

        // config/packages/swiftmailer.php
        $container->loadFromExtension('swiftmailer', [
             // ...
            'spool' => ['type' => 'memory'],
        ]);

.. _spool-using-a-file:

Spool Using Files
------------------

When you use the filesystem for spooling, Symfony creates a folder in the given
path for each mail service (e.g. "default" for the default service). This folder
will contain files for each email in the spool. So make sure this directory is
writable by Symfony (or your webserver/php)!

In order to use the spool with files, use the following configuration:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/swiftmailer.yaml
        swiftmailer:
            # ...
            spool:
                type: file
                path: /path/to/spooldir

    .. code-block:: xml

        <!-- config/packages/swiftmailer.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:swiftmailer="http://symfony.com/schema/dic/swiftmailer"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/swiftmailer https://symfony.com/schema/dic/swiftmailer/swiftmailer-1.0.xsd">

            <swiftmailer:config>
                <swiftmailer:spool
                    type="file"
                    path="/path/to/spooldir"
                />
            </swiftmailer:config>
        </container>

    .. code-block:: php

        // config/packages/swiftmailer.php
        $container->loadFromExtension('swiftmailer', [
             // ...

            'spool' => [
                'type' => 'file',
                'path' => '/path/to/spooldir',
            ],
        ]);

.. tip::

    If you want to store the spool somewhere with your project directory,
    remember that you can use the ``%kernel.project_dir%`` parameter to reference
    the project's root:

    .. code-block:: yaml

        path: '%kernel.project_dir%/var/spool'

Now, when your app sends an email, it will not actually be sent but instead
added to the spool. Sending the messages from the spool is done separately.
There is a console command to send the messages in the spool:

.. code-block:: terminal

    $ APP_ENV=prod php bin/console swiftmailer:spool:send

It has an option to limit the number of messages to be sent:

.. code-block:: terminal

    $ APP_ENV=prod php bin/console swiftmailer:spool:send --message-limit=10

You can also set the time limit in seconds:

.. code-block:: terminal

    $ APP_ENV=prod php bin/console swiftmailer:spool:send --time-limit=10

You will most likely not want to run this command manually in reality. Instead, the
console command should be triggered by a cron job or scheduled task and run
at a regular interval.

.. caution::

    When you create a message with SwiftMailer, it generates a ``Swift_Message``
    class. If the ``swiftmailer`` service is lazy loaded, it generates instead a
    proxy class named ``Swift_Message_<someRandomCharacters>``.

    If you use the memory spool, this change is transparent and has no impact.
    But when using the filesystem spool, the message class is serialized in
    a file with the randomized class name. The problem is that this random
    class name changes on every cache clear. So if you send a mail and then you
    clear the cache, the message will not be unserializable.

    On the next execution of ``swiftmailer:spool:send`` an error will raise because
    the class ``Swift_Message_<someRandomCharacters>`` doesn't exist (anymore).

    The solutions are either to use the memory spool or to load the
    ``swiftmailer`` service without the ``lazy`` option (see :doc:`/service_container/lazy_services`).
