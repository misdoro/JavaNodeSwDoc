.. _plugintest:

Database plugin testing
===========================

TAPValidator software may be used to test the database plugin operation,
so there is no need to install the web server and deploy web service framework on the development machine.
It comes with all the needed libraries bundled, and from the plugin point of view the operation 
with the TAPValidator is undistinguishable from the real-world operation.

To use TAPValidator for the plugin development, simply add the most recent TAPValidator jar file to the library path
and create the new run configuration, indicating the **org.vamdc.validator.ValidatorMain** as the main class.

During the first run, open the Settings dialog, switch to the plugin mode
and configure the Plugin class name to contain the fully-qualified name 
of your class implementing the :ref:`DatabasePlug` interface.

If everything is set up correctly, you should be able to see the list of supported restrictables
in the right-top text area and be able to do the queries.

For more information on the TAPValidator user interface and features, consult the [TAPValidator]_ documentation.

Screenshots
----------------

In case you are using the Eclipse for development, the following screenshots might help.
Open the project properties of your database plugin.

.. _buildpath:

Adding TAPValidator to the build path
+++++++++++++++++++++++++++++++++++++++


.. image:: img/validator/buildpath.png

Add the latest TAPValidator JAR to the build path, clicking on the "Add external JARs" button


.. _runconfs:

Managing run configurations
+++++++++++++++++++++++++++++


.. image:: img/validator/runconfs.png

Setup a new run configuration by clicking the "New..." button


.. _newrunconf:

Creating run configuration
+++++++++++++++++++++++++++++


.. image:: img/validator/newrunconf.png

Create the new run configuration with the following Main class path


