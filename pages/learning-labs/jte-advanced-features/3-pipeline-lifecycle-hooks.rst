.. _JTE Advanced Features Pipeline Lifecycle Hooks: 

------------------------
Pipeline Lifecycle Hooks 
------------------------

There can be interdependent functionalities that presented a challenge to the 
Jenkins Templating Engine.  

For example, let's say we wanted to introduce Splunk monitoring by sending events 
as part of the pipeline? 

How do you: 

1.  Maintain a clean, easy to read Pipeline Template? 
2.  Maintain a separation of duties between libraries as to not hardcode a Splunk integration into every library step? 

It would be great if there was a seemless way to inject functionality in response to different phases 
of the pipeline without having to tightly couple that functionality to existing library steps or 
Pipeline Templates. 

You can!  The Jenkins Templating Engine has a neat feature we call Pipeline Lifecycle Hooks that 
were made for just these situations. 

We'll walk through the Splunk use case to demonstrate this functionality. 

.. note:: 

    Read the `entire pipeline lifecycle hook documentation <https://jenkinsci.github.io/templating-engine-plugin/pages/Library_Development/lifecycle_hooks.html>`_. 

=======================
Create a Splunk Library
======================= 

Methods defined within steps are able to register themselves to correspond to specific lifecycle events 
via annotations.  As such, these steps are typically not invoked directly by other steps or from the 
Pipeline Template. 

Because of this, the name of the step is inconsequential but cannot conflict with other step names that 
are loaded. 

Therefore, we typically recommend following a naming convention of prepending the step name with the 
library name to namespace the hooks.  

.. important:: 

    It doesn't matter what you call steps that only contain Pipeline Lifecycle Hook annotated methods. 
    But to avoid collisions of everyone naming their hook steps ``beforeStep.groovy`` - we recommend 
    ``<libraryName>_<action>`` as we'll demonstrate in this lab.

************************
Notify of Pipeline Start
************************

Within the same Library Source created during JTE: The Basics create a step called 
``splunk_pipeline_start.groovy``: 

**libraries/splunk/splunk_pipeline_start.groovy**: 

.. code:: groovy 

    @Init 
    void call(context){
        println "Splunk: beginning of the pipeline!" 
    }

Breaking down this step, the ``@Init`` registers the ``call`` method defined in this 
step to be invoked at the beginning of the pipeline. 

This ``call`` method takes a Map as an input parameter.  This argument represents the 
pipeline context and will let you know the pipeline's current status as well as the 
library and step that the hook is responding to. 

.. note:: 

    This input parameter to annotated steps is typically called ``context`` but can 
    be called anything.  

    Since Groovy is a dynamically typed language, this input parameter does not have 
    to be given a class. 

*********************************
Update the Pipeline Configuration 
*********************************

In the Pipeline job again, update the Pipeline Configuration to load the ``splunk`` 
library we just created. 

.. code:: groovy

    libraries{
        maven
        sonarqube
        ansible
        splunk
    }

That's it! Just by loading the library, JTE will be able to find the methods within steps 
annotated with a Pipeline Lifecycle Hook. 

Run the Pipeline job and you should see output in the logs similar to:

.. code-block:: text 

    [JTE] [@Init - splunk/splunk_pipeline_start.call]
    [Pipeline] echo
    Splunk: beginning of the pipeline!


*****************************************
Add Before and After Step Execution Hooks
*****************************************

Let's add some hooks that inject themselves both before and after each step is executed in the pipeline. 

**libraries/splunk/splunk_step_watcher.groovy**: 

.. code:: 

    @BeforeStep
    void before(context){
    println "Splunk: running before the ${context.library} library's ${context.step} step" 
    }

    @AfterStep
    void after(context){
    println "Splunk: running after the ${context.library} library's ${context.step} step" 
    }

.. note:: 
    
    Here, we're defining two different methods in a single step.  In the next section we'll talk 
    about this in more detail.  

    For right now, the important piece is that the method's have the ``@BeforeStep`` and ``@AfterStep`` 
    annotations. 

Rerunning the pipeline, we can now see these hooks get executed: 

.. code-block:: text 

    [Pipeline] Start of Pipeline
    [JTE] Pipeline Configuration Modifications (show)
    [JTE] Loading Library maven (show)
    [JTE] Library maven does not have a configuration file.
    [JTE] Loading Library sonarqube (show)
    [JTE] Library sonarqube does not have a configuration file.
    [JTE] Loading Library ansible (show)
    [JTE] Library ansible does not have a configuration file.
    [JTE] Loading Library splunk (show)
    [JTE] Library splunk does not have a configuration file.
    [JTE] Creating step unit_test from the default step implementation.
    [JTE] Obtained Pipeline Template from job configuration
    [Pipeline] node
    Running on Jenkins in /var/jenkins_home/workspace/single-job
    [Pipeline] {
    [Pipeline] writeFile
    [Pipeline] archiveArtifacts
    Archiving artifacts
    [Pipeline] }
    [Pipeline] // node
    [JTE] [@Init - splunk/splunk_pipeline_start.call]
    [Pipeline] echo
    Splunk: beginning of the pipeline!
    [JTE] [Stage - continuous_integration]
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the Default Step Implementation library's unit_test step
    [JTE] [Step - Default Step Implementation/unit_test.call()]
    [Pipeline] stage
    [Pipeline] { (Unit Test)
    [Pipeline] node
    Running on Jenkins in /var/jenkins_home/workspace/single-job
    [Pipeline] {
    [Pipeline] isUnix
    [Pipeline] sh
    + docker inspect -f . maven
    .
    [Pipeline] withDockerContainer
    Jenkins seems to be running inside container cc7140d4fb91bef940e2fabe7225dcbcc9b44a3a5e17ee703b8fcbe42e53a17c
    $ docker run -t -d -u 0:0 -w /var/jenkins_home/workspace/single-job --volumes-from cc7140d4fb91bef940e2fabe7225dcbcc9b44a3a5e17ee703b8fcbe42e53a17c -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** maven cat
    $ docker top ead0198246fc908dfb815941ae07227b849ab092b49c9f9db59c46b24718b9d8 -eo pid,comm
    [Pipeline] {
    [Pipeline] unstash
    [Pipeline] sh
    + mvn -v
    Apache Maven 3.6.2 (40f52333136460af0dc0d7232c0dc0bcf0d9e117; 2019-08-27T15:06:16Z)
    Maven home: /usr/share/maven
    Java version: 11.0.5, vendor: Oracle Corporation, runtime: /usr/local/openjdk-11
    Default locale: en, platform encoding: UTF-8
    OS name: "linux", version: "4.9.125-linuxkit", arch: "amd64", family: "unix"
    [Pipeline] }
    $ docker stop --time=1 ead0198246fc908dfb815941ae07227b849ab092b49c9f9db59c46b24718b9d8
    $ docker rm -f ead0198246fc908dfb815941ae07227b849ab092b49c9f9db59c46b24718b9d8
    [Pipeline] // withDockerContainer
    [Pipeline] }
    [Pipeline] // node
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the Default Step Implementation library's unit_test step
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the maven library's build step
    [JTE] [Step - maven/build.call()]
    [Pipeline] stage
    [Pipeline] { (Maven: Build)
    [Pipeline] echo
    build from the maven library
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the maven library's build step
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the sonarqube library's static_code_analysis step
    [JTE] [Step - sonarqube/static_code_analysis.call()]
    [Pipeline] stage
    [Pipeline] { (SonarQube: Static Code Analysis)
    [Pipeline] echo
    static code analysis from the sonarqube library
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the sonarqube library's static_code_analysis step
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the ansible library's deploy_to step
    [JTE] [Step - ansible/deploy_to.call(ApplicationEnvironment)]
    [Pipeline] stage
    [Pipeline] { (Deploy To: dev)
    [Pipeline] echo
    performing a deployment through ansible..
    [Pipeline] echo
    deploying to 0.0.0.1
    [Pipeline] echo
    deploying to 0.0.0.2
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the ansible library's deploy_to step
    [Pipeline] timeout
    Timeout set to expire in 5 min 0 sec
    [Pipeline] {
    [Pipeline] input
    Approve the deployment?
    Proceed or Abort
    Approved by admin
    [Pipeline] }
    [Pipeline] // timeout
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the ansible library's deploy_to step
    [JTE] [Step - ansible/deploy_to.call(ApplicationEnvironment)]
    [Pipeline] stage
    [Pipeline] { (Deploy To: Production)
    [Pipeline] echo
    performing a deployment through ansible..
    [Pipeline] echo
    deploying to 0.0.1.1
    [Pipeline] echo
    deploying to 0.0.1.2
    [Pipeline] echo
    deploying to 0.0.1.3
    [Pipeline] echo
    deploying to 0.0.1.4
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the ansible library's deploy_to step
    [Pipeline] End of Pipeline
    Finished: SUCCESS

***********************************
Notify of End of Pipeline Execution
***********************************

Let's try out one more hook to get executed when the pipeline has finished: 

**libraries/splunk/splunk_pipeline_end.groovy**: 

.. code:: 

    @CleanUp
    void call(context){
        println "Splunk: end of the pipeline!" 
    }

Run the pipeline again and you should see logs similar to: 

.. code-block:: text

    [Pipeline] Start of Pipeline
    [JTE] Pipeline Configuration Modifications (show)
    [JTE] Loading Library maven (show)
    [JTE] Library maven does not have a configuration file.
    [JTE] Loading Library sonarqube (show)
    [JTE] Library sonarqube does not have a configuration file.
    [JTE] Loading Library ansible (show)
    [JTE] Library ansible does not have a configuration file.
    [JTE] Loading Library splunk (show)
    [JTE] Library splunk does not have a configuration file.
    [JTE] Creating step unit_test from the default step implementation.
    [JTE] Obtained Pipeline Template from job configuration
    [Pipeline] node
    Running on Jenkins in /var/jenkins_home/workspace/single-job
    [Pipeline] {
    [Pipeline] writeFile
    [Pipeline] archiveArtifacts
    Archiving artifacts
    [Pipeline] }
    [Pipeline] // node
    [JTE] [@Init - splunk/splunk_pipeline_start.call]
    [Pipeline] echo
    Sending Splunk event for beginning of the pipeline!
    [JTE] [Stage - continuous_integration]
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the Default Step Implementation library's unit_test step
    [JTE] [Step - Default Step Implementation/unit_test.call()]
    [Pipeline] stage
    [Pipeline] { (Unit Test)
    [Pipeline] node
    Running on Jenkins in /var/jenkins_home/workspace/single-job
    [Pipeline] {
    [Pipeline] isUnix
    [Pipeline] sh
    + docker inspect -f . maven
    .
    [Pipeline] withDockerContainer
    Jenkins seems to be running inside container cc7140d4fb91bef940e2fabe7225dcbcc9b44a3a5e17ee703b8fcbe42e53a17c
    $ docker run -t -d -u 0:0 -w /var/jenkins_home/workspace/single-job --volumes-from cc7140d4fb91bef940e2fabe7225dcbcc9b44a3a5e17ee703b8fcbe42e53a17c -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** maven cat
    $ docker top 109ac04fcc911f8df3ca5281720f50886497045230b43ae2a6ca4e9b1b0b1271 -eo pid,comm
    [Pipeline] {
    [Pipeline] unstash
    [Pipeline] sh
    + mvn -v
    Apache Maven 3.6.2 (40f52333136460af0dc0d7232c0dc0bcf0d9e117; 2019-08-27T15:06:16Z)
    Maven home: /usr/share/maven
    Java version: 11.0.5, vendor: Oracle Corporation, runtime: /usr/local/openjdk-11
    Default locale: en, platform encoding: UTF-8
    OS name: "linux", version: "4.9.125-linuxkit", arch: "amd64", family: "unix"
    [Pipeline] }
    $ docker stop --time=1 109ac04fcc911f8df3ca5281720f50886497045230b43ae2a6ca4e9b1b0b1271
    $ docker rm -f 109ac04fcc911f8df3ca5281720f50886497045230b43ae2a6ca4e9b1b0b1271
    [Pipeline] // withDockerContainer
    [Pipeline] }
    [Pipeline] // node
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the Default Step Implementation library's unit_test step
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the maven library's build step
    [JTE] [Step - maven/build.call()]
    [Pipeline] stage
    [Pipeline] { (Maven: Build)
    [Pipeline] echo
    build from the maven library
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the maven library's build step
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the sonarqube library's static_code_analysis step
    [JTE] [Step - sonarqube/static_code_analysis.call()]
    [Pipeline] stage
    [Pipeline] { (SonarQube: Static Code Analysis)
    [Pipeline] echo
    static code analysis from the sonarqube library
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the sonarqube library's static_code_analysis step
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the ansible library's deploy_to step
    [JTE] [Step - ansible/deploy_to.call(ApplicationEnvironment)]
    [Pipeline] stage
    [Pipeline] { (Deploy To: dev)
    [Pipeline] echo
    performing a deployment through ansible..
    [Pipeline] echo
    deploying to 0.0.0.1
    [Pipeline] echo
    deploying to 0.0.0.2
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the ansible library's deploy_to step
    [Pipeline] timeout
    Timeout set to expire in 5 min 0 sec
    [Pipeline] {
    [Pipeline] input
    Approve the deployment?
    Proceed or Abort
    Approved by admin
    [Pipeline] }
    [Pipeline] // timeout
    [JTE] [@BeforeStep - splunk/splunk_step_watcher.before]
    [Pipeline] echo
    Splunk: running before the ansible library's deploy_to step
    [JTE] [Step - ansible/deploy_to.call(ApplicationEnvironment)]
    [Pipeline] stage
    [Pipeline] { (Deploy To: Production)
    [Pipeline] echo
    performing a deployment through ansible..
    [Pipeline] echo
    deploying to 0.0.1.1
    [Pipeline] echo
    deploying to 0.0.1.2
    [Pipeline] echo
    deploying to 0.0.1.3
    [Pipeline] echo
    deploying to 0.0.1.4
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [@AfterStep - splunk/splunk_step_watcher.after]
    [Pipeline] echo
    Splunk: running after the ansible library's deploy_to step
    [JTE] [@CleanUp - splunk/splunk_pipeline_end.call]
    [Pipeline] echo
    Splunk: end of the pipeline!
    [Pipeline] End of Pipeline

==========================
Restricting Hook Execution
==========================

What if we only wanted to execute the ``@AfterStep`` hook to be executed 
after the ``static_code_analysis`` step? 

Pipeline Lifecycle Hook annotations accept a **Closure** parameter.  This 
Closure will be executed, and if the return of the Closure is non-false the 
step will be executed. 

.. important:: 

    Remember: Groovy has implicit return statements.  The last statement made 
    becomes the return object by default.  

We call this functionality **Conditional Hook Execution**. 

************************************
Update the ``@AfterStep`` Annotation
************************************ 

Let's see it in action. 

Update the ``@AfterStep`` created in **libraries/splunk/pipeline_step_watcher.groovy** to: 

.. code:: groovy 

    @AfterStep({ context.step.equals("static_code_analysis") })

Rerun the pipeline and notice that now, the hook has been restricted to only run after the desired 
step. 

.. important:: 

    When the ``Closure`` parameter is invoked, it will have access to the ``context`` variable that 
    is passed to the step itself as well as the library configuration that is stored via the ``config`` 
    variable. 

************************
Taking It A Step Further
************************

It would be even better if we could externalize the configuration of exactly which steps 
the ``@AfterStep`` hook should be triggered.  

To do this, update the ``@AfterStep`` annotation again to be: 

.. code:: groovy 

    @AfterStep({ context.step in config.afterSteps })

Now, we can conditionally execute the hook by checking if the name of the step that was just executed 
is in an array called ``afterSteps`` defined as part of the ``splunk`` library in the Pipeline Configuration!

Update the ``splunk`` portion of the Pipeline Configuration to: 

.. code:: groovy 

    libraries{
        maven
        sonarqube
        ansible
        splunk{
            afterSteps = [ "static_code_analysis", "unit_test"  ]
        }
    }

Run the pipeline again and notice that the hook was only executed after the steps defined in the Pipeline Configuration. 

.. note:: 

    Conditional Execution Closure Parameters can be passed to any Pipeline Lifecycle Hook annotation. 
    As long as the Closure returns a non-false value, the hook will be invoked. 


**Remember to read through the `Pipeline Lifecycle Hook documentation <https://jenkinsci.github.io/templating-engine-plugin/pages/Library_Development/lifecycle_hooks.html>`_ 
to see all the annotations available.**