Getting Started
===============
This getting started guide will provide a quick tour of some CloudBridge
features. For more details on individual features, see the
`Using CloudBridge <topics/overview.html>`_ section or the
`API reference <api_docs/ref.html>`_.

Installation
------------
CloudBridge is available on PyPI so to install the latest available version,
run::

    pip install --upgrade cloudbridge

Create a provider
-----------------
To start, you will need to create a reference to a provider object. The
provider object identifies the cloud you want to work with and supplies your
credentials. The following two code snippets setup a necessary provider object,
for AWS and OpenStack. For the details on other providers, take a look at the
`Setup page <topics/setup.html>`_. The remainder of the code is the same for
either provider.

AWS:

.. code-block:: python

    from cloudbridge.cloud.factory import CloudProviderFactory, ProviderList

    config = {'aws_access_key': 'AKIAJW2XCYO4AF55XFEQ',
              'aws_secret_key': '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'}
    provider = CloudProviderFactory().create_provider(ProviderList.AWS, config)
    image_id = 'ami-2d39803a'  # Ubuntu 14.04 (HVM)

OpenStack (with Keystone authentication v2):

.. code-block:: python

    from cloudbridge.cloud.factory import CloudProviderFactory, ProviderList

    config = {'os_username': 'username',
              'os_password': 'password',
              'os_auth_url': 'authentication URL',
              'os_region_name': 'region name',
              'os_project_name': 'project name'}
    provider = CloudProviderFactory().create_provider(ProviderList.OPENSTACK,
                                                      config)
    image_id = 'c1f4b7bc-a563-4feb-b439-a2e071d861aa'  # Ubuntu 14.04 @ NeCTAR

OpenStack (with Keystone authentication v3):

.. code-block:: python

    from cloudbridge.cloud.factory import CloudProviderFactory, ProviderList

    config = {'os_username': 'username',
              'os_password': 'password',
              'os_auth_url': 'authentication URL',
              'os_project_name': 'project name',
              'os_project_domain_name': 'project domain name',
              'os_user_domain_name': 'domain name'}
    provider = CloudProviderFactory().create_provider(ProviderList.OPENSTACK,
                                                      config)
    image_id = '97755049-ee4f-4515-b92f-ca00991ee99a'  # Ubuntu 14.04 @ Jetstream

List some resources
-------------------
Once you have a reference to a provider, explore the cloud platform:

.. code-block:: python

    provider.compute.images.list()
    provider.security.security_groups.list()
    provider.block_store.snapshots.list()
    provider.object_store.list()

This will demonstrate the fact that the library was properly installed and your
provider object is setup correctly but it is not very interesting. Therefore,
let's create a new instance we can ssh into using a key pair.

Create a key pair
-----------------
We'll create a new key pair and save the private portion of the key to a file
on disk as a read-only file.

.. code-block:: python

    kp = provider.security.key_pairs.create('cloudbridge_intro')
    with open('cloudbridge_intro.pem', 'w') as f:
        f.write(kp.material)
    import os
    os.chmod('cloudbridge_intro.pem', 0400)

Create a security group
-----------------------
Next, we need to create a security group and add a rule to allow ssh access.
A security group needs to be associated with a private network, so we'll also
need to fetch it.

.. code-block:: python

    provider.network.list()  # Find a desired network ID
    net = provider.network.get('desired network ID')
    sg = provider.security.security_groups.create(
        'cloudbridge_intro', 'A security group used by CloudBridge', net.id)
    sg.add_rule('tcp', 22, 22, '0.0.0.0/0')

Launch an instance
------------------
We can now launch an instance using the created key pair and security group.
We will launch an instance type that has at least 2 CPUs and 4GB RAM. We will
also add the network interface as a launch argument.

.. code-block:: python

    img = provider.compute.images.get(image_id)
    inst_type = sorted([t for t in provider.compute.instance_types.list()
                        if t.vcpus >= 2 and t.ram >= 4],
                       key=lambda x: x.vcpus*x.ram)[0]
    inst = provider.compute.instances.create(
        name='CloudBridge-intro', image=img, instance_type=inst_type,
        network=net, key_pair=kp, security_groups=[sg])
    # Wait until ready
    inst.wait_till_ready()  # This is a blocking call
    # Show instance state
    inst.state
    # 'running'

Assign a public IP address
--------------------------
To access the instance, let's assign a public IP address to the instance. For
this step, we'll first need to allocate a floating IP address for our account
and then associate it with the instance.

.. code-block:: python

    fip = provider.network.create_floating_ip()
    inst.add_floating_ip(fip.public_ip)
    inst.refresh()
    inst.public_ips
    # [u'54.166.125.219']

From the command prompt, you can now ssh into the instance
``ssh -i cloudbridge_intro.pem ubuntu@54.166.125.219``.

Cleanup
-------
To wrap things up, let's clean up all the resources we have created

.. code-block:: python

    inst.terminate()
    from cloudbridge.cloud.interfaces import InstanceState
    inst.wait_for([InstanceState.TERMINATED, InstanceState.UNKNOWN],
                   terminal_states=[InstanceState.ERROR])  # Blocking call
    fip.delete()
    sg.delete()
    kp.delete()
    os.remove('cloudbridge_intro.pem')
    router.remove_route(sn.id)
    router.detach_network()
    router.delete()
    sn.delete()
    net.delete()

And that's it - a full circle in a few lines of code. You can now try
the same with a different provider. All you will need to change is the
cloud-specific data, namely the provider setup and the image ID.
