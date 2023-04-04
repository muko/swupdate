.. SPDX-FileCopyrightText: 2013-2021 Stefano Babic <sbabic@denx.de>
.. SPDX-License-Identifier: GPL-2.0-only

=====================
Suricatta daemon mode
=====================

Introduction
------------

..
    Suricatta is -- like mongoose -- a daemon mode of SWUpdate, hence the
    name suricatta (engl. meerkat) as it belongs to the mongoose family.
Suricatta は、 mongoose と同様に SWUpdate のデーモン モードであるため、マングース ファミリーに属しているため、suricatta (engl. ミーアキャット) という名前が付けられています。

..
    Suricatta regularly polls a remote server for updates, downloads, and
    installs them. Thereafter, it reboots the system and reports the update
    status to the server, based on an update state variable currently stored
    in bootloader's environment ensuring persistent storage across reboots. Some
    U-Boot script logics or U-Boot's ``bootcount`` feature may be utilized
    to alter this update state variable, e.g., by setting it to reflect
    failure in case booting the newly flashed root file system has failed
    and a switchback had to be performed.
Suricatta は定期的にリモート サーバーをポーリングして更新、ダウンロード、およびインストールを行います。
その後、システムを再起動し、ブートローダーの環境に現在保存されている更新状態変数に基づいて更新ステータスをサーバーに報告し、再起動後も永続的なストレージを確保します。
一部の U-Boot スクリプト ロジックまたは U-Boot の ``bootcount`` 機能を利用して、この更新状態変数を変更できます。
たとえば、新しくフラッシュされたルート ファイル システムの起動に失敗し、スイッチバックを実行する必要があった場合に、失敗を反映するように変数を設定します。

Suricatta is designed to be extensible in terms of the servers supported
as described in Section `The Suricatta Interface`_. Currently,
support for the `hawkBit`_ server is implemented via the `hawkBit Direct
Device Integration API`_ alongside a simple general purpose HTTP server.
The support for suricatta modules written in Lua is not a particular server
support implementation but rather an option for writing such in Lua instead
of C.

.. _hawkBit Direct Device Integration API:  http://sp.apps.bosch-iot-cloud.com/documentation/developerguide/apispecifications/directdeviceintegrationapi.html
.. _hawkBit:  https://projects.eclipse.org/projects/iot.hawkbit


Running suricatta
-----------------

After having configured and compiled SWUpdate with enabled suricatta
support,

.. code::

  ./swupdate --help

lists the mandatory and optional arguments to be provided to suricatta
when using hawkBit as server. As an example,

.. code:: bash

    ./swupdate -l 5 -u '-t default -u http://10.0.0.2:8080 -i 25'

runs SWUpdate in suricatta daemon mode with log-level ``TRACE``, polling
a hawkBit instance at ``http://10.0.0.2:8080`` with tenant ``default``
and device ID ``25``.


Note that on startup when having installed an update, suricatta
tries to report the update status to its upstream server, e.g.,
hawkBit, prior to entering the main loop awaiting further updates.
If this initial report fails, e.g., because of a not (yet) configured
network or a currently unavailable hawkBit server, SWUpdate may exit
with an according error code. This behavior allows one to, for example,
try several upstream servers sequentially.
If suricatta should keep retrying until the update status is reported
to its upstream server irrespective of the error conditions, this has
to be realized externally in terms of restarting SWUpdate on exit.


After an update has been performed, an agent listening on the progress
interface may execute post-update actions, e.g., a reboot, on receiving
``DONE``. 
Additionally, a post-update command specified in the configuration file or
given by the ``-p`` command line option can be executed.

Note that at least a restart of SWUpdate has to be performed as post-update
action since only then suricatta tries to report the update status to its
upstream server. Otherwise, succinct update actions announced by the
upstream server are skipped with an according message until a restart of
SWUpdate has happened in order to not install the same update again.


The Suricatta Interface
-----------------------

Support for servers other than hawkBit or the general purpose HTTP server can be
realized by implementing the "interfaces" described in ``include/channel.h`` and
``include/suricatta/server.h``, the latter either in C or in Lua.
The channel interface abstracts a particular connection to the server, e.g.,
HTTP-based in case of hawkBit. The server interface defines the logics to poll
and install updates. See ``corelib/channel_curl.c`` / ``include/channel_curl.h``
and ``suricatta/server_hawkbit.{c,h}`` for an example implementation in C targeted
towards hawkBit.

``include/channel.h`` describes the functionality a channel
has to implement:

.. code:: c

    typedef struct channel channel_t;
    struct channel {
        ...
    };

    channel_t *channel_new(void);

which sets up and returns a ``channel_t`` struct with pointers to
functions for opening, closing, fetching, and sending data over
the channel.

``include/suricatta/server.h`` describes the functionality a server has
to implement:

.. code:: c

    server_op_res_t server_has_pending_action(int *action_id);
    server_op_res_t server_install_update(void);
    server_op_res_t server_send_target_data(void);
    unsigned int server_get_polling_interval(void);
    server_op_res_t server_start(const char *cfgfname, int argc, char *argv[]);
    server_op_res_t server_stop(void);
    server_op_res_t server_ipc(int fd);

The type ``server_op_res_t`` is defined in ``include/suricatta/suricatta.h``.
It represents the valid function return codes for a server's implementation.

In addition to implementing the particular channel and server, the
``suricatta/Config.in`` file has to be adapted to include a new option
so that the new implementation becomes selectable in SWUpdate's
configuration. In the simplest case, adding an option like the following
one for hawkBit into the ``menu "Server"`` section is sufficient.

.. code:: bash

    config SURICATTA_HAWKBIT
        bool "hawkBit support"
        depends on HAVE_LIBCURL
        depends on HAVE_JSON_C
        select JSON
        select CURL
        help
          Support for hawkBit server.
          https://projects.eclipse.org/projects/iot.hawkbit

Having included the new server implementation into the configuration,
edit ``suricatta/Makefile`` to specify the implementation's linkage into
the SWUpdate binary, e.g., for the hawkBit example implementation, the
following lines add ``server_hawkbit.o`` to the resulting SWUpdate binary
if ``SURICATTA_HAWKBIT`` was selected while configuring SWUpdate.

.. code:: bash

    ifneq ($(CONFIG_SURICATTA_HAWKBIT),)
    lib-$(CONFIG_SURICATTA) += server_hawkbit.o
    endif


Support for general purpose HTTP server
---------------------------------------

This is a very simple backend that uses standard HTTP response codes to signal if
an update is available. There are closed source backends implementing this interface,
but because the interface is very simple interface, this server type is also suitable
for implementing an own backend server. For inspiration, there's a simple (mock)
server implementation available in ``examples/suricatta/server_general.py``.

The API consists of a GET with Query parameters to inform the server about the installed version.
The query string has the format:

::

        http(s)://<base URL>?param1=val1&param2=value2...

As examples for parameters, the device can send its serial number, MAC address and the running version of the software.
It is duty of the backend to interpret this - SWUpdate just takes them from the "identify" section of
the configuration file and encodes the URL.

The server answers with the following return codes:

+-----------+-------------+------------------------------------------------------------+
| HTTP Code | Text        | Description                                                |
+===========+=============+============================================================+
|    302    | Found       | A new software is available at URL in the Location header  |
+-----------+-------------+------------------------------------------------------------+
|    400    | Bad Request | Some query parameters are missing or in wrong format       |
+-----------+-------------+------------------------------------------------------------+
|    403    | Forbidden   | Client certificate not valid                               |
+-----------+-------------+------------------------------------------------------------+
|    404    | Not found   | No update is available for this device                     |
+-----------+-------------+------------------------------------------------------------+
|    503    | Unavailable | An update is available but server can't handle another     |
|           |             | update process now.                                        |
+-----------+-------------+------------------------------------------------------------+

Server's answer can contain the following headers:

+---------------+--------+------------------------------------------------------------+
| Header's name | Codes  | Description                                                |
+===============+========+============================================================+
| Retry-after   |   503  | Contains a number which tells the device how long to wait  |
|               |        | until ask the next time for updates. (Seconds)             |
+---------------+--------+------------------------------------------------------------+
| Content-MD5   |   302  | Contains the checksum of the update file which is available|
|               |        | under the url of location header                           |
+---------------+--------+------------------------------------------------------------+
| Location      |   302  | URL where the update file can be downloaded.               |
+---------------+--------+------------------------------------------------------------+

The device can send logging data to the server. Any information is transmitted in a HTTP
PUT request with the data as plain string in the message body. The Content-Type Header
need to be set to text/plain.

The URL for the logging can be set as separate URL in the configuration file or via
--logurl command line parameter:

The device sends data in a CSV format (Comma Separated Values). The format is:

::

        value1,value2,...

The format can be specified in the configuration file. A *format* For each *event* can be set.
The supported events are:

+---------------+------------------------------------------------------------+
| Event         | Description                                                |
+===============+========+===================================================+
| check         | dummy. It could send an event each time the server is      |
|               | polled.                                                    |
+---------------+------------------------------------------------------------+
| started       | A new software is found and SWUpdate starts to install it  |
+---------------+------------------------------------------------------------+
| success       | A new software was successfully installed                  |
+---------------+------------------------------------------------------------+
| fail          | Failure by installing the new software                     |
+---------------+------------------------------------------------------------+

The `general server` has an own section inside the configuration file. As example:

::

        gservice =
        {
	        url 		= ....;
	        logurl		= ;
	        logevent : (
		        {event = "check"; format="#2,date,fw,hw,sp"},
		        {event = "started"; format="#12,date,fw,hw,sp"},
		        {event = "success"; format="#13,date,fw,hw,sp"},
		        {event = "fail"; format="#14,date,fw,hw,sp"}
	        );
        }


`date` is a special field and it is interpreted as localtime in RFC 2822 format. Each
Comma Separated field is looked up inside the `identify` section in the configuration
file, and if a match is found the substitution occurs. In case of no match, the field
is sent as it is. For example, if the identify section has the following values:


::

        identify : (
        	{ name = "sp"; value = "333"; },
        	{ name = "hw"; value = "ipse"; },
        	{ name = "fw"; value = "1.0"; }
        );


with the events set as above, the formatted text in case of "success" will be:

::

        Formatted log: #13,Mon, 17 Sep 2018 10:55:18 CEST,1.0,ipse,333


Support for Suricatta Modules in Lua
------------------------------------

The ``server_lua.c`` C-to-Lua bridge enables writing suricatta modules in Lua. It
provides the infrastructure in terms of the interface to SWUpdate "core" to the Lua
realm, enabling the "business logic" such as handling update flows and communicating
with backend server APIs to be modeled in Lua. To the Lua realm, the ``server_lua.c``
C-to-Lua bridge provides the same functionality as the other suricatta modules
written in C have, realizing a separation of means and control. Effectively, it lifts
the interface outlined in Section `The Suricatta Interface`_ to the Lua realm.


As an example server implementation, see ``examples/suricatta/server_general.py`` for
a simple (mock) server of a backend that's modeled after the "General Purpose HTTP
Server" (cf. Section `Support for general purpose HTTP server`_). The matching Lua
suricatta module is found in ``examples/suricatta/swupdate_suricatta.lua``. Place it in
Lua's path so that a ``require("swupdate_suricatta")`` can load it or embed it into the
SWUpdate binary by enabling ``CONFIG_EMBEDDED_SURICATTA_LUA`` and setting
``CONFIG_EMBEDDED_SURICATTA_LUA_SOURCE`` accordingly.

The interface specification in terms of a Lua (suricatta) module is found in
``suricatta/suricatta.lua``.


`suricatta`
...........

The ``suricatta`` table is the module's main table housing the exposed functions and
definitions via the sub-tables described below.
In addition, the main functions ``suricatta.install()`` and ``suricatta.download()``
as well as the convenience functions ``suricatta.getversion()``, ``suricatta.sleep()``,
and ``suricatta.get_tmpdir()`` are exposed:

The function ``suricatta.install(install_channel)`` installs an update artifact from
a remote server or a local file. The ``install_channel`` table parameter designates
the channel to be used for accessing the artifact plus channel options diverging
from the defaults set at channel creation time. For example, an ``install_channel``
table may look like this:

.. code-block:: lua

    { channel = chn, url = "https://artifacts.io/update.swu" }

where ``chn`` is the return value of a call to ``channel.open()``. The other table
attributes, like ``url`` in this example, are channel options diverging from or
omitted while channel creation time, see :ref:`suricatta.channel`. For installing
a local file, an ``install_channel`` table may look like this:

.. code-block:: lua

    { channel = chn, url = "file:///path/to/file.swu" }


The function ``suricatta.download(download_channel, localpath)`` just downloads an
update artifact. The parameter ``download_channel`` is as for ``suricatta.install()``.
The parameter ``localpath`` designates the output path for the artifact. The
``suricatta.get_tmpdir()`` function (see below) is in particular useful for this case
to supply a temporary download location as ``localpath``. A just downloaded artifact
may be installed later using ``suricata.install()`` with an appropriate ``file://``
URL, realizing a deferred installation.

Both, ``suricatta.install()`` and ``suricatta.download()`` return ``true``, or, in
case of error, ``nil``, a ``suricatta.status`` value, and a table with messages in
case of errors, else an empty table.

|

The function ``suricatta.getversion()`` returns a table with SWUpdate's ``version``
and ``patchlevel`` fields. This information can be used to determine API
(in-)compatibility of the Lua suricatta module with the SWUpdate version running it.

The function ``suricatta.sleep(seconds)`` is a wrapper around `SLEEP(3)` for, e.g.,
implementing a REST API call retry mechanism after a number of given seconds have
elapsed.

The function ``suricatta.get_tmpdir()`` returns the path to SWUpdate's temporary
working directory where, e.g., the ``suricatta.download()`` function may place the
downloaded artifacts.


`suricatta.status`
..................

The ``suricatta.status`` table exposes the ``server_op_res_t`` enum values defined in
``include/util.h`` to the Lua realm.


`suricatta.notify`
..................

The ``suricatta.notify`` table provides the usual logging functions to the Lua
suricatta module matching their uppercase-named pendants available in the C realm.

One notable exception is ``suricatta.notify.progress(message)`` which dispatches the
message to the progress interface (see :doc:`progress`). Custom progress client
implementations listening and acting on custom progress messages can be realized
using this function.

All notify functions return ``nil``.


`suricatta.pstate`
..................

The ``suricatta.pstate`` table provides a binding to SWUpdate's (persistent) state
handling functions defined in ``include/state.h``, however, limited to the bootloader
environment variable ``STATE_KEY`` defined by ``CONFIG_UPDATE_STATE_BOOTLOADER`` and
defaulting to ``ustate``. In addition, it captures the ``update_state_t`` enum values.

The function ``suricatta.pstate.save(state)`` requires one of ``suricatta.pstate``'s
"enum" values as parameter and returns ``true``, or, in case of error, ``nil``.
The function ``suricatta.pstate.get()`` returns ``true``, or, in case of error, ``nil``,
plus one of ``suricatta.pstate``'s "enum" values in the former case.


`suricatta.server`
..................

The ``suricatta.server`` table provides the sole function
``suricatta.server.register(function_p, purpose)``. It registers a Lua function
"pointed" to by ``function_p`` for the purpose ``purpose`` which is defined by
``suricatta.server``'s "enum" values. Those enum values correspond to the functions
defined in the interface outlined in the Section on `The Suricatta Interface`_.

In addition to these functions, the two callback functions ``CALLBACK_PROGRESS`` and
``CALLBACK_CHECK_CANCEL`` can be registered optionally: The former can be used to upload
progress information to the server while the latter serves as ``dwlwrdata`` function
(see ``include/channel_curl.h``) to decide on whether an installation should be aborted
while the download phase.

For details on the (callback) functions and their signatures, see the interface
specification ``suricatta/suricatta.lua`` and the documented example Lua suricatta
module found in ``examples/suricatta/swupdate_suricatta.lua``.

The ``suricatta.server.register()`` function returns ``true``, or, in case of error,
``nil``.


.. _suricatta.channel:

`suricatta.channel`
...................

The ``suricatta.channel`` table captures channel handling for suricatta Lua modules.
The single function ``suricatta.channel.open(options)`` creates and opens a channel
to a server. Its single parameter ``options`` is a table specifying the channel's
default options such as `proxy`, `retries`, `usessl`, `strictssl`, or
`headers_to_send`. For convenience, options that may change per request such as
`url`, `content-type`, or `headers_to_send` may be set as defaults on channel
creation time while being selectively overruled on a per request basis. The channel
options currently supported to be set are listed in the ``suricatta.channel.options``
table. In essence, the ``options`` parameter table is the Lua table equivalent of
``include/channel_curl.h``'s ``channel_data_t``.


The ``suricatta.channel.open(options)`` function returns a channel table which is
either passed to the ``suricatta.install()`` and ``suricatta.download()`` functions
or used directly for communication with a server. More specifically, it has the three
functions

* ``get(options)`` for retrieving information from the server,
* ``put(options)`` for sending information to the server, and
* ``close()`` for closing the channel.

The ``get()`` and ``put()`` functions' single parameter ``options`` is a per-request
channel option table as described above.

The functions ``get()`` and ``put()`` return ``true``, or, in case of error, ``nil``,
a ``suricatta.status`` value, and an operation result table.
The latter contains the fields:

* ``http_response_code`` carrying the HTTP error code,
* ``format`` as one of ``suricatta.channel.content``'s options,
* ``raw_reply`` if ``options`` contained ``format = suricatta.channel.content.RAW``,
* ``json_reply`` if ``options`` contained ``format = suricatta.channel.content.JSON``, and
* the HTTP headers received in the ``received_headers`` table, if any.


The ``suricatta.channel.content`` "enum" table defines the "format", i.e., the response
body content type and whether to parse it or not:

* ``NONE`` means the response body is discarded.
* ``RAW`` means the raw server's reply is available as ``raw_reply``.
* ``JSON`` means the server's JSON reply is parsed into a Lua table and available
  as ``json_reply``.


The ``suricatta.channel.method`` "enum" table defines the HTTP method to use for
a request issued with the ``put(options)`` function, i.e., `POST`, `PATCH`, or `PUT` as
specified in the ``options`` parameter table via the ``method`` attribute.
In addition to the HTTP method, the request body's content is set with the
``request_body`` attribute in the ``options`` parameter table.


As a contrived example, consider the following call to a channel's ``put()`` function

.. code-block:: lua

    ...
    local res, _, data = channel.put({
            url          = string.format("%s/%s", base_url, device_id),
            content_type = "application/json",
            method       = suricatta.channel.method.PATCH,
            format       = suricatta.channel.content.NONE,
            request_body = "{ ... }"
        })
    ...

that issues a HTTP `PATCH` to some URL with a JSON content without having interest in
the response body.

More examples of how to use a channel can be found in the example suricatta Lua
module ``examples/suricatta/swupdate_suricatta.lua``.

`suricatta.bootloader`
......................

The ``suricatta.bootloader`` table exposes SWUpdate's bootloader environment
modification functions to suricatta Lua modules.

The enum-like table ``suricatta.bootloader.bootloaders`` holds the bootloaders
SWUpdate supports, i.e.

   .. code-block:: lua

    suricatta.bootloader.bootloaders = {
        EBG   = "ebg",
        NONE  = "none",
        GRUB  = "grub",
        UBOOT = "uboot",
    },


The function ``suricatta.bootloader.get()`` returns the currently selected
bootloader in terms of a ``suricatta.bootloader.bootloaders`` field value.

The function ``suricatta.bootloader.is(name)`` takes one of
``suricatta.bootloader.bootloaders``'s field values as ``name`` and returns
``true`` if it is the currently selected bootloader, ``false`` otherwise.

The functions in the ``suricatta.bootloader.env`` table interact with the
currently selected bootloader's environment:

The function ``suricatta.bootloader.env.get(variable)`` retrieves the value
associated to ``variable`` from the bootloader's environment.

The function ``suricatta.bootloader.env.set(variable, value)`` sets the
bootloader environment's key ``variable`` to ``value``.

The function ``suricatta.bootloader.env.unset(variable)`` deletes the bootloader
environment's key ``variable``.

The function ``suricatta.bootloader.env.apply(filename)`` applies
all key=value lines of a local file ``filename`` to the currently selected
bootloader's environment.
