Using Lookups
=============

Lookup plugins allow access of data in Ansible from outside sources.  These plugins are evaluated on the Ansible control
machine, and can include reading the filesystem but also contacting external datastores and services.
These values are then made available using the standard templating system
in Ansible, and are typically used to load variables or templates with information from those systems.

.. note:: This is considered an advanced feature, and many users will probably not rely on these features.

.. note:: Lookups occur on the local computer, not on the remote computer.

.. note:: Since 1.9 you can pass wantlist=True to lookups to use in jinja2 template "for" loops.

.. contents:: Topics

.. _getting_file_contents:

Intro to Lookups: Getting File Contents
```````````````````````````````````````

The file lookup is the most basic lookup type.

Contents can be read off the filesystem as follows::

    - hosts: all
      vars:
         contents: "{{ lookup('file', '/etc/foo.txt') }}"

      tasks:

         - debug: msg="the value of foo.txt is {{ contents }}"

.. _password_lookup:

The Password Lookup
```````````````````

.. note::

    A great alternative to the password lookup plugin, if you don't need to generate random passwords on a per-host basis, would be to use :doc:`playbooks_vault`.  Read the documentation there and consider using it first, it will be more desirable for most applications.

``password`` generates a random plaintext password and stores it in
a file at a given filepath.  

(Docs about crypted save modes are pending)
 
If the file exists previously, it will retrieve its contents, behaving just like with_file. Usage of variables like "{{ inventory_hostname }}" in the filepath can be used to set
up random passwords per host (what simplifies password management in 'host_vars' variables).

Generated passwords contain a random mix of upper and lowercase ASCII letters, the
numbers 0-9 and punctuation (". , : - _"). The default length of a generated password is 20 characters.
This length can be changed by passing an extra parameter::

    ---
    - hosts: all

      tasks:

        # create a mysql user with a random password:
        - mysql_user: name={{ client }}
                      password="{{ lookup('password', 'credentials/' + client + '/' + tier + '/' + role + '/mysqlpassword length=15') }}"
                      priv={{ client }}_{{ tier }}_{{ role }}.*:ALL

        (...)

.. note:: If the file already exists, no data will be written to it. If the file has contents, those contents will be read in as the password. Empty files cause the password to return as an empty string        

Starting in version 1.4, password accepts a "chars" parameter to allow defining a custom character set in the generated passwords. It accepts comma separated list of names that are either string module attributes (ascii_letters,digits, etc) or are used literally::

    ---
    - hosts: all

      tasks:

        # create a mysql user with a random password using only ascii letters:
        - mysql_user: name={{ client }}
                      password="{{ lookup('password', '/tmp/passwordfile chars=ascii_letters') }}"
                      priv={{ client }}_{{ tier }}_{{ role }}.*:ALL

        # create a mysql user with a random password using only digits:
        - mysql_user: name={{ client }}
                      password="{{ lookup('password', '/tmp/passwordfile chars=digits') }}"
                      priv={{ client }}_{{ tier }}_{{ role }}.*:ALL

        # create a mysql user with a random password using many different char sets:
        - mysql_user: name={{ client }}
                      password="{{ lookup('password', '/tmp/passwordfile chars=ascii_letters,digits,hexdigits,punctuation') }}"
                      priv={{ client }}_{{ tier }}_{{ role }}.*:ALL

        (...)

To enter comma use two commas ',,' somewhere - preferably at the end. Quotes and double quotes are not supported.

.. _csvfile_lookup:

The CSV File Lookup
```````````````````
.. versionadded:: 1.5

The ``csvfile`` lookup reads the contents of a file in CSV (comma-separated value)
format. The lookup looks for the row where the first column matches ``keyname``, and
returns the value in the second column, unless a different column is specified.

The example below shows the contents of a CSV file named elements.csv with information about the
periodic table of elements::

    Symbol,Atomic Number,Atomic Mass
    H,1,1.008
    He,2,4.0026
    Li,3,6.94
    Be,4,9.012
    B,5,10.81


We can use the ``csvfile`` plugin to look up the atomic number or atomic of Lithium by its symbol::

    - debug: msg="The atomic number of Lithium is {{ lookup('csvfile', 'Li file=elements.csv delimiter=,') }}"
    - debug: msg="The atomic mass of Lithium is {{ lookup('csvfile', 'Li file=elements.csv delimiter=, col=2') }}"


The ``csvfile`` lookup supports several arguments. The format for passing
arguments is::

    lookup('csvfile', 'key arg1=val1 arg2=val2 ...')

The first value in the argument is the ``key``, which must be an entry that
appears exactly once in column 0 (the first column, 0-indexed) of the table. All other arguments are optional.


==========   ============   =========================================================================================
Field        Default        Description
----------   ------------   -----------------------------------------------------------------------------------------
file         ansible.csv    Name of the file to load
delimiter    TAB            Delimiter used by CSV file. As a special case, tab can be specified as either TAB or \t.
col          1              The column to output, indexed by 0
default      empty string   return value if the key is not in the csv file
==========   ============   =========================================================================================

.. note:: The default delimiter is TAB, *not* comma.

.. _ini_lookup:

The INI File Lookup
```````````````````
.. versionadded:: 2.0

The ``ini`` lookup reads the contents of a file in INI format (key1=value1).
This plugin retrieve the value on the right side after the equal sign ('=') of
a given section ([section]). You can also read a property file which - in this
case - does not contain section.

Here's a simple example of an INI file with user/password configuration::

    [production]
    # My production information
    user=robert
    pass=somerandompassword

    [integration]
    # My integration information
    user=gertrude
    pass=anotherpassword


We can use the ``ini`` plugin to lookup user configuration::

    - debug: msg="User in integration is {{ lookup('ini', 'user section=integration file=users.ini') }}"
    - debug: msg="User in production  is {{ lookup('ini', 'user section=production  file=users.ini') }}"

Another example for this plugin is for looking for a value on java properties.
Here's a simple properties we'll take as an example::

    user.name=robert
    user.pass=somerandompassword

You can retrieve the ``user.name`` field with the following lookup::

    - debug: msg="user.name is {{ lookup('ini', 'user.name type=properties file=user.properties') }}"

The ``ini`` lookup supports several arguments like the csv plugin. The format for passing
arguments is::

    lookup('ini', 'key [type=<properties|ini>] [section=section] [file=file.ini] [re=true] [default=<defaultvalue>]')

The first value in the argument is the ``key``, which must be an entry that
appears exactly once on keys. All other arguments are optional.


==========   ============   =========================================================================================
Field        Default        Description
----------   ------------   -----------------------------------------------------------------------------------------
type         ini            Type of the file. Can be ini or properties (for java properties).
file         ansible.ini    Name of the file to load
section      global         Default section where to lookup for key.
re           False          The key is a regexp.
default      empty string   return value if the key is not in the ini file
==========   ============   =========================================================================================

.. note:: In java properties files, there's no need to specify a section.

.. _credstash_lookup:

The Credstash Lookup
````````````````````
.. versionadded:: 2.0

Credstash is a small utility for managing secrets using AWS's KMS and DynamoDB: https://github.com/LuminalOSS/credstash

First, you need to store your secrets with credstash::


    $ credstash put my-github-password secure123

    my-github-password has been stored


Example usage::


    ---
    - name: "Test credstash lookup plugin -- get my github password"
      debug: msg="Credstash lookup! {{ lookup('credstash', 'my-github-password') }}"


You can specify regions or tables to fetch secrets from::


    ---
    - name: "Test credstash lookup plugin -- get my other password from us-west-1"
      debug: msg="Credstash lookup! {{ lookup('credstash', 'my-other-password', region='us-west-1') }}"


    - name: "Test credstash lookup plugin -- get the company's github password"
      debug: msg="Credstash lookup! {{ lookup('credstash', 'company-github-password', table='company-passwords') }}"

If you're not using 2.0 yet, you can do something similar with the credstash tool and the pipe lookup (see below)::

    debug: msg="Poor man's credstash lookup! {{ lookup('pipe', 'credstash -r us-west-1 get my-other-password') }}"

.. _more_lookups:

More Lookups
````````````

Various *lookup plugins* allow additional ways to iterate over data.  In :doc:`Loops <playbooks_loops>` you will learn
how to use them to walk over collections of numerous types.  However, they can also be used to pull in data
from remote sources, such as shell commands or even key value stores. This section will cover lookup
plugins in this capacity.

Here are some examples::

    ---
    - hosts: all

      tasks:

         - debug: msg="{{ lookup('env','HOME') }} is an environment variable"

         - debug: msg="{{ item }} is a line from the result of this command"
           with_lines:
             - cat /etc/motd

         - debug: msg="{{ lookup('pipe','date') }} is the raw result of running this command"

         # redis_kv lookup requires the Python redis package
         - debug: msg="{{ lookup('redis_kv', 'redis://localhost:6379,somekey') }} is value in Redis for somekey"

         # dnstxt lookup requires the Python dnspython package
         - debug: msg="{{ lookup('dnstxt', 'example.com') }} is a DNS TXT record for example.com"

         - debug: msg="{{ lookup('template', './some_template.j2') }} is a value from evaluation of this template"

         # loading a json file from a template as a string
         - debug: msg="{{ lookup('template', './some_json.json.j2', convert_data=False) }} is a value from evaluation of this template"


         - debug: msg="{{ lookup('etcd', 'foo') }} is a value from a locally running etcd"

         # shelvefile lookup retrieves a string value corresponding to a key inside a Python shelve file
         - debug: msg="{{ lookup('shelvefile', 'file=path_to_some_shelve_file.db key=key_to_retrieve') }}

         # The following lookups were added in 1.9
         - debug: msg="{{item}}"
           with_url:
                - 'https://github.com/gremlin.keys'

         # outputs the cartesian product of the supplied lists
         - debug: msg="{{item}}"
           with_cartesian:
                - list1
                - list2
                - list3

As an alternative you can also assign lookup plugins to variables or use them
elsewhere.  This macros are evaluated each time they are used in a task (or
template)::

    vars:
      motd_value: "{{ lookup('file', '/etc/motd') }}"

    tasks:

      - debug: msg="motd value is {{ motd_value }}"

.. seealso::

   :doc:`playbooks`
       An introduction to playbooks
   :doc:`playbooks_conditionals`
       Conditional statements in playbooks
   :doc:`playbooks_variables`
       All about variables
   :doc:`playbooks_loops`
       Looping in playbooks
   `User Mailing List <http://groups.google.com/group/ansible-devel>`_
       Have a question?  Stop by the google group!
   `irc.freenode.net <http://irc.freenode.net>`_
       #ansible IRC chat channel



