.. _JTE Primitives Application Environments: 

------------------------
Application Environments
------------------------

===================================
What is an Application Environment?
===================================

Performing an automated deployment is a ubiquitious step in continuous delivery pipelines. 

Libraries can implement steps to perform deployments and when doing so, need a mechanism to 
tell the deployment step which application environment is being deployed to. 

The Application Environment acts to encapsulate the contextual information that identifies 
and differentiates an applications different application environments. 

In general, library steps should not accept input parameters.  Deployment steps are one of 
the few exceptions to this rule. 

.. note:: 

    `View the Application Environment documentation <https://jenkinsci.github.io/templating-engine-plugin/pages/Primitives/application_environments.html>`_

*******************************************
A Word on Input Parameters to Library Steps 
*******************************************

Understanding *why*, in general, library steps should not accept input parameters is fundamental to 
understanding the goals of the Jenkins Templating Engine. 

Let's say we had some teams in an organization leveraging SonarQube for static code analysis and 
others using Fortify.  If the ``static_code_analysis`` step implemented by the ``sonarqube`` library 
took input parameters - it would then require that **every** library that implements ``static_code_analysis`` 
take input parameters lest you break the interchangeability of libraries to use the same template. 

This would mean that the ``fortify``'s implementation would have to take the **same** input parameters 
as the ``sonarqube``'s implementation - otherwise switching between the two would break. 

Being able to swap implementations of steps in an out through different libraries is the primary mechanism 
through which JTE supports creating reusable, tool-agnostic pipeline templates.  

It's understandable that library steps require some externalized configuration as to not hard-code dependencies 
like server locations, thresholds for failure, etc.  This is why library configuration is done through the 
pipeline configuration file and passed directly to the steps through an autowired variable as opposed to directly 
via input parameters.  

.. note:: 

    You can read more about how to externalize library configuration
    `here <https://jenkinsci.github.io/templating-engine-plugin/pages/Library_Development/externalizing_config.html>`_.

Deployment steps are different.  It is safe to assume that every step that performs a deployment needs to know some 
environment context.

=========================================
Define and Use an Application Environment
=========================================

***************************
Create a Deployment Library
***************************

Let's actually create a mock deployment library to demonstrate the utility of Application Environments.

In the same repository used during JTE: The Basics, add a new library called ``ansible`` with a step 
called ``deploy_to`` that takes an Application Environment parameter

.. note:: 
    
    Remember that libraries are just subdirectories within a source code repository and that steps are 
    just groovy files in those subdirectories that typically implement a ``call`` method. 

For the sake of our pretend ``ansible`` library, let's assume that it needs to know a list of IP addresses 
relevant to the environment it's deploying too. 

**libraries/ansible/deploy_to.groovy**: 

.. code:: groovy 

    void call(app_env){
        stage("Deploy To: ${app_env.long_name}"){
            println "performing a deployment through ansible.."
            app_env.ip_addresses.each{ ip ->
                println "deploying to ${ip}"
            }
        }
    }


This step will announce it's performing an ansible deployment and then iterate over the 
IP addresses provided on the application environment and print out the target server. 

.. note:: 

    The file structure within your ``libraries`` directory should now be: 

    .. code:: 

        .
        ├── libraries
            ├── ansible
            │   └── deploy_to.groovy
            ├── gradle
            │   └── build.groovy
            ├── maven
            │   └── build.groovy
            └── sonarqube
                └── static_code_analysis.groovy

*****************************************************************
Define the Application Environments in the Pipeline Configuration
*****************************************************************

We now need to load the ``ansible`` library and define the Application Environments. 

Update the **Pipeline Configuration** to: 

.. code:: groovy 

    libraries{
        maven
        sonarqube
        ansible
    }

    stages{
        continuous_integration{
            build
            static_code_analysis
        }
    }

    application_environments{
        dev{
            ip_addresses = [ "0.0.0.1", "0.0.0.2" ]
        }
        prod{
            long_name = "Production" 
            ip_addresses = [ "0.0.1.1", "0.0.1.2", "0.0.1.3", "0.0.1.4" ]
        }
    }

.. important:: 

    Application Environments are defined in the ``application_environments`` block 
    within the pipeline pipeline configuration. 

    Each key defined in this block will represent an Application Environment and a
    variable will be made available in the pipeline template based upon this name. 

    The only two keys that Application Environments explicitly define are ``short_name`` 
    and ``long_name``.  These values default to the key defininig the Application Environment 
    in the Pipelien Configuration, but can be overridden. 

****************************
Update the Pipeline Template
****************************

Now that we have a library that performs a deployment step and Application Environments
defined in the Pipeline Configuration, let's update the Pipeline Template to pull it all together.

Update the **Pipeline Template** to: 

.. code:: groovy

    continuous_integration() 
    deploy_to dev 
    deploy_to prod 

.. note:: 

    These variables ``dev`` and ``prod`` come directly from the Applications Environments we 
    just defined in the Pipeline Configuration. 

****************
Run the Pipeline
**************** 

From the Pipeline job's main page, click ``Build Now`` in the lefthand navigation menu. 

When viewing the build logs, you should see output similar to:

.. code-block:: text 

    [Pipeline] node
    Running on Jenkins in /var/jenkins_home/workspace/single-job
    [Pipeline] {
    [Pipeline] writeFile
    [Pipeline] archiveArtifacts
    Archiving artifacts
    [Pipeline] }
    [Pipeline] // node
    [JTE] [Stage - continuous_integration]
    [JTE] [Step - maven/build.call()]
    [Pipeline] stage
    [Pipeline] { (Maven: Build)
    [Pipeline] echo
    build from the maven library
    [Pipeline] }
    [Pipeline] // stage
    [JTE] [Step - sonarqube/static_code_analysis.call()]
    [Pipeline] stage
    [Pipeline] { (SonarQube: Static Code Analysis)
    [Pipeline] echo
    static code analysis from the sonarqube library
    [Pipeline] }
    [Pipeline] // stage
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
    [Pipeline] End of Pipeline
    Finished: SUCCESS

Notice the output was different for the deployment to the ``dev`` environment vs the deployment to ``prod``.  This is 
because different values were stored in each Application Environment and the library was able to use this contextual 
information and respond accordingly. 