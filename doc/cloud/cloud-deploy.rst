Deploy
======

This section will help you to deploy the KNoT Cloud.

----------------------------------------------------------------

Installation and usage
----------------------

Stacks for development and production environments are provided in order to
assist the user's needs. The development stack must be used if one needs to
modify any of the components of the stack. A command line tool will download
the source for each component and plug it into the containers, which all have
*hot reload* enabled. The production stack must be used in all other cases.

Download
''''''''

The provided stacks and CLI contained in this repository aren't yet published
in any package manager, hence it is necessary that you clone the repository or
download the ``.zip`` containing all the files.

   .. code-block:: bash

        git clone https://github.com/CESARBR/knot-cloud.git

The following instructions always assume you are in the directory created
after cloning the repository. If you downloaded the ``.zip``, navigate to
the appropriate folder.

----------------------------------------------------------------

Development only preparation
----------------------------

If you intend to use the development stack, a command line tool that
downloads and configures the stack is required.

Build and install CLI tool
''''''''''''''''''''''''''

The tool is built and installed as follows:

   .. code-block:: bash

        npm install
        npm run build
        npm link

Depending on npm configurations, it might be necessary to run ``npm link``
with super user privileges.

   .. code-block:: bash

        sudo npm link

----------------------------------------------------------------

Choose the stack
----------------

Development
'''''''''''

The development stack must be created using the tool described in the previous
section. Choose a ``<path>`` where the stack should be created
(defaults to the current directory) and then run:

   .. code-block:: bash

        knot-cloud init [path]

The source code and stack template files will be created under ``<path>/stack``.

Production
''''''''''

The production stack comes in two flavours: all-in-one and multinode. The
former will deploy all the services on a single machine while the latter will
deploy them in multiple nodes, at least two. The files for the two flavours
are available at ``stacks/prod``.

----------------------------------------------------------------

Initialize Swarm mode
---------------------

In your deployment machine, initialize Docker Swarm mode:

   .. code-block:: bash

        docker swarm init

In case you are deploying to multiple nodes, all the nodes must be connected
to the same swarm. In this case, run the command above in the machine that
must be the swarm manager and run ``docker swarm join`` in all other nodes, as
instructed by the execution of ``docker swarm init`` in the manager machine.
Check the `Docker Swarm <https://docs.docker.com/engine/swarm/>`_ documentation
for more on how to set up your cluster.

----------------------------------------------------------------

Configure DNS
-------------

This stack uses `Traefik <https://traefik.io>`_ as a reverse proxy for the
services and requires that you configure your DNS server to point, at least,
the **www**, **ws**, **auth** and **bootstrap** subdomains to the machine
where the stack is being deployed, so that it can route the requests to the
appropriated service. It is possible to configure it to route by path or port,
but instructions for that won't be provided here for brevity.

If you don't have a domain or can't configure the main DNS server, you can
configure a test domain in your machine before proceeding. Either set up a
local DNS server, e.g. `bind9 <https://wiki.debian.org/Bind9>`_, or
alternatively update your hosts file to include the following addresses:

   .. code-block:: text

        127.0.0.1	www
        127.0.0.1	ws
        127.0.0.1	auth
        127.0.0.1	bootstrap

On Windows, the hosts file is usually located under
``c:\Windows\System32\Drivers\etc\hosts``. On Unix systems, it is commonly
found at ``/etc/hosts``. Regardless of your operating system, administrator or
super user privileges will be required.

Notice that when deploying KNoT Cloud locally, most of the times the
``<your-domain>`` subdomains in the following sections should be
disregarded. For instance, you would access your KNoT Cloud at
``https://www`` after deploy.

----------------------------------------------------------------

Deploy: stage 1
---------------

Stage 1 contains the core services. The next sections provide the instructions
to configure and deploy them. Whenever a configuration file is mentioned, it
refers to a file found at ``stacks/prod/env.d``, for production mode, or
``<path>/stack/dev/env.d``, for development mode.

Configure services
''''''''''''''''''

Create, if you already don't have, a private/public key pair:

   .. code-block:: bash

        openssl genpkey -algorithm RSA -out privateKey.pem -pkeyopt rsa_keygen_bits:2048
        openssl rsa -pubout -in privateKey.pem -out publicKey.pem

Then, convert the keys to base 64:

   .. code-block:: bash

        base64 < privateKey.pem
        base64 < publicKey.pem

And generate a 16-bit random value in hexadecimal format:

   .. code-block:: bash

        openssl rand -hex 16

Finally, set ``TOKEN``, ``PRIVATE_KEY_BASE64`` and ``PUBLIC_KEY_BASE64`` to
the values above in:

- ``meshblu-core-dispatcher.env``
- ``meshblu-core-worker-webhook.env``
- ``knot-cloud-storage.env``

Deploy
''''''

Deploy the stage 1 services:

   .. code-block:: bash

        docker stack deploy -c stage-1.yml knot-cloud

Verify
''''''

Wait until all the services are started. You can check it by running:

   .. code-block:: bash

        docker stack services knot-cloud

And verifying that every service has one replica.

----------------------------------------------------------------

Deploy: stage 2 bootstrap
-------------------------

Before bringing the stage 2 services up, a bootstrap process must be executed
in the stage 1 services. The next sections provide the instructions to execute
this process.

Deploy
''''''

Deploy the stage 2 bootstrap services:

   .. code-block:: bash

        docker stack deploy -c stage-2-bootstrap.yml knot-cloud

Wait until the bootstrap service is responsive, when the following command
should succeed:

   .. code-block:: bash

        curl https://bootstrap.<your domain>/healthcheck

**Note:** If the HTTPS certificates are not configured locally, traefik has a
default certificate that is used in such cases. To use traefik's default
certificate, it is necessary to add ``-k`` parameter to the ``curl``
command otherwise the request will fail.

Bootstrap
'''''''''

Once the services are started, run the bootstrap process:

   .. code-block:: bash

        curl -X POST https://bootstrap.<your domain>/bootstrap

Save the output for the next steps.

**Note:** If the HTTPS certificates are not configured locally, traefik has a
default certificate that is used in such cases. To use traefik's default certificate,
it is necessary to add ``-k`` parameter to the ``curl`` command otherwise the request
will fail.

Tear down
'''''''''

List the stack services:

   .. code-block:: bash

        docker stack services knot-cloud

Remove the bootstrap service (get the name from the list above, probably will
be ``knot-cloud_boostrap``):

   .. code-block:: bash

        docker service rm <bootstrap-service-name>

----------------------------------------------------------------

Deploy: stage 2
---------------

The stage 2 is the last stage, in which the user authentication service and
the configuration UI are started. The next sections provide the instructions
to configure and deploy them.

Configure services
''''''''''''''''''

Get the authenticator's UUID and token from the bootstrap output and set
``AUTHENTICATOR_UUID`` and ``AUTHENTICATOR_TOKEN`` variables in
``knot-cloud-authenticator.env``.

Configure mail service
''''''''''''''''''''''

KNoT Cloud supports several mail services and is built in such a modular
fashion that it is rather trivial to include a new one. Deploying KNoT Cloud
without a mail service is also allowed, although not recommended other than
for testing purposes.

The supported mail services and related environment variables are:

- Disable mail service
    ``MAIL_SERVICE=NONE``
- Mailgun
    .. code-block:: bash

        MAIL_SERVICE=MAILGUN
        MAILGUN_DOMAIN=<your_mailgun_domain>
        MAILGUN_API_KEY=<you_mailgun_api_key>
    Where ``MAILGUN_DOMAIN`` and ``MAILGUN_API_KEY`` are your
    `Mailgun <https://mailgun.com>`_'s account domain and API key.

- AWS SES
    .. code-block:: bash

        MAIL_SERVICE=AWS-SES
        AWS_REGION=<aws_user_region>
        AWS_ACCESS_KEY_ID=<aws_user_acess_key_id>
        AWS_SECRET_ACCESS_KEY=<aws_user_secret_acess_key>

    Where ``AWS_REGION``, ``AWS_ACCESS_KEY_ID`` and ``AWS_SECRET_ACCESS_KEY``
    are your `Amazon Web Services <https://aws.amazon.com/>`_ account
    information.

    .. note::
        If you are running KNoT Cloud on AWS EC2 that is using
        `roles togrant permissions <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html/>`_,
        it is not necessary to set ``AWS_ACCESS_KEY_ID`` and
        ``AWS_SECRET_ACCESS_KEY``. The role attached to the EC2 instance must
        include a policy that allows ``ses:SendEmail`` actions at least on the
        domain used to send the reset e-mail (see ``RESET_SENDER_ADDRESS`` below).
        For more information on AWS SES policies, refer to their
        `documentation <https://docs.aws.amazon.com/ses/latest/DeveloperGuide/control-user-access.html>`_.

Configure reset address
'''''''''''''''''''''''

Set ``RESET_SENDER_ADDRESS`` with the e-mail address that will send the reset
password e-mails.
If this stack is being deployed on an accessible domain, replaced ``RESET_URI``
with **http://<your-domain>/reset**. This is the reset password address
that is going to be sent by e-mail.

This is a **required** option, but you could fill using a bogus e-mail address
if it will not be used or you have set the mail service to ``NONE``.

Deploy
''''''

Deploy the stage 2 services

   .. code-block:: bash

        docker stack deploy -c stage-2.yml knot-cloud

Access
''''''

Access KNoT Cloud in your browser at ``https://www.<your-domain>``.