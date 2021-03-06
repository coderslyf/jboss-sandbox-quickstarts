ejb-security-plus:  Use client and server side interceptors to provide authentication information before EJB calls.
====================
Author: Darran Lofthouse  
Level: Advanced  
Technologies: EJB, Security, Interceptors  
Summary: Demonstrates how to use interceptors to supply information for authentication before EJB calls in JBoss EAP.  
Target Product: Sandbox  
Source: <https://github.com/jboss-developer/jboss-sandbox-quickstarts>  

What is it?
-----------

The `ejb-security-plus` quickstart demonstrates how to use client and server side interceptors to provide additional authentication information before making EJB calls in Red Hat JBoss Enterprise Application Platform.

By default, when you make a remote call to an EJB deployed to the JBoss EAP server, the connection to the server is authenticated and any request received over this connection is executed as the identity that authenticated the connection. The authentication at the connection level is dependent on the capabilities of the underlying SASL mechanisms.

Rather than writing custom SASL mechanisms or combining multiple parameters into one this quickstart demonstrates how username / password authentication can be used to open the connection to the server and subsequently supply an additional security token for authentication before the EJB invocation. This is achieved with the addition of the following three components: 

1. A client side interceptor to pass the additional token to the remote server.
2. A server side interceptor to receive the security token and ensure this is passes to the JAAS domain for verification.
3. A JAAS LoginModule to perform authentication taking into account the authenticated user of the connection and the additional security token.
 
The quickstart then makes use of a single remote EJB, `SecuredEJB` to verify that the propagation and verification of the security token is correct and a `RemoteClient` standalone client. 

### SecuredEJB

For this quickstart the `SecuredEJB` only has the following method: -

    String getPrincipalInformation();

This method is used to confirm the identity of the authenticated user, on the client side changing either the users password or the additional authentication token will cause access to this method to be unavailable.  The output from a successfull call to this method will typically look like: -

	[Principal={quickstartUser}]

### RemoteClient

Finally there is the `RemoteClient` stand-alone client. The client demonstrates how a Principal with a custom credential can be set using the `SecurityContextAssociation` and how this can subsequently be used by a `CallbackHandler` for the username/password authentication required for the SASL authentication and subsequently how the additional token can be picked up by a client side interceptor and passed to the server with the invocation.


Note on EJB client interceptors
-----------------------

Red Hat JBoss Enterprise Application Platform allows client side interceptors for EJB invocations. Such interceptors are expected to implement the `org.jboss.ejb.client.EJBClientInterceptor` interface. User applications can then plug in such interceptors in the 'EJBClientContext' either programatically or through the ServiceLoader mechanism.

- The programmatic way involves calling the `org.jboss.ejb.client.EJBClientContext.registerInterceptor(int order, EJBClientInterceptor interceptor)` API and passing the 'order' and the 'interceptor' instance. The 'order' is used to decide where exactly in the client interceptor chain, this 'interceptor' is going to be placed.
- The ServiceLoader mechanism is an alternate approach which involves creating a `META-INF/services/org.jboss.ejb.client.EJBClientInterceptor` file and placing/packaging it in the classpath of the client application. The rules for such a file are dictated by the [Java ServiceLoader Mechanism](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html). This file is expected to contain in each separate line the fully qualified class name of the EJB client interceptor implementation, which is expected to be available in the classpath. EJB client interceptors added via the ServiceLoader mechanism are added to the end of the client interceptor chain, in the order they were found in the classpath.

This quickstart uses the ServiceLoader mechanism for registering the EJB client interceptor and places the `META-INF/services/org.jboss.ejb.client.EJBClientInterceptor` in the classpath, with the following content:

	# EJB client interceptor(s) that will be added to the end of the interceptor chain during an invocation
	# on EJB. If these interceptors are to be added at a specific position, other than last, then use the
	# programmatic API in the application to register it explicitly to the EJBClientContext

	org.jboss.as.quickstarts.ejb_security_plus.ClientSecurityInterceptor


System requirements
-------------------

The application this project produces is designed to be run on Red Hat JBoss Enterprise Application Platform 6.1 or later. 

All you need to build this project is Java 6.0 (Java SDK 1.6) or later, Maven 3.0 or later.

Configure Maven
---------------

If you have not yet done so, you must [Configure Maven](https://github.com/jboss-developer/jboss-developer-shared-resources/blob/master/guides/CONFIGURE_MAVEN.md#configure-maven-to-build-and-deploy-the-quickstarts) before testing the quickstarts.


Use of EAP_HOME
---------------

In the following instructions, replace `EAP_HOME` with the actual path to your JBoss EAP 6 installation. The installation path is described in detail here: [Use of EAP_HOME and JBOSS_HOME Variables](https://github.com/jboss-developer/jboss-developer-shared-resources/blob/master/guides/USE_OF_EAP_HOME.md#use-of-eap_home-and-jboss_home-variables).


Prerequisites
-------------

This quickstart uses the default standalone configuration plus the modifications described here.

It is recommended that you test this approach in a separate and clean environment before you attempt to port the changes in your own environment.


Configure the JBoss server
---------------------------

These steps asume that you are running the server in standalone mode and using the default standalone.xml supplied with the distribution.

You configure the security domain by running the  `configure-security-domain.cli` script provided in the root directory of this quickstart.

After the server is configured you then need to define four user accounts, this can be achieved either by using the add-user tool included with the server or by copying and pasting the appropriate entries into the properties files.  Both of these approaches are described below and whichever approach is chosen it must be completed before running the quickstart - the users can be added before or after starting the server.

1. Before you begin, back up your server configuration file
    * If it is running, stop the JBoss EAP server.
    * Backup the file: `EAP_HOME/standalone/configuration/standalone-full.xml`
    * After you have completed testing this quickstart, you can replace this file to restore the server to its original configuration.
2. Start the JBoss server by typing the following: 

        For Linux:  EAP_HOME/bin/standalone.sh 
        For Windows:  EAP_HOME\bin\standalone.bat
3. Open a new command line, navigate to the root directory of this quickstart, and run the following command, replacing EAP_HOME with the path to your server:

        EAP_HOME/bin/jboss-cli.sh --connect --file=configure-security-domain.cli
This script adds the `quickstart-domain` domain to the `security` subsystem in the server configuration and configures authentication access. You should see the following result when you run the script:

        #1 /subsystem=security/security-domain=quickstart-domain:add(cache-type=default)
        #2 /subsystem=security/security-domain=quickstart-domain/authentication=classic:add
        #3 /subsystem=security/security-domain=quickstart-domain/authentication=classic/login-module=DelegationLoginModule:add(code=org.jboss.as.quickstarts.ejb_security_plus.SaslPlusLoginModule,flag=optional,module-options={password-stacking=useFirstPass})
        #4 /subsystem=security/security-domain=quickstart-domain/authentication=classic/login-module=RealmDirect:add(code=RealmDirect,flag=required,module-options={password-stacking=useFirstPass})
        The batch executed successfully.
        {"outcome" => "success"}



### Review the Modified Server Configuration

After stopping the server, open the `EAP_HOME/standalone/configuration/standalone.xml` file and review the changes.

The EJB side of this quickstart makes use of a new security domain called `quickstart-domain`, which delegates to the `ApplicationRealm`. In order to support identity switching we use the `SaslPlusLoginModule` from this quickstart.

	<security-domain name="quickstart-domain" cache-type="default">
	    <authentication>
	        <login-module code="org.jboss.as.quickstarts.ejb_security_plus.SaslPlusLoginModule" flag="required">
	            <module-option name="password-stacking" value="useFirstPass"/>
	        </login-module>
	        <login-module code="RealmDirect" flag="required">
	            <module-option name="password-stacking" value="useFirstPass"/>
	        </login-module>
	    </authentication>
	</security-domain>

This login module MUST be before the existing `RealmDirect` login module, this means that the `SaslPlusLoginModule` can perform the verification of the remote user whilst the `RealmDirect` login module can load the users roles.

If this approach is used and the majority of requests will involve an identity switch, then it is recommended to have this module as the first module in the list. However, if the majority of requests will run as the connection user with occasional switches, it is recommended to place the `Remoting` login module first and this one second.

This login module will load the properties file `additional-secret.properties` from the deployment. The location of this properties file can be overridden with the module-option `additionalSecretProperties`.

At runtime, this login module is used to obtain the current user from the Remoting connection and verify that an additional supplied authentication token matches the value for that user in the properties file.

For this quickstart we use the following entry: 

	quickstartUser=7f5cc521-5061-4a5b-b814-bdc37f021acc

This means that for quickstartUser to be able to call the EJB the specified authentication token must also be supplied with the request.

Add the Application Users
---------------

This quickstart is built around the default `ApplicationRealm` as configured in the JBoss server distribution. Using the add-user utility script, you must add the following user to the `ApplicationRealm`:

| **UserName** | **Realm** | **Password** | **Roles** |
|:-----------|:-----------|:-----------|:-----------|
| quickstartUser| ApplicationRealm | quiskstartPwd1!| User |

This user is used to both connect to the server and is used for the actual EJB invocation.

For an example of how to use the add-user utility, see the instructions located here: [Add an Application User](https://github.com/jboss-developer/jboss-developer-shared-resources/blob/master/guides/CREATE_USERS.md#add-an-application-user).


### Add Users Manually

Alternatively you can edit the properties file for the users and manually add the required entry:

1. Add the user accounts by editing the file `{jboss.home}/standalone/configuration/application-users.properties` and pasting in the following line:

		quickstartUser=c2d60ae3c894489fa59196c192e351ca

2. Add the users roles by editing the file {jboss.home}/standalone/configuration/application-roles.properties and pasting in the following line: -

		quickstartUser=User

The application server checks the properties files for modifications at runtime so there is no need to restart the server after changing these files.


Start the JBoss Server
-------------------------

1. Open a command line and navigate to the root of the JBoss server directory.
2. The following shows the command line to start the server:

		For Linux:   EAP_HOME/bin/standalone.sh
		For Windows: EAP_HOME\bin\standalone.bat


Build and Deploy the Quickstart
-------------------------

_NOTE: The following build command assumes you have configured your Maven user settings. If you have not, you must include Maven setting arguments on the command line. See [Build and Deploy the Quickstarts](https://github.com/jboss-developer/jboss-developer-shared-resources/blob/master/guides/BUILD_AND_DEPLOY.md#build-and-deploy-the-quickstarts) for complete instructions and additional options._

1. Make sure you have started the JBoss Server as described above.
2. Open a command line and navigate to the root directory of this quickstart.
3. Type this command to build and deploy the archive:

		mvn clean install jboss-as:deploy

4. This will deploy `target/jboss-ejb-security-plus.jar` to the running instance of the server.


Run the client
---------------------

The step here assumes you have already successfully deployed the EJB to the server in the previous step and that your command prompt is still in the same folder.

1.  Type this command to execute the client:

		mvn exec:exec


Investigate the Console Output
----------------------------

When you run the `mvn exec:exec` command, you see the following output.

    * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *
    [Principal={quickstartUser}]
    * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *

This output is only displayed when the client has supplied the correct username, password and authentication token combination.

Re-Run the client
---------------------

You can edit the class `RemoteClient` and update any one of username, password or authentication token.

1.  Type this command to execute the client:

		mvn compile exec:exec

At this point instead of the message shown above you should see a failure.

Undeploy the Archive
--------------------

1. Make sure you have started the JBoss Server as described above.
2. Open a command line and navigate to the root directory of this quickstart.
3. When you are finished testing, type this command to undeploy the archive:

		mvn jboss-as:undeploy


Remove the Security Domain Configuration
----------------------------

You can remove the security domain configuration by running the  `remove-security-domain.cli` script provided in the root directory of this quickstart or by manually restoring the back-up copy the configuration file. 

### Remove the Security Domain Configuration by Running the JBoss CLI Script

1. Start the JBoss server by typing the following: 

        For Linux:  EAP_HOME_SERVER_1/bin/standalone.sh
        For Windows:  EAP_HOME_SERVER_1\bin\standalone.bat
2. Open a new command line, navigate to the root directory of this quickstart, and run the following command, replacing EAP_HOME with the path to your server:

        EAP_HOME/bin/jboss-cli.sh --connect --file=remove-security-domain.cli 
This script removes the `test` queue from the `messaging` subsystem in the server configuration. You should see the following result when you run the script:

        #1 /subsystem=security/security-domain=quickstart-domain:remove
        The batch executed successfully.
        {"outcome" => "success"}


### Remove the Security Domain Configuration Manually
1. If it is running, stop the JBoss server.
2. Replace the `EAP_HOME/standalone/configuration/standalone.xml` file with the back-up copy of the file.



Run the Quickstart in Red Hat JBoss Developer Studio or Eclipse
-------------------------------------

You can also start the server and deploy the quickstarts or run the Arquillian tests from Eclipse using JBoss tools. For more information, see [Use JBoss Developer Studio or Eclipse to Run the Quickstarts](https://github.com/jboss-developer/jboss-developer-shared-resources/blob/master/guides/USE_JBDS.md#use-jboss-developer-studio-or-eclipse-to-run-the-quickstarts) 


Debug the Application
------------------------------------

If you want to debug the source code of any library in the project, run the following command to pull the source into your local repository. The IDE should then detect it.

    mvn dependency:sources
   	
