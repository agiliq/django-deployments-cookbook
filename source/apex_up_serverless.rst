Using Apex-Up deploy in Lambda and use Aurora Serverless
----------------------------------------

We will try to deploy a basic django app onto **AWS Lambda** using
**Apex Up**.

`AWS Lambda <https://aws.amazon.com/lambda/>`__ is a serverless
computing platform by amazon, which is completely event driven and it
automatically manages the computing resources. It scales automatically
when needed, depending upon the requests the application gets.

`Apex Up <https://up.docs.apex.sh/>`__ is a Open Source framework
used for deploying serverless applications onto AWS-Lambda. Up currently
supports Node.js, Golang, Python, Java, Crystal, and static sites out of
the box. Up is platform-agnostic, supporting AWS Lambda and API Gateway.

**Note** :

-  **Apex-Up currently supports only Node.js lambda environment**, but
   we can use python 2.7 and 3.4 in it.

-  **We have to use Django 2.0 as it is the only latest version which
   supports python3.4**

Install and Configure the Environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First configure the AWS credentials

`https://books.agiliq.com/projects/django-deployments-cookbook/en/latest/using_zappa_lambda_aurora.html#configure-aws-credentials  <https://books.agiliq.com/projects/django-deployments-cookbook/en/latest/using_zappa_lambda_aurora.html#configure-aws-credentials>`_.


Install Apex Up
^^^^^^^^^^^^^^^

Currently *Up* has only `binary form
releases <https://github.com/apex/up/releases>`__ and can be installed
by

.. code:: sh

    $ curl -sf https://up.apex.sh/install | sh

this installs *Up* in ``/usr/local/bin`` by default.

We can verify the installation by

.. code:: sh

    $ up version

    # or 

    $ up --help

Go to Django app
^^^^^^^^^^^^^^^^

We will use *Pollsapi* (https://github.com/agiliq/building-api-django)
as the django project.

**Note**: **We cannot see the django error messages in the url(even if
we have DEBUG=True), we can see them in the apex-up logs only**

Now go inside the *pollsapi* app in this repo.

Next create a virtualenv with python34 and install ``requirements.txt``

.. code:: sh

    $ pip install -r requirements.txt

.. code:: sh

    $ django-admin --version        # check the django version
    2.0.3

Now rename the ``manage.py`` to ``app.py`` for *apex-up* to work.

.. code:: sh

    $ python  app.py runserver

which will show us

.. figure:: /_static/images/apex-up/django.png
   :alt: 

and in ``polls/settings.py`` add aws subdomain to the 'ALLOWED\_HOSTS'

.. code:: py

    ...
    ALLOWED_HOSTS = [".amazonaws.com", "127.0.0.1"]  # lambda subdomain and localhost
    ...

Serving Static Files
^^^^^^^^^^^^^^^^^^^^

To configure static files in django
https://www.agiliq.com/blog/2019/01/complete-serverless-django/#serving-static-files

Setup Serverless MySQL Database
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To set up Aurora serverless DB follow
https://www.agiliq.com/blog/2019/01/complete-serverless-django/#setup-serverless-mysql-database

Connect Our App to MySQL DB
^^^^^^^^^^^^^^^^^^^^^^^^^^^

To connect our Django App to aurora db, follow
https://www.agiliq.com/blog/2019/01/complete-serverless-django/#connect-django-to-mysql-db

After configuring our ``settings.py`` file should have a similar
database config

.. code:: py

    ...

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'pollsdb', # dbname
            'USER': 'polls_admin', # master username
            'PASSWORD': 'pollsadmin', # master password
            'HOST': 'pollsapi-cluster.cluster-chcxxxxx.us-east-2.rds.amazonaws.com', # Endpoint
            'PORT': '3306',
        }
    }
    ...

Now create a file in the same level as the ``app.py`` file named
**``up.json``** and add the following lines

.. code:: json

    {
      "name": "pollsapi",
      "profile": "default",
      "regions": [
        "us-east-2"
      ],
      "proxy": {
        "command": "python3 app.py runserver 0.0.0.0:$PORT"
      }
    }

here ``name`` is the name of the project to be deployed

``profile`` is the aws credentials profile name

``region`` is the region of the lambda function

``proxy`` acts as a reverse proxy in front of our server, which
provides features like CORS, redirection, script injection and
middleware style features.

We have to include the following configuration to our proxy object

Add ``command`` Command run through the shell to start our server
(Default ./server)

In the proxy command we have to give the command to start the django
server ie *runserver* .

As presently *Up* supports only Node.js lambda runtime environment, but
we can use python 2.7 and 3.4 in it. So we can use python3 by mentioning
the command as ``python3 app.py runserver 0.0.0.0:$PORT`` where the
``$PORT`` is the port where our app runs(which is generated
dynamically).

for more configuration settings like using custom domains, secrets,
deploying to multiple AWS regions or multiple stages(test/staging/prod
etc) check the `docs <https://up.docs.apex.sh/#configuration>`__

Now let us test the app by deploying it,

.. code:: sh

    $ up
    # or
    $ up deploy
    # or
    $ up -v         # verbose

.. code:: sh

    $ up

         build: 4,752 files, 16 MB (9.463s)
         deploy: staging (commit 3asdfjj) (17.103s)
         stack: complete (26.324s)
         endpoint: https://Xpiix0c1.execute-api.us-east-2.amazonaws.com/staging/

         Please consider subscribing to Up Pro for additional features and to help keep the project alive!
         Visit https://github.com/apex/up#pro-features for details.

to get the url of the application

.. code:: sh


    $ up url
    # or
    $ up url --open

Now when we open the url, we get

.. figure:: /_static/images/apex-up/drf.png
   :alt: 

The logs can be checked by these commands

.. code:: sh

    $ up logs
    # or
    $ up logs -f            # for live logs

*Up* also sends our logs to AWS cloudwatch, so we can search for the
logs there also.

To run Django Migrations
^^^^^^^^^^^^^^^^^^^^^^^^

We have to add the migrate command to the ``proxy.command`` in the
*up.json* file.

.. code:: json

    {
      "name": "pollsapi",
      "profile": "default",
      "regions": [
        "us-east-2"
      ],
      "proxy": {
        "command": "python3 app.py migrate && python3 app.py runserver 0.0.0.0:$PORT"
      }
    }

Troubleshooting
~~~~~~~~~~~~~~~

**We should note that we cannot see the django error messages in the
url(even if we have DEBUG=True), we can see them in the apex-up logs**

We can check for the errors by

.. code:: sh


    $ up logs error               # Shows error logs.

    $ up logs 'error or fatal'    # Shows error and fatal logs.

    $ up logs 'status >= 400'     # Shows 4xx and 5xx responses.

To delete the deployment

.. code:: sh

    $ up stack delete   # delete the deployment

**We have to note that we have only python 2.7 and python 3.4 versions
available at present in Apex-Up**

--------------


