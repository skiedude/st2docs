Action Aliases
==============

Action Aliases are a simplified and more human readable representation
of actions in |st2|. They are useful in text based interfaces, notably ChatOps.

Action Alias Structure
^^^^^^^^^^^^^^^^^^^^^^

Action aliases are content like actions, rules, and sensors. They are defined in YAML
files and deployed via packs, e.g.:

.. code-block:: yaml

    ---
    name: "remote_shell_cmd"
    pack: "examples"
    action_ref: "core.remote"
    description: "Execute a command on a remote host via SSH."
    formats:
      - "run {{cmd}} on {{hosts}}"


In the above example ``remote_shell_cmd`` is an alias for the ``core.remote`` action. The
supported format for the alias is specified in the ``formats`` field. A single alias can support
multiple formats for the same action.

Property description
~~~~~~~~~~~~~~~~~~~~

1. ``name``: unique name of the alias.
2. ``action_ref``: the action that is being aliased.
3. ``formats``: possible options for users to invoke the action. Typically entered in a textual
   interface supported by ChatOps.

Location
~~~~~~~~

Action Aliases are supplied in packs as YAML files:

.. code-block:: bash

    packs/my_pack$ ls
    actions  aliases  rules  sensors

Each alias is a single YAML file containing the alias definition.

Listing
~~~~~~~

To list all currently registered action aliases, use:

.. code-block:: bash

   st2 action-alias list

Loading
~~~~~~~

Aliases are not automatically loaded when a pack is registered. To load all aliases use:

.. code-block:: bash

   st2ctl reload --register-aliases

Removing
~~~~~~~~

To remove an action alias, use:

.. code-block:: bash

   st2 action-alias delete {ALIAS}

After removing or modifying existing aliases, you may need to restart the ``st2chatops`` service,
or you may see old or duplicate commands still showing up on your chatbot.

Testing an alias end to end
~~~~~~~~~~~~~~~~~~~~~~~~~~~

To test an alias end to end (from matching to triggering an execution and formatting the result),
you can use ``st2 action-alias test <message string>`` command which has been added in |st2|
v3.7.0.

This command is useful for testing and developing aliases since it allows you to skip the chat
layer and verify and adjust command matching and result formatting directly using the CLI. In the
end you should of course still verify everything works end to end via the chat / hubot layer by
executing a command on chat.

This command first checks if the provided command string matches any alias (same as the
``st2 action-alias match`` command) and if it does, it runs an execution for the matched
alias (same as the ``st2 action-alias execute``) command and at the end, prints the formatted
result for the triggered execution.

Example usage and output:

.. code-block:: bash

   $ st2 action-alias test "run 'whoami ; date ; echo stdout ; echo stderr >&2' on localhost"
   Triggering execution via action alias

   Execution (6027f61dffb5b8fc2ebc204c) has been started, waiting for it to finish...

   .

   Execution (6027f61dffb5b8fc2ebc204c) has finished, rendering result...

   Execution (6027f61fffb5b8fc2ebc204f) has been started, waiting for it to finish...

   .

   Formatted ChatOps result message

   ================================================================================
   Ran command *whoami ; date ; echo stdout ; echo stderr >&2* on *1* hosts.

   Details are as follows:
   Host: *localhost*
       ---> stdout: stanley
   Sat Feb 13 15:54:05 UTC 2021
   stdout
       ---> stderr: stderr

   ================================================================================

Supported formats
^^^^^^^^^^^^^^^^^

Aliases support the following format structures:

Basic
~~~~~

.. code-block:: yaml

    formats:
      - "run {{cmd}} on {{hosts}}""


If the user entered ``run date on localhost`` via a ChatOps interface, the aliasing mechanism
would interpret this as ``cmd=date hosts=localhost``. The action ``core.remote`` would then be
called with the parameters:

.. code-block:: yaml

   parameters:
       cmd: date
       hosts: localhost

Since ``core.remote`` accepts multiple hosts, you can also use a comma-separated list:
``run date on 10.0.10.1,10.0.10.2``.

With default
~~~~~~~~~~~~

Using this example:

.. code-block:: yaml

    formats:
      - "run {{cmd}} {{hosts=localhost}}"

In this case, the query has a default value assigned which will be used if no value is provided by
the user. Therefore, a simple ``run date`` instead of ``run date 10.0.10.1`` would result in
assigning the default value, in a similar manner to how Action default parameter values are
interpreted.

For default inputs like JSON, the following pattern can be applied:

.. code-block:: yaml

    formats:
      - "run {{thing={'key': 'value'}}}"

It is therefore possible to pass information to a runner like a HTTP header as a default value
using this pattern.


With immutable parameters
~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes an alias must have default values that cannot be changed by the chat user.

Using this example:

.. code-block:: yaml

    formats:
      - "run {{cmd}}"
    immutable_parameters:
      hosts: localhost

In this case, the action will always receive the hosts parameter as ``localhost``. An attempt to
override this parameter on the message will raise an error.

You can pass any number of arguments to the action using ``immutable_parameters`` and values
support Jinja templating so you can, for example, retrieve a value from the datastore.

.. code-block:: yaml

    formats:
      - "run {{cmd}}"
    immutable_parameters:
      hosts: dev.server
      sudo_password: "{{ st2kv('system.dev_server_sudo_password', decrypt=true) }}"

Regular Expressions
~~~~~~~~~~~~~~~~~~~

You can use regular expressions in the format string:

.. code-block:: yaml

    formats:
      - "(run|execute) {{cmd}}( on {{hosts=localhost}})?[!.]?"

They can be as complex as you want, just exercise reasonable caution as regexes tend to be
difficult to debug. If you think you have a problem that can be solved by a regex...well now you
have two problems to solve.

Key-Value Parameters
~~~~~~~~~~~~~~~~~~~~

Using this example:

.. code-block:: yaml

    formats:
      - "run {{cmd}} on {{hosts}}"

Users can supply extra key value parameters like ``run date on localhost timeout=120``. In this
case even though ``timeout`` does not appear in any alias format it will still be extracted and
supplied for execution. In this case the action ``core.remote`` would be called with the
parameters:

.. code-block:: yaml

   parameters:
       cmd: date
       hosts: localhost
       timeout: 120

Additional ChatOps Parameters Passed
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An execution triggered via ChatOps will contain variables such as ``action_context.api_user``,
``action_context.user`` and ``action_context.source_channel``. ``api_user`` is the user who runs
the ChatOps command from the client and ``user`` is the |st2| user configured in hubot.
``source_channel`` is the channel in which the ChatOps command was entered.

If you are attempting to access this information from inside an ActionChain, you will need to
reference the variables through the parent, e.g. ``action_context.parent.api_user``

If you are attempting to access this from inside an Orquesta workflow, you will need to reference the
``st2()`` context, e.g. ``<% ctx(st2).source_channel %>``.

Multiple Formats in a Single Alias
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A single alias file allows multiple formats to be specified for a single alias e.g.:

.. code-block:: yaml

    ---
    name: "st2_sensors_list"
    pack: "st2"
    action_ref: "st2.sensors.list"
    description: "List available StackStorm sensors."
    formats:
        - "list sensors"
        - "list sensors from {{ pack }}"
        - "sensors list"

The above alias supports the following commands:

.. code-block:: bash

    !sensors list
    !list sensors
    !sensors list pack=examples
    !list sensors from examples
    !list sensors from examples limit=2


"Display-representation" Format Objects
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

By default, every format string is exposed in Hubot help as-is. This is not always desirable in
cases where you want to make a complicated regex, have ten very similar format strings to
"humanize" the input, or need to hide one of the strings for any other reason.

In this case, instead of having a string in ``formats``, you can write an object with a ``display``
parameter (a string that will show up in help) and a ``representation`` list (matches that Hubot
will actually look for):

.. code-block:: yaml

    formats:
      - display: "run {{cmd}} on {{hosts}}"
        representation:
          - "(run|execute) {{cmd}}( on {{hosts=localhost}})?[!.]?"
          - "run remote command {{cmd}} on {{hosts}}"

This works as follows:

  - the ``display`` string (``run {{cmd}} on {{hosts}}``) will be exposed via the ``!help``
    command.
  - strings from the ``representation`` list (``(run|execute) {{cmd}}( on
    {{hosts=localhost}})?[!.]?`` regex, and ``run remote command {{cmd}} on {{hosts}}`` string)
    will be matched by Hubot.

You can use both strings and display-representation objects in ``formats`` at the same time:

.. code-block:: yaml

    formats:
      - display: "run {{cmd}} on {{hosts}}"
        representation:
          - "(run|execute) {{cmd}}( on {{hosts=localhost}})?[!.]?"
          - "run remote command {{cmd}} on {{hosts}}"
      - "ssh to hosts {{hosts}} and run command {{cmd}}"
      - "OMG st2 just run this command {{cmd}} on ma boxes {{hosts}} already"

Acknowledgment Options
^^^^^^^^^^^^^^^^^^^^^^

Hubot will acknowledge every ChatOps command with a random message containing the |st2| execution
ID and a link to the Web UI. This message can be customized in your alias definition:

.. code-block:: yaml

    ack:
      format: "acknowledged!"
      append_url: false

The ``format`` parameter will customize your message, and the ``append_url`` flag controls the Web
UI link at the end. It is also possible to use Jinja in the format string, with ``actionalias``
and ``execution`` comprising the Jinja context:

.. code-block:: yaml

    ack:
      format: "Executing `{{ actionalias.ref }}`, your ID is `{{ execution.id[:2] }}..{{ execution.id[-2:] }}`"

The ``enabled`` parameter controls whether the message will be sent. It defaults to ``true``.
Setting it to ``false`` will disable the acknowledgment message:

.. code-block:: yaml

    ack:
      enabled: false

Result Options
^^^^^^^^^^^^^^

Similar to ``ack``, you can configure ``result`` to disable result messages or set a custom format
so that Hubot will output a nicely formatted list, filter strings, or switch the message text
depending on execution status:

.. code-block:: yaml

    result:
      format: |
        Ran command *{{execution.parameters.cmd}}* on *{{ execution.result | length }}* hosts.

        Details are as follows:
        {% for host in execution.result -%}
            Host: *{{host}}*
            ---> stdout: {{execution.result[host].stdout}}
            ---> stderr: {{execution.result[host].stderr}}
        {%+ endfor %}


To disable the result message, you can use the ``enabled`` flag in the same way as in ``ack``.

Threading Replies (Slack)
^^^^^^^^^^^^^^^^^^^^^^^^^

You can configure the hubot ``ack`` and ``result`` messages to be sent as a threaded reply to the invoked command in slack. This defaults to false if not set.

.. code-block:: yaml

    ack:
      extra:
        slack:
          thread_response: true
      format: ":green-check: Querying those items for you..."
      append_url: false
    result:
      extra:
        slack:
          thread_response: true
      format: |
        ```{{execution.result.output.results}}```

Plaintext Messages (Slack)
^^^^^^^^^^^^^^^^^^^^^^^^^^

Result messages tend to be quite long, and Hubot will utilize the extra formatting capabilities of
some chat clients: Slack messages will be sent as attachments. While this is a good fit in most
cases, sometimes you want part or all of your message in plaintext. Use ``{~}`` as a delimiter to
split a message into a plaintext/attachment pair:

.. code-block:: yaml

    result:
      format: "action completed! {~} {{ execution.result }}"

In this case "action completed!" will be displayed in plaintext, and the execution result will
follow as an attachment.

``{~}`` at the end of the string will display the whole message in plaintext.

Passing Attachment API parameters (Slack, Mattermost, and Rocketchat only)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Slack formats ChatOps output as attachments, and you can configure the API parameters in the
``result.extra.slack`` field.

.. code-block:: yaml

  ---
  name: "kitten"
  pack: "kitten"
  description: "Post a kitten picture to cheer people up."
  action_ref: "core.noop"
  formats:
    - "kitten pic"
  ack:
    enabled: false
  result:
    format: "your kittens are here! {~} Regards from the Box Kingdom."
    extra:
      slack:
        image_url: "http://i.imgur.com/Gb9kAYK.jpg"
        fields:
          - title: Kitten headcount
            value: Eight.
            short: true
          - title: Number of boxes
            value: A bunch.
            short: true
        color: "#00AA00"

Everything that's specified in ``extra.slack`` will be passed as-is to the
`Slack Attachment API <https://api.slack.com/docs/attachments>`_.

Note: parameters in ``extra`` support Jinja templating, and you can calculate the values
dynamically:

.. code-block:: yaml

  [...]
  formats:
    - "say {{ phrase }} in {{ color }}"
  result:
    extra:
      slack:
        color: "{{execution.parameters.color}}"
  [...]

Mattermost and Rocketchat also support the Slack attachments API. However, you will need
to use the ``mattermost`` and ``rocketchat`` keys of ``extra``:

.. code-block:: yaml

  [...]
  formats:
    - "say {{ phrase }} in {{ color }}"
  result:
    extra:
      mattermost:
        color: "{{execution.parameters.color}}"
  [...]

.. code-block:: yaml

  formats:
    - "say {{ phrase }} in {{ color }}"
  result:
    extra:
      rocketchat:
        color: "{{execution.parameters.color}}"
  [...]

.. _specifying-multiple-extra-keys-for-different-providers:

Specifying Multiple ``extra`` Keys For Different Providers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

And you absolutely can specify more than one chat provider in a single alias by using
more than one key in ``extra``. This can be useful if you might switch chat providers
in the future, or if you are trying to prototype an alias to (for instance) compare and
contrast chat providers (although we can tell you that, right now, Slack definitely has
the best integration with st2chatops).

.. code-block:: yaml

  [...]
  formats:
    - "say {{ phrase }} in {{ color }}"
  result:
    extra:
      slack:
        image_url: "http://i.imgur.com/Gb9kAYK.jpg"
        fields:
          - title: Kitten headcount
            value: Eight.
            short: true
          - title: Number of boxes
            value: A bunch.
            short: true
        color: "#00AA00"
      mattermost:
        color: "{{execution.parameters.color}}"
      rocketchat:
        color: "{{execution.parameters.color}}"
  [...]

Other chat providers also support ``extra``. See the :ref:`example below <extra-hacking>` for
hacking on ``extra``. You may also need to dig into the
`source code <https://github.com/StackStorm/hubot-stackstorm/tree/master/src/lib>`_ for
individual adapters to see how to use ``extra`` for your chat provider.

Testing and extending alias parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Action Aliases have a strict schema, and normally you have to modify it if you want to introduce
new parameters to Hubot. However, ``extra`` (see above) is schema-less and can be used for hacking
on ``hubot-stackstorm`` without having to modify StackStorm source code.

For example, you might want to introduce an ``audit`` parameter that would make Hubot log
executions of certain aliases into a separate file. You would define it in your aliases like this:

.. _extra-hacking:

.. code-block:: yaml

    ---
    name: "remote_shell_cmd"
    pack: "examples"
    action_ref: "core.remote"
    description: "Execute a command on a remote host via SSH."
    formats:
        - "run {{cmd}} on {{hosts}}"
    extra:
      audit: true


Then you can access it as ``extra.audit`` inside the Hubot StackStorm plugin. A good example of
working with ``extra`` parameters is the `Slack post handler
<https://github.com/StackStorm/hubot-stackstorm/blob/v0.4.2/lib/post_data.js#L43>`_
in ``hubot-stackstorm``.

A sample alias ships with |st2|. Please checkout :github_st2:`st2/contrib/examples/aliases/remote_shell_cmd.yaml <contrib/examples/aliases/remote_shell_cmd.yaml>`.
