Unit tests: mock driver
=======================

In order to easily mock all actions done by a napalm driver, a driver `mock`
has been written. It can be used for unit tests, either to test napalm itself
or inside external projects making use of napalm.


Introduction
------------

For any action, the ``mock`` driver will use a file matching a specific pattern
to return its content as a result.

Each of these files will be located inside a directory specified at the driver
initialization. Their names depend about the entire call name made to the
driver, and about their order in the call stack.


Replacing a standard driver by a ``mock``
-----------------------------------------

Get the driver in napalm::

    >>> import napalm
    >>> driver = napalm.get_network_driver('mock')

And instantiate it with any host and credentials::

    device = driver(
        hostname='foo', username='user', password='pass',
        optional_args={
            'path': path_to_results, 'profile': ('model_mocked', )
        }
    )

Like other drivers, ``mock`` takes some optional arguments:

- ``path`` - Required. Directory where results files are located
- ``profile`` - Device type to mock: ``['ios']``, ``['nxos']``, ``['junos']``,
  etc. Not used for now.

Open the driver::

    >>> device.open()

A user should now be able to call any function of a standard driver::

    >>> device.get_network_instances()

But should get an error because no mocked data is yet written::

    NotImplementedError: You can provide mocked data in get_network_instances.1


Mocked data
-----------

We will use ``/tmp/mock`` as an example of a directory that will contain
our mocked data. Define a device using this path::

    >>> device = driver('foo', 'user', 'pass', optional_args={'path': '/tmp/mock'})
    >>> device.open()

Mock a single call
~~~~~~~~~~~~~~~~~~

In order to be able to call, for example, ``device.get_interfaces()``, a mocked
data is needed.

To build the file name that the driver will look for, take the function name
(``get_interfaces``) and suffix it with the place of this call in the device
call stack.

.. note::
    ``device.open()`` counts as a command. Each following order of call will
    start at 1.

Here, ``get_interfaces`` is the first call made to ``device`` after ``open()``,
so the mocked data need to be put in ``/tmp/mock/get_interfaces.1``::


    {
        "Ethernet1/1": {
            "is_up": true, "is_enabled": true, "description": "",
            "last_flapped": 1478175306.5162635, "speed": 10000,
            "mac_address": "FF:FF:FF:FF:FF:FF"
        },
        "Ethernet1/2": {
            "is_up": true, "is_enabled": true, "description": "",
            "last_flapped": 1492172106.5163276, "speed": 10000,
            "mac_address": "FF:FF:FF:FF:FF:FF"
        }
    }

The content is the wanted result of ``get_interfaces`` in JSON, exactly as
another driver would return it.

Mock multiple iterative calls
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If ``/tmp/mock/get_interfaces.1`` was defined and used, for any other call on
the same device, the number of calls needs to be incremented.

For example, to call ``device.get_interfaces_ip()`` after
``device.get_interfaces()``, the file ``/tmp/mock/get_interfaces_ip.2`` needs
to be defined::

    {
        "Ethernet1/1": {
            "ipv6": {"2001:DB8::": {"prefix_length": 64}}
        }
    }

Mock a CLI call
~~~~~~~~~~~~~~~

``device.cli(commands)`` calls are a bit different to mock, as a suffix
corresponding to the command applied to the device need to be added. As before,
the data mocked file will start by ``cli`` and the number of calls done before
(here, ``cli.1``). Then, the same process needs to be applied to each
command.

Each command needs to be sanitized: any special character (`` -,./\``, etc.)
needs to be replaced by ``_``. Add the index of this command as it is sent to
``device.cli()``. Each file then will contain the raw wanted output of its
associated command.

Example
^^^^^^^

Example with 2 commands, ``show interface Ethernet 1/1`` and ``show interface
Ethernet 1/2``.

To define the mocked data, create a file ``/tmp/mock/cli.1.show_interface_Ethernet_1_1.0``::

    Ethernet1/1 is up
    admin state is up, Dedicated Interface

And a file ``/tmp/mock/cli.1.show_interface_Ethernet_1_2.1``::

    Ethernet1/2 is up
    admin state is up, Dedicated Interface

And now they can be called::

    >>> device.cli(["show interface Ethernet 1/1", "show interface Ethernet 1/2"])
