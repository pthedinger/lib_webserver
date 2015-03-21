.. include:: ../../../README.rst

Usage
-----

To use the library you need to add ``lib_webserver`` to the
``USED_MODULES`` variable in your application Makefile.

Within your application you can also add the following files to
configure the server:

``web/``
   Directory containing the HTML and other files to be served by the
   webserver

``web/webserver.conf``
   A configuration file to control the generation of the web data

``web_server_conf.h``
   The file can be anywhere in your source tree and contains #defines
   for configuring the webserver code.

Choosing the webserver mode
...........................

Configuring the webserver to run from program memory
++++++++++++++++++++++++++++++++++++++++++++++++++++

The module is set up to run from program memory by default so no extra
configuration is required.

Configuring the webserver to run from flash
+++++++++++++++++++++++++++++++++++++++++++

To run from flash you need to:

  #. Set the define ``WEB_SERVER_USE_FLASH`` to 1 in your
     ``web_server_conf.h`` file
  #. Add the following lines to ``webserver.conf`` in your ``web/`` directory::

      [Webserver]
      use_flash=true

This will cause the web pages to be packed into a binary image and
placed into the application binary directory. See
:ref:`lib_webserver_flashing_pages` for details on how to write the
data to SPI flash.

Configuring the webserver to run from flash with a separate flash task
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

To run from flash you need to:

  #. Set the define ``WEB_SERVER_USE_FLASH`` to 1 in your
     ``web_server_conf.h`` file
  #. Set the define ``WEB_SERVER_SEPARATE_FLASH_TASK`` to 1 in your
     ``web_server_conf.h`` file
  #. Add the following lines to ``webserver.conf`` in your ``web/`` directory::

      [Webserver]
      use_flash=true

This will cause the web pages to be packed into a binary image and
placed into the application binary directory. See
:ref:`lib_webserver_flashing_pages` for details on how to write the
data to SPI flash.

Creating the web pages
......................

To create your web pages just place them in the ``web/`` sub-directory
of your application. This can include text and images and be a file
tree. For example, the demo application web tree is::

    web/webserver.conf
    web/index.html
    web/form.html
    web/dir1/page.html
    web/dir2/page.html
    web/images/xmos_logo.png

Writing dynamic content
.......................

With the HTML pages in the web directory. You can include dynamic
content via ``{% ... %}`` tags. This will evaluate the C expression
within the tag. This expression has several C variables available to
it:

   * ``char *buf`` - a buffer to fill with the content to be rendered
     in place of the tag
   * ``int connection_state`` - an integer representing the connection
     state of the current HTTP connection. This can be used with the
     functions in :ref:`lib_webserver_dynamic_content_api`.
   * ``int app_state`` - an integer representing the application
     state. This integer can be set with the :c:func:`web_server_set_app_state`.

The application must return the length of data it has placed in the
``buf`` variable.

.. note::

   All functions that are called by expressions in your web pages need
   to be prototyped in ``web_server_conf.h``

For example when the following HTML is rendered::

   <p>{%  my_function(buf) %}</p>

The web server will render up to the end of the first `<p>` tag and
then call the ``my_function`` C/XC function passing in the ``buf``
array to be filled.

The function ``my_function`` must be declared in ``web_server_conf.h``
e.g.::

   int my_function(char buf[]);

The implementation of the function needs to be somewhere in your
source tree::

   int my_function(char buf[]) {
     char msg[] = "Hello World!";
     strcpy(buf, msg);
     return sizeof(msg)-1;
   }

Note that this example function returns the number of characters it
has placed in the buffer (not including the terminating '\0'
character).

The web server will then insert these characters into the page. So the
page returned to the client will be::

  <p>Hello World!</p>

.. _lib_webserver_flashing_pages:

Writing the pages to SPI flash
..............................

If you configure the webserver to use flash then you need to place the
data onto the attached SPI flash before running your program. To do
this use the ``xflash`` command.

The build of your program will place the data in a file called
``web_data.bin`` in your binary directory. You can flash it using a
command like::

	xflash --boot-partition-size 0x10000 bin/myprog.xe --data bin/web_data.bin

See :ref:`xflash_manual` for more details on how to use ``xflash``.

Integrating the webserver into your code
........................................

All functions in your code to call the webserver can be found in the
``web_server.h`` header::

   #include <web_server.h>

Without SPI Flash
+++++++++++++++++

To use the webserver you must have an instance of an XTCP server
running on your system. You can then write a function that implements
a task that handles tcp events. This may do other functions/handle
other tcp traffic as well as implementing the webserver. Here is a
rough template of how the code should look::

   void tcp_handler(chanend c_xtcp) {
     xtcp_connection_t conn;
     web_server_init(c_xtcp, null, null);

     // Initialize your other code here
    
     while (1) {
       select
         {
         case xtcp_event(c_xtcp,conn):

           // handle other kinds of tcp traffic here

           web_server_handle_event(c_xtcp, null, null, conn);
           break;

         // handle other events in your system here
         }
     }
   }


Using SPI Flash
+++++++++++++++

To use SPI flash you need to pass the ports to access the flash into
the webserver functions. For example::

    void tcp_handler(chanend c_xtcp, fl_SPIPorts &flash_ports) {
      xtcp_connection_t conn;
      web_server_init(c_xtcp, null, flash_ports);
      while (1) {
        select
          {
          case xtcp_event(c_xtcp, conn):
            web_server_handle_event(c_xtcp, null, flash_ports, conn);
            break;
          }
      }
    }

See :ref:`libflash_api` for details on the flash ports. You also need
to define an array of flash devices and
specify the :c:macro:`WEB_SERVER_FLASH_DEVICES` and
:c:macro:`WEB_SERVER_NUM_FLASH_DEVICES` defines in
``web_server_conf.h`` to tell the web server which flash devices to expect.


Using SPI Flash in a separate task
++++++++++++++++++++++++++++++++++

If you configure the web server to use a separate task for flash you
need to run two tasks. The TCP handling tasks now takes a chanend to
talk to the other task. For example::

  void tcp_handler(chanend c_xtcp, chanend c_flash) {
   xtcp_connection_t conn;
   web_server_init(c_xtcp, c_flash, null);
   while (1) {
     select
       {
       case xtcp_event(c_xtcp, conn):
         web_server_handle_event(c_xtcp, c_flash, null, conn);
         break;
       case web_server_flash_response(c_flash):
         web_server_flash_handler(c_flash, c_xtcp);
         break;
       }
   }
  
When serving a web page the :c:func:`web_server_handle_event` may request
data over the ``c_flash`` channel. Later the flash task may respond
and this is handled by the :c:func:`web_server_flash_response` case
which will then progress the TCP transaction.

The task handling the flash should look something like this::

    void flash_handler(chanend c_flash, fl_SPIPorts &flash_ports) {
      web_server_flash_init(flash_ports);
      // Initialize your application code here
      while (1) {
        select {
        case web_server_flash(c_flash, flash_ports);
        // handle over application events here
        }
      }
    }

Again this task may perform other application tasks (that may access
the SPI flash) as well as assisting the web server.

Communicating with other tasks in webpage content
+++++++++++++++++++++++++++++++++++++++++++++++++

It is possible to communicate with other tasks from dynamic content
calls when rendering the webpage. The top-level main of your program
will be something like the following::

  int main() {
     ...
     par {
      ...
      on tile[1]: xtcp(c_xtcp, 1, i_mii,
                       null, null, null,
                       i_smi, ETHERNET_SMI_PHY_ADDRESS,
                       null, otp_ports, ipconfig);

      on tile[1]: tcp_handler(c_xtcp[0]);
      ...
      }
  }

Where ``tcp_handler`` is the task that calls the web event handler
and serves the web pages. Now suppose, we add a new task that we want
to communicate to via the web page (in this example, an I2C bus)::

  int main() {
     ...
     par {
      ...
      on tile[1]: xtcp(c_xtcp, 1, i_mii,
                       null, null, null,
                       i_smi, ETHERNET_SMI_PHY_ADDRESS,
                       null, otp_ports, ipconfig);

      on tile[1]: tcp_handler(c_xtcp[0], i_i2c);
      on tile[1]: i2c_master(i_i2c, 1, p_scl, p_sda, 100);
      ...
      }
  }

The ``i_i2c`` connection to the ``i2c_master`` task is passed to the
``tcp_handler``.

The ``tcp_handler`` needs to store the connection to the I2C task as a
global to allow the web-page function to access it. This is done via a
xC *movable* pointer at the start of the task::

  client i2c_master_if * movable p_i2c;

  void tcp_handler(chanend c_xtcp, client i2c_master_if i2c) {
     client i2c_master_if * movable p = &i2c;
     p_i2c = move(p);

     xtcp_connection_t conn;
     web_server_init(c_xtcp, null, null);
     ...

This has *moved* a pointer to the I2C interface to the global variable
``p_i2c``. This can now be used in a web-page function. In a seperate
file you can write::

  extern client i2c_master_if * movable p_i2c;

  void do_i2c_stuff() {
     i2c_regop_res_t result;
     result = p_i2c->write_reg(0x45, 0x07, 0x12);
  }

This function can be called from a dynamic expression in a webpage
e.g.::

   {% do_i2c_stuff() %}

Remember that functions call from dynamic webpage content must be
prototyped in the ``web_server_conf.h`` header in your application.


API - Configuration defines
---------------------------

These defines can either be set in your Makefile (by adding
``-DNAME=VALUE`` to ``XCC_FLAGS``) or in a file called
``web_server_conf.h`` within your application source tree (this will
get examined by the library).

.. doxygendefine:: WEB_SERVER_PORT
.. doxygendefine:: WEB_SERVER_USE_FLASH
.. doxygendefine:: WEB_SERVER_SEPARATE_FLASH_TASK
.. doxygendefine:: WEB_SERVER_FLASH_DEVICES
.. doxygendefine:: WEB_SERVER_NUM_FLASH_DEVICES

.. c:macro:: WEB_SERVER_POST_RENDER_FUNCTION

   This define can be set to a function that will be run after every
   rendering of part of a web page. The function gets passed the
   application and connection state and must have the type::

     void render(int app_state, int connection_state)

API
---

All functions can be found in the ``web_server.h`` header::

   #include <web_server.h>

Web Server Functions
....................

The following functions can be uses in code that has a channel
connected to an XTCP server task. The functions handle all the
communication with the tcp stack.

.. doxygenfunction:: web_server_init
.. doxygenfunction:: web_server_handle_event
.. doxygenfunction:: web_server_set_app_state

Separate Flash Task Functions
.............................

If ``WEB_SERVER_SEPARATE_FLASH_TASK`` is enabled then a task
with access to flash must use these functions. This task will need to
follow a pattern similar to::

    void flash_handler(chanend c_flash) {
      web_server_flash_init(flash_ports);
      while (1) {
        select {
          // Do other flash handling stuff here
          case web_server_flash(c_flash, flash_ports);
        }
      }
    }


.. doxygenfunction:: web_server_flash_init
.. doxygenfunction:: web_server_flash

In addition the task handling the xtcp connnections needs to respond
to cache events from the flash handling task. This task will need to
follow a pattern similar to::

    void tcp_handler(chanend c_xtcp, chanend c_flash) {
      xtcp_connection_t conn;
      web_server_init(c_xtcp, c_flash, null);
      while (1) {
        select
          {
          case xtcp_event(c_xtcp, conn):
            // handle non web related tcp events here
            web_server_handle_event(c_xtcp, c_flash, null, conn);
            break;
          case web_server_flash_response(c_flash):
            web_server_flash_handler(c_flash, c_xtcp);
            break;
          }
      }
    }

.. doxygenfunction:: web_server_flash_response
.. doxygenfunction:: web_server_flash_handler


.. _lib_webserver_dynamic_content_api:

Functions that can be called during page rendering
..................................................

When functions are called during page rendering (either via the ``{% ... %}``
template escaping in the webpages or via the
``WEB_SERVER_POST_RENDER_FUNCTION`` function), the following utility functions
can be called.

.. doxygenfunction:: web_server_get_param
.. doxygenfunction:: web_server_copy_param
.. doxygenfunction:: web_server_is_post
.. doxygenfunction:: web_server_get_current_file
.. doxygenfunction:: web_server_end_of_page


|newpage|

|appendix|

Known Issues
------------

There are no known issues with this library.

.. include:: ../../../CHANGELOG.rst
