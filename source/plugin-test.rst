.. _plugintest:

Database plugin testing
===========================

To test the node plugin, the VAMDC-TAP Validator software may be used.
It implements the same DatabasePlugin interface call logic and RequestInterface methods operation
as the VAMDC-TAP node webservice. 
All the dependency libraries are bundled in a single TAPValidator.jar archive.

Such an approach facilitates the node plugin development: there is no need to install the 
application server and deploy web service framework on the development machine.

To use the TAPValidator for the plugin development, few steps are required:

*	The most recent version of TAPValidator.jar should be added to the project library path.

*	New Java Application run/execution configuration should be defined,
	indicating the **org.vamdc.validator.ValidatorMain** as the main class.

*	TAPValidator should be configured to operate in the plugin development mode.
	In the Settings window, plugin mode radiobutton should be selected;
	The class name implementing the :ref:`DatabasePlug` should be indicated,
	including the java package name.

If everything is set up correctly, the list of supported restrictables should appear
in the right-top text area. Plugin **buildXSAMS(...)** method is called every time the *Query* button is pressed.

For more information on the VAMDC-TAP Validator user interface and features, 
please refer to the [TAPValidator]_ documentation.


.. raw:: latex

    \newpage

Eclipse project setup / Screenshots
--------------------------------------

The screenshots corresponding to the Eclipse project setup are presented below.


.. _buildpath:

Adding VAMDC-TAP Validator to the build path
+++++++++++++++++++++++++++++++++++++++++++++++++

Open the project properties of your database plugin.

.. image:: img/validator/buildpath.png

Add the latest VAMDC-TAP Validator JAR to the build path by clicking on the "Add external JARs" button


.. _runconfs:

Managing run configurations
+++++++++++++++++++++++++++++


.. image:: img/validator/runconfs.png

Setup a new run configuration by clicking the "New..." button


.. _newrunconf:

Creating run configuration
+++++++++++++++++++++++++++++


.. image:: img/validator/newrunconf.png

Create a new run configuration with the **org.vamdc.validator.ValidatorMain** Main class path

.. raw:: latex

    \newpage

Maven Integration
--------------------

All Java software developed as a part of VAMDC is available at VAMDC Maven repository

http://nexus.vamdc.org/nexus/content/repositories/releases/

To use Maven for dependency management of your plugin, a following sample POM.xml may be used::

	<project xmlns="http://maven.apache.org/POM/4.0.0"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
			http://maven.apache.org/xsd/maven-4.0.0.xsd">
		<modelVersion>4.0.0</modelVersion>
		<groupId>org.vamdc.%databasename%</groupId>
		<artifactId>plugin</artifactId>
		<name>databasename plugin for java node software</name>
		
		<repositories>
			<repository>
				<id>nexus</id>
				<name>VAMDC Releases</name>
				<url>http://nexus.vamdc.org/nexus/content/groups/public/</url>
			</repository>
		</repositories>

		<parent>
			<groupId>org.vamdc.tap</groupId>
			<artifactId>vamdctap-plugin</artifactId>
			<version>12.07r2</version>
		</parent>

		<dependencies>
			<dependency>
				<groupId>org.vamdc.%databasename%</groupId>
				<artifactId>database_dao</artifactId>
				<version>0.0.1-SNAPSHOT</version>
			</dependency>
			<dependency>
				<groupId>org.vamdc</groupId>
				<artifactId>TAPValidator</artifactId>
				<version>12.07r2</version>
				<type>jar</type>
				<scope>compile</scope>
			</dependency>
		</dependencies>
	</project>

All the required dependencies are included in the *parent* project, **vamdctap-plugin**

