Using Public Clouds as backends
===============================

Introduction
------------

As stated in our `GETTING STARTED guide <../GETTING_STARTED/#location-configuration>`__,
new backends can be added by creating a region (also called location constraint)
with the right endpoint and credentials.
This section of the documentation shows you how to set up our currently
supported public cloud backends:

- `Amazon S3 <#aws-s3-as-a-backend>`__ ;
- `Microsoft Azure <#microsoft-azure-as-a-backend>`__ .

For all Public Cloud backends, you'll have to do some editing inside the
repository, and some editing from the Public Cloud Console.

AWS S3 as a backend
-------------------

From the AWS S3 Console
~~~~~~~~~~~~~~~~~~~~~~~

Create a bucket where you will host your data for this new location constraint.
This bucket must have versioning enabled. This is an option you may choose to
activate at step 2 of Bucket Creation in the console.
In this example, our bucket will be name ``zenkobucket`` and has versioning
enabled.

From the CloudServer repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

locationConfig.json
^^^^^^^^^^^^^^^^^^^

Edit this file to add a new region, which should describe the AWS S3 hosted
bucket you'll be writing your data to. There are a few configurable options here:

- :code:`type` : set to :code:`aws_s3` to indicate this region is hosted in AWS S3;
- :code:`legacyAwsBehavior` : set to :code:`true` to indicate this region should
  behave like AWS S3 :code:`us-east-1` region, set to :code:`false` to indicate
  this region should behave like any other AWS S3 region;
- :code:`bucketName` : set to an *existing bucket* in your AWS S3 Account; this
  is the bucket that will host all the data written in this region;
- :code:`awsEndpoint` : set to your bucket's endpoint, usually :code:`s3.amazonaws.com`;
- :code:`bucketMatch` : set to :code:`true` if you want your object name to be the
  same in your local bucket and your AWS S3 bucket; set to :code:`false` if you
  want your object name to be of the form :code:`{{localBucketName}}/{{objectname}}`
  in your AWS S3 hosted bucket;
- :code:`credentialsProfile` and :code:`credentials` are two ways to provide
  your AWS S3 credentials for that bucket, *use only one of them* :
  - :code:`credentialsProfile` : set to the profile name allowing you to access
    your AWS S3 bucket from your :code:`~/.aws/credentials` file;
  - :code:`credentials` : set the two fields inside the object (:code:`accessKey`
    and :code:`secretKey`) to their respective values from your AWS credentials.

.. code:: json

    (...)
    "aws-test": {
        "type": "aws_s3",
        "legacyAwsBehavior": true,
        "details": {
            "awsEndpoint": "s3.amazonaws.com",
            "bucketName": "zenkobucket",
            "bucketMatch": true,
            "credentialsProfile": "zenko"
        }
    },
    (...)

.. code:: json

    (...)
    "aws-test": {
        "type": "aws_s3",
        "legacyAwsBehavior": true,
        "details": {
            "awsEndpoint": "s3.amazonaws.com",
            "bucketName": "zenkobucket",
            "bucketMatch": true,
            "credentials": {
                "accessKey": "WHDBFKILOSDDVF78NPMQ",
                "secretKey": "87hdfGCvDS+YYzefKLnjjZEYstOIuIjs/2X72eET"
            }
        }
    },
    (...)

.. WARNING::
   If you set :code:`bucketMatch` to :code:`true`, we strongly advise for having
   only one local bucket per AWS S3 location. Indeed, since your objects names
   won't be prefixed by the local bucket name in the AWS S3 bucket, you could
   create inconsistencies seamlessly by putting two objects with the same name
   in two different local buckets, but since they are ultimately saved in the
   same AWS bucket, the most recent one would overwrite the earlier one, as the
   namespace will conflict.

config.json
^^^^^^^^^^^

Edit the :code:`restEndpoint` section of your :code:`config.json` file to add
an endpoint definition matching your new AWS S3 hosted region. Following our
previous example, it would look like:

.. code:: json

    (...)
        "restEndpoints": {
        "localhost": "us-east-1",
        "127.0.0.1": "us-east-1",
        "cloudserver-front": "us-east-1",
        "s3.docker.test": "us-east-1",
        "127.0.0.2": "us-east-1",
        "s3.amazonaws.com": "aws-test"
    },
    (...)

~/.aws/credentials
^^^^^^^^^^^^^^^^^^

.. TIP::
   If you set the :code:`credentials` object in your
   :code:`locationConfig.json` file, you may skip this section

Make sure your :code:`~/.aws/credentials` file has a profile matching the one
defined in your :code:`locationConfig.json`. Following our previous example, it
would look like:


.. code:: shell

    [zenko]
    aws_access_key_id=WHDBFKILOSDDVF78NPMQ
    aws_secret_access_key=87hdfGCvDS+YYzefKLnjjZEYstOIuIjs/2X72eET

Start the server with the ability to write to AWS S3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Inside the repository, once all the files have been edited, you should be able
to start the server and start testing pushing to AWS S3.

.. code:: shell

   # Start the server locally
   $> S3DATA=multiple npm start

Run the server as a docker container with the ability to write to AWS S3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. TIP::
   If you set the :code:`credentials` object in your
   :code:`locationConfig.json` file, you don't need to mount your
   :code:`.aws/credentials` file

Mount all the files that have been edited to override defaults, and do a
standard Docker run; then you can start testing pushing to AWS S3.

.. code:: shell

   # Start the server in a Docker container
   $> sudo docker run -d --name CloudServer \
   -v $(pwd)/data:/usr/src/app/localData \
   -v $(pwd)/metadata:/usr/src/app/localMetadata \
   -v $(pwd)/locationConfig.json:/usr/src/app/locationConfig.json \
   -v $(pwd)/conf/authdata.json:/usr/src/app/conf/authdata.json \
   -v ~/.aws/credentials:/root/.aws/credentials \
   -e S3DATA=multiple -e ENDPOINT=http://localhost -p 8000:8000
   -d scality/s3server

Testing: put an object to AWS S3 using CloudServer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to start testing pushing to AWS S3, you will need to create a local
bucket in the AWS S3 location constraint - this local bucket will only store the
metadata locally, while both the data and the metadata will be stored on AWS S3.
This example is based on all our previous steps.

.. code:: shell

   # Create a local bucket hosting data in AWS S3
   $> s3cmd --host=127.0.0.1:8000 mb s3://zenkobucket --region=aws-test
   # Put an object to AWS S3, and store the metadata locally
   $> s3cmd --host=127.0.0.1:8000 put /etc/hosts s3://zenkobucket/testput
    upload: '/etc/hosts' -> 's3://zenkobucket/testput'  [1 of 1]
     330 of 330   100% in    0s   380.87 B/s  done
   # List locally to check you have the metadata
   $> s3cmd --host=127.0.0.1:8000 ls s3://zenkobucket
    2017-10-23 10:26       330   s3://zenkobucket/testput

Then, from the AWS Console, if you go into your bucket, you should see your
newly uploaded object:

.. figure:: ../res/aws-console-successful-put.png
   :alt: AWS S3 Console upload example

Troubleshooting
~~~~~~~~~~~~~~~

Make sure your :code:`~/.s3cfg` file has credentials matching your local
CloudServer credentials defined in :code:`conf/authdata.json`. By default, the
access key is :code:`accessKey1` and the secret key is :code:`verySecretKey1`.
For more informations, refer to our template `~/.s3cfg <./CLIENTS/#s3cmd>`__ .

Pre existing objects in your AWS S3 hosted bucket can unfortunately not be
accessed by CloudServer at this time.

Make sure versioning is enabled in your remote AWS S3 hosted bucket. To check,
using the AWS Console, click on your bucket name, then on "Properties" at the
top, and then you should see something like this:

.. figure:: ../res/aws-console-versioning-enabled.png
   :alt: AWS Console showing versioning enabled
