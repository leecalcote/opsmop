.. image:: opsmop.png
   :alt: OpsMop Logo

.. _push:

Push Mode
---------

How do you configure multiple hosts at a time, in very specific orderings?

OpsMop push mode uses policy language files just like :ref:`local`, but operates on multiple hosts using SSH.
Different tiers of hosts can be contacted and configured in very cleanly ordered, but still paralellized workflows.

This is best understood while reviewing the `push_demo.py <https://github.com/opsmop/opsmop-demo/blob/master/content/push_demo.py>`_ example.
Please review this in another tab while reading the rest of the documentation.

.. _how_push_works:

How Things Work
===============

OpsMop was designed from the beginning to be extremely fast in local mode by executing providers in process.

When in remote mode, it is helpful to think of remote communication as happening on a role by role basis.
For each role, the system will connect to any hosts (or groups of hosts) referenced in that role.

Then, the list of roles will be transferred to the remote host, and executed remotely.  This happens asynchronously,
so each host is trying to execute through the role as fast as it can, rather than task by task.

Along the way, various events occur and are sent back to the remote client, showing realtime status as configuration
occurs.

Reporting on what hosts changed for each role is provided before moving on to the next role.

If any host hits an error, the whole process stops with that role, and does not proceed to future roles.  
(Some future controls over this behavior may be provided.)

Before proceeding to the next role, a summary of the changes made to each host is provided.

.. _push_cli:

Command Line Usage
==================

Similar to :ref:`local`, the opsmop command line for push mode is very simple::

    cd opsmop-demo/content
    python3 filename.py --check --push
    python3 filename.py --apply --push

Configuration is largely defined in the policy file.  Additional CLI flags may be added over time.

.. _push_role_methods:

Role Methods
==============

Roles in OpsMop are described in :ref:`roles`, but when executed in push mode, they can make use of
additional methods.

The following new methods are useful on each *Role* object in OpsMop when invoked in pull mode. Let's review
the most important ones:

.. code-block: python

    inventory = TomlInventory("inventory/inventory.toml")

    class DemoRole(Role):

        def inventory(self):
            # required!
            return inventory.filter(groups='webservers*')

        def ssh_as(self):
            # username and optionally a password
            return ('opsmop', None)

        def sudo(self):
            # whether to sudo (usually this should be True)
            return True

        def sudo_as(self):
            # username and optionally a password for the sudo account
            return ('root', None)

        def check_host_keys(self):
            # whether to check host keys, the default is True
            return False

To review, these are all is is in addition the the usual parts of the role, which are used in both local and push modes:

.. code-block: python

    def set_resources(self):
        # ...

    def set_handlers(self):
        # ...

    def set_variables(self):
        # ...

    def should_process_when(self):
        # ...

    def pre(self):
        # ...

    def post(self):
        # ...

If you need a review of basic language features, see :ref:`language` and :ref:`advanced`.

This may seem like a lot of methods to define for each role, but remember that OpsMop is Python, and you can define
a BaseRole and then subclass from it to keep your roles short and organized!

.. note::

   In all computer systems, not just OpsMop, SSH connection with keys to untrusted hosts can be insecure, 
   because that host can read your password. Wherever possible, you should connect by SSH key.

.. note::

    Usage of ssh-agent with  SSH keys is highly recommended for ease of automation, and the integrated SSH key management
    features of `Vespene <http://docs.vespene.io>` pair well with OpsMop push mode.

.. _sudo_notes:

Sudo Notes
==========

It is worth noting that the sudo operations that happen above happen only once per role.

If a role requires sudoing to more than one account, it can usually be broken up into more than one role, or instead if sudo without password
is allowed for a particular class of user, the sudo command can be embedded directly in any :ref:`module_shell` resources.

.. _push_defaults:

Defaults Files
==============

For values not specified on the role object, defaults are first looked for in ~/.opsmop/defaults.toml.  If that file does
not exist, defaults are looked for in /etc/opsmop/defaults.toml.

The file has the following format::

    [ssh]
    username = "mpdehaan"
    # password = "1234"
    # check_host_keys = "ignore" # or 'all'

    [sudo]
    username = "root"
    # password = "letmein1234"

    [tuning]
    max_workers = 16

    [python]
    # this is the default for remote hosts
    python_path = '/usr/bin/python3'

    [log]
    path = '~/.opsmop/opsmop.log'
    format = "%(asctime)s %(message)s"


These values are ignored if specified in the "sudo_as" or "connect_as" methods on the *Role* object.
         
.. _push_inventory:

Inventory
=========

Pull mode requires an inventory to decide what hosts to target.  Inventory can also attach variables
to each host (for use in :ref:`templates` or :ref:`conditionals`), and there are certain special
variables that can influence how the push mode operates.

Inventory objects can be filtered, as shown above and in 'push_demo.py', by specifying a `fnmatch <https://docs.python.org/3/library/fnmatch.html>`_ pattern.
For instance, an inventory can be carved down to a particular list of groups and/or hosts.

As detailed above, inventory is specified on each role, like this:

.. code-block:: python

    def inventory(self):
        return inventory.filter(groups='webservers')

That's an explicit group name.  We could also match groups starting with a pattern:

.. code-block:: python

    def inventory(self):
        return inventory.filter(groups='dc*')

The inventory class also allow filtering by host names, though usually you should just use groups:

.. code-block:: python

    def inventory(self):
        return inventory.filter(hosts='*.dc.example.com')

And, finally, the inventory filtering supports multiple patterns:

.. code-block:: python

    def inventory(self):
        return inventory.filter(groups=['webservers','dbservers'])

Recall that OpsMop is pure python, so as long as you return an inventory object from this method, you can do whatever
you want with it, including subclassing inventory.

.. _inventory_limits:

Inventory Limits on the Command Line
====================================
       
The inventory groups used can be further limited on the command line as follows::

    python3 push_demo.py --push --apply --limit-groups 'rack1'
    python3 push_demo.py --push --apply --limit-hosts 'foo.example.com'

This way, it's easy to write generic automation scripts that can target arbitrary inventory, without having to change the policy files.
It is of course important to remember that, once again, OpsMop is pure python, and you could also do all this dynamically from within the policy file.

.. _toml_inventory:

Toml Inventory
==============

An easy method of keeping inventory in source code is the TOML Inventory, best demonstrated 
by `inventory.toml <https://github.com/opsmop/opsmop-demo/blob/master/content/inventory/inventory.toml>`_.

Variables can be assigned at either host or group level.

.. _other_inventory:

Other Inventory Types
=====================

Additional inventory types classes, particularly for cloud providers, would make excellent contributions to OpsMop.  If you are interested in 
adding one, stop by `talk.msphere.io <talk.msphere.io>`_.

This will likely include cloud providers, querying inventory from configurations, and group membership from tags.  Once complete, setup and usage
will be documented here.

.. _magic_inventory_variables:

Magic Inventory Variables
=========================

Certain variables, when assigned in inventory, can be used to specify default values for SSH and Sudo behavior, and are used
*INSTEAD* of the values in default.toml files if they exist.

These variables are usable regardless of inventory source::

    * opsmop_host - the address to connect to
    * opsmop_ssh_username - the SSH username
    * opsmop_ssh_password - the SSH password
    * opsmop_sudo_username - the sudo username
    * opsmop_sudo_password - the sudo password
    * opsmop_via - name of the parent host (see :ref:`connection_trees`)
    * opsmop_python_path - the path to python 3 on the remote system (defaults to /usr/bin/python3)

Variables can be set on  hosts or groups.  Setting them on groups is usually preferred where possible to reduce duplication, though obviously
this doesn't make sense for 'opsmop_host'.

.. _connection_trees:

Connection Trees
================

Connection trees are an optional feature supported by the underlying library "mitogen" that we use for SSH communications 
(help is needed testing them!).  

OpsMop (via mitogen) can SSH-connect through multiple-layers of intermediate hosts, in a fan-out architecture.

Here is an Example using the TOML inventory, to make it easier to understand the structure:

.. code-block: toml

    [groups.bastions.hosts]
    "bastion.example.com" = ""

    [groups.rack1.hosts]
    "rack1-top.example.com" = "opsmop_via=bastion.example.com"
    "rack1-101.example.com" = ""
    "rack1-102.example.com" = ""

    [groups.rack2.hosts]
    "rack2-top.example.com" = "opsmop_via=bastion.example.com"
    "rack2-201.example.com" = ""
    "rack2-202.example.com" = ""

    [groups.rack1.vars]
    opsmop_via = "rack1-top.example.com"

    [groups.rack2.vars]
    opsmop_via = "rack2-top.example.com"

    [groups.fooapp.hosts]
    "rack1-101.example.com" = ""
    "rack2-202.example.com" = ""

    [groups.barapp.hosts]
    "rack2-102.example.com" = ""

.. code-block: python

    class FooApp(Role):

        def inventory(self):
            return inventory.filter(groups='fooapp')
        # ...

.. _push_fileserver:

Understanding the FileServer
============================

OpsMop provides files to servers that need them through the SSHd channel, also courtesy of the mitogen library.

To prevent a rogue host from requesting files that it should not have access to, the file serving features of OpsMop in push mode
are 'opt-in'.

By default, it is possible to reference any paths relative to the main policy file, as featured in 'push_demo.py', and those
files "just work".

To access other paths, a method can be added to the change what paths are served for that role:

.. code-block: python

    class FooRole(Role):

        def allow_fileserving_paths(self):
            return [ '.', '/opt/files' ]

        def set_resources(self):
            return Resources(
                File("/opt/destination/large.file", from_file="/opt/files/large.file")
            )

"." in this case, always means the path of the policy file being executed on the command line.  If any other paths are given,
they should be referenced as absolute paths by any resources that use them, as shown above.  If an 'allow_fileserving_paths'
method is not found on the Role, there is also an opportunity to override the default path ('.') by defining a method on the Policy
class. 

The basic takeaway here is that each Role has fine grained control over what files may be served up.  


When the paths are added to the role, checksumming is performed to avoid transferring any files that do not need to be transferred.

To avoid excessive checksumming, and also for security reasons, a set of patterns to be included and excluded from FileServing
is available on the policy object.  The defaults are largely sensible for most applications:

.. code-block: python

    class YourPolicy(Policy):

        def allow_fileserving_patterns(self):
            return [ '*' ]

        def deny_fileserving_patterns(self):
            return [ '*.py', "*__pycache__*", '*.pyo', '*.pyc', '.git', '.bak', '.swp' ]

You may ask why this is important.  Part of the reason is we don't want to allow a rogue host SSHd or Python to request files it should
not have access to, or to allow accidental errors from users sending sensitive files to untrusted hosts.  The other part is we want to avoid
calculating checksums for files we are unlikely to serve up.

.. _push_advanced_tricks:

Advanced Tricks: Rolling Updates And More
=========================================

While less commonly needed in cloud-enabled scenarios where "blue-green" deployments are common, the scenario of rolling updates
is a good one to use to describe many of the advanced features of OpsMop push mode.  These features are not, however, limited to
rolling update capabilities.

In a rolling update, suppose we have 100 hosts connected to a physical load balancer.  What we want to do is contact 10 hosts
at a time, and before updating them, take them out of a load balanced pool.  If they succeed with their updates, we want to put
them back into that load balanced pool.

The OpsMop role might look like this:

.. code-block: python

    class RollingWebServerUpdate(Role):

        def inventory(self):
            # ...

        def set_resources(self):
            # ...

        def set_handlers(self):
            # ....

        def should_contact(self, host):
            # can decide to ignore specific hosts
            return True

        def ssh_as(self):
            return (UserDefaults.ssh_username(), None) # use keys

        def sudo_as(self):
            # if no sudo password is required, just say "None"
            return (UserDefaults.sudo_username(), UserDefaults.sudo_password())

        def sudo(self):
            # yes, we should sudo
            return True

        def serial(self):
            # this many hosts execute at once
            return 10

        def before_connect(self, host):
            # this runs on the control machine
            subprocess.check_rc("unbalance.sh %s" % host.hostname())

        def pre(self):
            # this runs on the remote machine
            pass

        def after_connect(self, host):
            # this runs on the control machine
            subprocess.check_rc("balance.sh %s" % host.hostname())

        def post(self):
            # this runs on the remote machine

As you can see, there are a lot of details to this example, but full control is provided.  Interaction with any piece of hardware, database, or system - including
waiting on external locks, is completely possible *without* needing to rely on extra modules.

While this type of workflow mostly makes sense for a rolling updates with hardware load balancers, the "before_connect" and "after_connect" hooks are completely generic
and can be used for any purpose.

Similarly, the serial control affects how many hosts are going to be processed at any one time, and can be useful when controlling load on a package updates. For instance, if you
had 3000 hosts, it might be a bad idea to let them all hit your package mirror at once.

The serial control also provides a nice failsafe - if there are errors in a serial batch, it can prevent the rest of the hosts from being taken out by a failure during the policy
application.  There is *always* a default value for "serial" in OpsMop, but the default is currently hard coded to do 80 roles a time.  This can easily be made configurable
in future releases.

.. _push_tuning:

Tuning
======

The SSH implementation is already very fast, but there are a few things you can do to boost performance.

Your ansible providers likely have many dependencies.  While opsmop does not require
that you install these dependencies on managed nodes, if you install them, this will
greatly speed up execution time.

These include python packages: jinja2, toml, dill, colorama, and PyYAML.

If not installed, the module code for these are copied over once per each push execution.

.. _push_status:

Current Status
==============

Push mode is still new, and can use help testing in all manner of configurations, including in high-
performance, high-host-count, and high-latency scenarios.  However, most features are already implemented
and this is completely usable today.

1. SELinux (enforcing) support is not operational yet and is waiting on enhancements in mitogen. You should
be able to switch selinux to permissive mode.  Non-SELinux distributions (Debian, Ubuntu, Arch, etc) 
are of course not effected.

2. Connections to hosts are conducted in a threadpool with a default of 16 threaded workers (see :ref:`push_defaults`). If you have a large
number of hosts there may be some lag for the very first time they are contacted that will not occur in subsequent
roles. A future forks flag like "-j4" should allow this to use additional CPUs by dividing the list of hosts up
between processors.

.. _review:

Review
======

All of the features of local mode are usuable in push mode.  Be sure to review useful features documented in :ref:`language`
and :ref:`advanced`.

You will find that while configuration management use cases are the ones that most immediately come to mind, many other tricks
and useful admin utilities can be implemented in OpsMop.

For instance, it is possible to combine :ref:`hooks_should_process_when` with :ref:`file_tests` to write a push mode script that does
something to systems only if they have a particular package installed.

Another example is you could use the "def serial()" method, set to 1 coupled with :ref:`extra_vars`, to make a very basic distributed "cat" that made
use of OpsMop inventory.

What other combinations can you think of?

Credits
=======

Much of the support for push mode in OpsMop comes from the libraries underpinning the implementation, and we would be remiss to not give them
due credit for makings these features much easier to implement.

OpsMop SSH features, including sudo support, file transfer, dependency transfers, remote error handling, and multi-tier connections 
are all powered by `mitogen <https://mitogen.readthedocs.io/en/latest/>`_.

Additionally, heavy use is made of `dill <https://pypi.org/project/dill/>`_ for serialization of python objects.

The asynchronous connections benefit strongly from `concurrent futures <https://docs.python.org/3/library/concurrent.futures.html>`_, a great
improvement on the multiprocessing layer.

