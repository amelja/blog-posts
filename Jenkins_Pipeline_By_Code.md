In this post we will look at how to setup a Jenkins 2.0 pipeline, entirely through code.  The benefit of this is that you can keep all the build configuration in the same place as the project you are building.  It also means that changes can be more easily tracked and changed than through the traditional GUI.

# Prerequisites

Before we get started, you’ll first need to have a working Jenkins 2.0 instance.  You could either use an existing instance if you have one, or setup your own by registering with Cloudbees or manually installing it on your favourite operating system (Steps for [Ubuntu can be found here](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+Ubuntu)).

Once this is up and running, you will want to install the [Pipeline plugin](https://wiki.jenkins-ci.org/display/JENKINS/Pipeline+Plugin) which will enable you to write [Groovy]( http://www.groovy-lang.org/) scripts against the plugin’s DSL.  

For this post we will be using Git with GitHub, however you can easily adapt the example for BitBucket or other Git based VCS’s.

*For brevity, Maven and a JDK have already been setup and configured in Jenkins.  GitHub credentials have also been setup to use SSH*

# The Initial File
![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_logo.png?raw=true "Jenkins Logo")

Like any good butler, Jenkins is only able to perform tasks for which it is given instructions.  In this instance those instructions come in the form of a single file included with the code base we wish to build.  Whilst it is possible to name the file anything you desire (you will tell Jenkins where to get the file later), we will follow the standard convention of calling it [JenkinsFile](https://github.com/amelja/jenkins-pipeline-as-code-example/blob/master/Jenkinsfile) and locating it at the route of our project.

The JenkinsFile itself is written in Groovy, but doesn’t actually require anyone to be a Groovy or Java wizard to get started.  The following sub-sections try to break down the structure of the example JenkinsFile included with this post.

## Nodes and Stages 

Each step in the Jenkins pipeline is referred to as a stage within the JenkinsFile, and describes an individual and isolated piece of behaviour.  A stage must complete successfully prior to the next been executed.

Within the JenkinsFile, a node dictates which Jenkins instance the stages which it contain run.  In a small environment you won’t need to worry about this too much, but for larger slave-and-master setups this can be used to specify which instance certain pipeline steps run on.

## Version Control

As previously described nodes and stages define where and when parts of a build run, but not how.  In the below syntax, the Jenkins build will checkout the master branch from the [Git based example repository](https://github.com/amelja/jenkins-pipeline-as-code-example) for this blog post into the Jenkins workspace.

```groovy
node("master") {

    //  Version Control Stage
    def gitUri = "git@github.com:amelja/jenkins-pipeline-as-code-example.git"
    def branch = "master"

    stage("Get code from version control") {

        //  Version Control
        echo "Checking out the ${branch} branch from VCS"
        git branch: branch, url: gitUri

    }

}
```

It performs this step on the master node as dictated by the presence of ```node("master")``` at the file start.  The pipeline GUI (see below) will also show the current stage as the value provided to the stage command ```stage("Get code from version control")```.

![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_stage_view_a.png?raw=true "Jenkins Stage View")

*The echo command allows for the console to log additional information as provided to it, which can be useful for debugging or monitoring builds.  The above syntax uses parameters as appropriate, to help keep it clean and make it more re-usable.*

## Building the files with Maven

Now that we have established how to checkout files from VCS, it’s time to get Jenkins to do something more useful with them. 

For this step we will get Jenkins to call Maven to perform the actual building of the Java source code.  All this will do is build a very simple [one-class Spring Boot](https://github.com/amelja/jenkins-pipeline-as-code-example/blob/master/src/main/java/org/example/web/ExampleController.java) based application, to illustrate how a Maven stage can be used.

As the example Maven project has no tests associated with it, we are only going to be using the *Compile* lifecycle goal.

As you will be able to see in the code snippet below, we have a number of new and different commands taking place.  The ```sh``` command simply tells Jenkins to run this line as a shell command, whilst ```${tool 'maven_3'}``` is calling on the ```tool``` command with the environment variable ```maven_3``` (see sub-section Notes on Environment Variables for more info).  The {$mvnGoals} is simply a variable defined elsewhere in the file to pass the argument ```-B verify``` to Maven.

``` sh "${tool 'maven_3'}/bin/mvn ${mvnGoals}"```

You could quite easily change this to a more appropriate lifecycle goal if your build had more maven steps.  If you have many tests you may even want to consider the [Parallel Test Executor plugin](https://wiki.jenkins-ci.org/display/JENKINS/Parallel+Test+Executor+Plugin)  so that tests can be split between nodes and available executors, thereby reducing the overall time for running tests.

At this point our complete JenkinsFile looks something like the below.  As noted earlier, the Pipeline plugin uses Groovy, and therefore fully supports the goodness of Java.  For that reason, you will notice that a try/catch block has been included to help handle the outcome of builds at this stage.

```groovy
node("master") {

    //  Version Control Stage
    def gitUri = "git@github.com:amelja/jenkins-pipeline-as-code-example.git"
    def branch = "master"
	
	//  Build Stage
    def mvnGoals = "-B verify"
	
    stage("Get code from version control") {

        //  Version Control
        echo "Checking out the ${branch} branch from VCS"
        git branch: branch, url: gitUri

    }
	
	stage("Build with Maven") {

        //  Environment Variables
        echo "Performing a Maven ${mvnGoals}"

        try {
            sh "${tool 'maven_3'}/bin/mvn ${mvnGoals}"
            currentBuild.result
        } catch (error) {
			//  Sets build to failed if job Maven fails
            currentBuild.result
        }
    }
}
```

*Whilst Maven has been used as the example here, there is no reason why NPM or any other build tools can’t be used with their own implementation specific commands.*

### Notes on Environment Variables

Maven 3 has been setup with the global variable of maven_3, however you may choose to include the exact version number in yours if you have more than one version of Maven 3 installed.  The same applies for the JDK, which is setup as JDK_8 in this example. 

## Archive with Jenkins

The final step in the Jenkins build pipeline syntax that we are going to cover in any great detail is how to archive the artifacts produced by the build.  Whilst some builds might only run a test suite and therefore not output anything of use (aside from results), other builds will output a compiled JAR, ZIP or other file(s) for later use.

As our example Maven build outputs JAR type files as part of the build process, we want to ensure that they are all made available for later use.  To do this, we add the syntax ```archiveArtifacts artifacts: target/*.jar, fingerprint: true```, which essentially scoops up any JAR files within the target directory, and performs a fingerprint hash on the files.

Note that in the following code example, the file mask is externalised to a variable to make it easier to re-use the same script between different projects.

```groovy
node("master") {

    //  Version Control Stage
    def gitUri = "git@github.com:amelja/jenkins-pipeline-as-code-example.git"
    def branch = "master"
	
	//  Build Stage
    def mvnGoals = "-B verify"
	
	//  File Archive Stage
    def fileMask = "target/*.jar"
	
    stage("Get code from version control") {

        //  Version Control
        echo "Checking out the ${branch} branch from VCS"
        git branch: branch, url: gitUri

    }
	
	stage("Build with Maven") {

        //  Environment Variables
        echo "Performing a Maven ${mvnGoals}"

        try {
            sh "${tool 'maven_3'}/bin/mvn ${mvnGoals}"
            currentBuild.result
        } catch (error) {
			//  Sets build to failed if job Maven fails
            currentBuild.result
        }
    }
	
	stage("Archive in Jenkins") {

        echo "Archiving artifacts to Jenkins"
        archiveArtifacts artifacts: fileMask, fingerprint: true

    }
}
```

# Setting up the Jenkins Job
Once the JenkinsFile has been configured it is time to tell Jenkins about it, so that it can execute it.

The first step is to create a new Jenkins build, and select **Pipeline** from the available templates.

![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_new_job_a.png?raw=true "Jenkins Setup")

You will then need to fill in some basic details such as the **project name**, and a **description** if you feel so inclined.

![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_new_job_b.png?raw=true "Jenkins Setup")

Scrolling down will present you with the options that really give the Pipeline its flexibility and power.  From the **Definition** dropdown you can either choose to provide the build with a script locally, or point it at a VCS source.  For what we are doing in this post, select the remote source control option.  

The below screenshot shows the configuration required to pull code from our remote [example project]( https://github.com/amelja/jenkins-pipeline-as-code-example).  Note that in addition to specifying the correct branch to work from, the **Script Path** contains the name of the file which Jenkins will use to build the Pipeline.

![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_new_job_c.png?raw=true "Jenkins Setup")

# Running Jenkins

Once you have written your pipeline DSL file and saved the new Jenkins project, you are ready to start using it.  

On the first run you will hopefully find the build works without issue, however it also possible that something will cause it to fail.  If this is the case, then the build console will provide details of the error.

The following error message gives an indication of an error in the JenkinsFile script, and provides you with the information necessary to identify and resolve the issue.  In this case, it was because of an incorrectly named variable called ```state```.

![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_error.png?raw=true "Jenkins Error")

Once issues have been identified and resolved, you will need to push the changes to the remote JenkinsFile Git repository before re-trying the Jenkins build.  

Once everything is working, you should expect to see a console similar to this for your jobs.  As you can likely discern, green means good and red means bad!

![alt text](https://github.com/amelja/blog-posts/blob/master/images/jenkins_stage_view_b.png?raw=true "Jenkins Stage View")

# Further Thoughts

Whilst this example has provided a simple framework for performing a build and running automated tests, it could be easily expanded upon to support a more complex Continuous Integration (CI) environment.  

Jenkins also allows the possibility for other tools to be coupled with it, meaning that it can be used as an orchestration service for Continuous Delivery (CD).  One example of this is [Artifactory]( https://www.jfrog.com/open-source/), which has its own [plugin]( https://wiki.jenkins-ci.org/display/JENKINS/Artifactory+-+Working+With+the+Pipeline+Jenkins+Plugin) that greatly simplifies integration between the two systems.  Furthermore you can potentially craft your own Shell scripts (or similar) to deploy built artifacts to Test, Production or any other environment you choose.

## Artifactory Integration
The following is a brief code snippet which you could add to your JenkinsFile to have it push to an in-house Artifactory server.

```groovy
stage("Push to Artifactory") {

	def fileMask = "target/*.jar"

	def server = Artifactory.server("my-artifactory-server")
	
	def uploadSpec = """{
					"files": [
								{
								"pattern": "${fileMask}",
								"target": "example-snapshot/org/example/${projectName}/"
								},
								{
								"pattern": "${fileMask}",
								"target": "example-release/org/example/${projectName}/"
								}
							  ]
					 }"""

	def buildInfo = server.upload spec: uploadSpec

	server.publishBuildInfo buildInfo

}
```