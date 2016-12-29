# Building Databases with Maven
There are many different build tools that exist for building and compiling code, managing dependencies and executing tests.  That been said, the data persistency and transactional nature of databases can make this hard to achieve in an elegant and simple manner.

One of the biggest issues with database builds is not how to add new functionality or expand the existing structure of a database, but provide a controlled mechanism of rolling back changes that isn’t totally reliant on database backups or snapshots.

As a result a number of different ways of building databases have evolved over the years from simple SQL scripts which call other SQL scripts and run from the command line, through to entire applications aimed at trying to solve the issue.  Whilst none of them completely solve the problems that can be encountered with a database build, each tries to provide its own set of solutions to the same common problems.

From my own professional and personal project experience I have used two or three different methods and read up on several more, but have to say that using the [SQL Maven Plugin](http://www.mojohaus.org/sql-maven-plugin/examples/execute.html) is probably one of the simplest and most reliable methods I have come across.

Before I go any further, I should perhaps talk a little about [Maven](https://maven.apache.org/), and what it is.  For anyone that has ever done much more than a hello world example with Java, Maven is probably a tool that you have come across.  You’ve probably never considered using it for building anything other than Java though.  Other readers are probably now thinking that I have lost my mind to suggest using a tool originally designed for building Java projects could be used to handle database builds.

Through using the SQL-Maven-Plugin, it is possibly to physically separate database changes into different SQL files that can then be run depending on the options passed to Maven from the command line.

Let’s take a look at the following scenario for a better idea.

## New Project
A new project is started by a software team, and they have decided that they are going to use PostgreSQL as their relational database.  Because the team needs to be able to control and track changes to the database structure over time, they have chosen to use [GitHub](https://www.github.com) as their source control.

To get started they will first create a basic database structure, which contains both a deployment and rollback script, and is separated down into the following files.
```
├── pom.xml
├── src
	└── main
		└── deploy
		|	└── example
		|		└── sch_example.sql
		|		└── tables
		|		|	└── users_tbl.sql
		|		|	└── comments_tbl.sql
		|		└── constraints
		|		|	└── pk_usr_id.sql
		|		|	└── pk_cmt_id.sql
		|		|	└── fk_cmt_usr_id.sql
		|		└── functions
		|			└── add_comment_fnc.sql
		└── rollback
			└── example
				└── sch_example.sql
```

These files allow for the basic structure to be setup, and appropriate constraints applied.  Before these files can be run however, it is necessary for the build process to be defined.  This is where the Maven magic happens through a combination of the SQL-Maven-Plugin, [Maven phases](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html) and [Maven profiles](http://maven.apache.org/guides/introduction/introduction-to-profiles.html).  

*It should also be noted that using individual files for different database functionality not only reduces the overall complexity of code (think OO over Procedural), but also allows for easier management in version control.*

### Maven POM
The first step is to create a Maven POM file at the root of the database project, from which the required XML configuration can be added.
```
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>org.example</groupId>
	<artifactId>new-database-build-example</artifactId>
	<version>1.0.0</version>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>
</project>
```

### Maven Plugins
In order for Maven to do anything, it needs to be told what to do.  Out of the box Maven is relatively powerful for Java code, however a little less so for databases.  Fortunately, Maven is designed to allow for plugins to be created to greatly expand its basic capabilities.  Through adding the following simple configuration, it becomes possible for Maven to perform database operations.
```
<build>
	<plugins>
		<plugin>
				<groupId>org.codehaus.mojo</groupId>
				<artifactId>sql-maven-plugin</artifactId>
				<version>1.5</version>

				<dependencies>
					<!-- specify the dependent jdbc driver here -->
					<dependency>
						<groupId>org.postgresql</groupId>
						<artifactId>postgresql</artifactId>
						<version>9.4.1212.jre7</version>
					</dependency>
				</dependencies>

				<!-- common configuration shared by all executions -->
				<configuration>
					<driver>org.postgresql.Driver</driver>
					<url>jdbc:postgresql://127.0.0.1:5432/example_db</url>
					<username>example_user</username>
					<password>password</password>
				</configuration>
		</plugin>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.6.0</version>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
			</configuration>
		</plugin>
	</plugins>
</build>
```
However as the project team is using individual files to physically separate the different elements that make up the database structure, the plugin needs to be told about these files.

These files can be specified in two ways.  Either as absolute file paths, such as:
```
<execution>
	<id>deploy-tables</id>
	<phase>compile</phase>
	<goals>
		<goal>execute</goal>
	</goals>
	<configuration>
		<fileset>
			<includes>
				<include>src/main/deploy/example/tables/users_tbl.sql</include>
			</includes>
		</fileset>
	</configuration>
</execution>
```
Or more conveniently as a partial file mask:
```
<fileset>
	<basedir>src/main/deploy/</basedir>
	<includes>
		<include>**/*_tbl.sql</include>
	</includes>
</fileset>
```
Obviously the order in which certain files gets executed can also be important.  The project team don’t want the build to fail because it tries to deploy the tables, before the schema which they are part of exists.  To get around this, the SQL-Maven-Plugin executes files in the order they are defined within the XML.

Once the project team have updated their POM to include all the different database files described earlier, their POM will look something like this.
```
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>org.example</groupId>
	<artifactId>new-database-build-example</artifactId>
	<version>1.0.0</version>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<deploy.build.dir>src/main/deploy/</deploy.build.dir>
		<rollback.build.dir>src/main/rollback/</rollback.build.dir>
	</properties>

	<build>
		<plugins>
			<plugin>
					<groupId>org.codehaus.mojo</groupId>
					<artifactId>sql-maven-plugin</artifactId>
					<version>1.5</version>

					<dependencies>
						<!-- specify the dependent jdbc driver here -->
						<dependency>
							<groupId>org.postgresql</groupId>
							<artifactId>postgresql</artifactId>
							<version>9.4.1212.jre7</version>
						</dependency>
					</dependencies>

					<!-- common configuration shared by all executions -->
					<configuration>
						<driver>org.postgresql.Driver</driver>
						<url>jdbc:postgresql://127.0.0.1:5432/example_db</url>
						<username>example_user</username>
						<password>password</password>
					</configuration>
					
					<executions>
						<!-- 
							CLEAN PHASE
						-->

						<!-- Rollback FK's -->
						<execution>
							<id>rollback-foreign-keys</id>
							<phase>clean</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${rollback.build.dir}</basedir>
									<includes>
										<include>**/fk_*.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>

						<!-- Rollback PK's -->
						<execution>
							<id>rollback-primary-keys</id>
							<phase>clean</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${rollback.build.dir}</basedir>
									<includes>
										<include>**/pk_*.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>		
						
						<!-- Rollback Tables -->
						<execution>
							<id>rollback-tables</id>
							<phase>clean</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${rollback.build.dir}</basedir>
									<includes>
										<include>**/*_tbl.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>
						
						<!-- Rollback Schemas -->
						<execution>
							<id>rollback-schemas</id>
							<phase>clean</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${rollback.build.dir}</basedir>
									<includes>
										<include>**/sch_*.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>
						
					<!-- 
						COMPILE PHASE
					-->
				
						<!-- Deploy Scehmas -->
						<execution>
							<id>deploy-schemas</id>
							<phase>compile</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${deploy.build.dir}</basedir>
									<includes>
										<include>**/sch_*.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>	
																
						<!-- Deploy Tables -->
						<execution>
							<id>deploy-tables</id>
							<phase>compile</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${deploy.build.dir}</basedir>
									<includes>
										<include>**/*_tbl.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>
											
						<!-- Deploy PK's -->
						<execution>
							<id>deploy-primary-keys</id>
							<phase>compile</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${deploy.build.dir}</basedir>
									<includes>
										<include>**/pk_*.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>
						
						<!-- Deploy FK's -->
						<execution>
							<id>deploy-foreign-keys</id>
							<phase>compile</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<fileset>
									<basedir>${deploy.build.dir}</basedir>
									<includes>
										<include>**/fk_*.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>
						
						<!-- Deploy Functions -->
						<execution>
							<id>deploy-functions</id>
							<phase>compile</phase>
							<goals>
								<goal>execute</goal>
							</goals>
							<configuration>
								<delimiter>/</delimiter>
								<delimiterType>row</delimiterType>
								<fileset>
									<basedir>${deploy.build.dir}</basedir>
									<includes>
										<include>**/*_fnc.sql</include>
									</includes>
								</fileset>
							</configuration>
						</execution>
					
					</executions>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.6.0</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```

### Maven Phases
At this point in developing the POM process things are starting to come together, however there is a little more work required before the first build can be run.  The next thing that needs to be defined, is which files will be run on a rollback, and which will be run on a deployment.

This can simply and easily be done by adding the maven phase to each of the SQL files already defined in the POM.
For a deployment you would include ```<phase>compile</phase>``` and for a rollback ```<phase>clean</phase>```

*More can be read about [Maven phases](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html), and how they are more traditionally used in Java to create a hierarchal build process.*

### Maven Profiles
The final bit of configuration the development team need to add to the POM before they can run the maven build, is to tell it how to connect to a database.  Whilst they have started out with only a development environment, they also know that as the project progresses they will have a CI, Test and Production environment.  

This is exactly where the power of Maven Profiles comes into play.  In this case, it allows for multiple different database instances to have their connection details configured in one location.  The specific database can then be targeted at the time the Maven build is run.

The following XML configuration is all that is needed to connect to the localhost environment, and serves as the basis from which to add future profiles.
```
<profiles>
	<profile>
		<id>local</id>
		<properties>
			<url>jdbc:postgresql://127.0.0.1:5432/example_db</url>
		</properties>
	</profile>
	
	<profile>
		<id>dev</id>
		<properties>
			<url>jdbc:postgresql://192.168.0.10:5432/example_db</url>
		</properties>
	</profile>
</profiles>
```

### Running the Build
Now that the development team have finally completed all the Maven configuration, they are ready to start running their builds against the development environment.  They do this by defining a set of goals (and parameters if required) after the word mvn on the command line.

As maven executes its lifecycle goals in the order specified on the command line, it becomes possible to chain together multiple phases to create advance builds.  An example of this maybe if a Rollback and Deployment are to be run concurrently (more on this later).

*To run a build, maven first needs to be downloaded and setup.  For brevity this means downloading [Maven 3](https://maven.apache.org/download.cgi), and setting it so that it’s /bin directory is available from the PATH environment variable.  Once this is done, it should be possible to run mvn -version from the command line.*

#### Deploy
A deployment can be run without any rollback steps being executed, with the following command.  This will result in only the files marked with a ```<phase>compile</phase>``` being called on.
```
mvn compile -Pdev
```

#### Rollback
A rollback can be run without any deployment steps being executed, with the following command. This will result in only the files marked with a ```<phase>clean</phase>``` being called on.
```
mvn clean -Pdev
```

#### Rollback and Deploy
Whilst there may be times when only a deployment or rollback will be run, it is good-practice to run a rollback immediately before a deployment.  This is to ensure that both the database is put into an ‘expected’ and ‘consistent’ state, but perhaps more importantly validate that the rollback scripts run correctly on that environment.

The following example will execute the scripts marked with ```<phase>clean</phase>``` immediately followed by ```<phase>clean</phase>```
```
mvn clean compile -Pdev
```

#### Security Enhancements
For security reasons it preferable to pass the database password in at run-time, thereby removing the need to store the password in source control.  This can easily be achieved with a [Maven variable](http://books.sonatype.com/mvnref-book/reference/resource-filtering-sect-properties.html) with the following small changes to the POM, where ```${password}``` has replaced the previously hard-coded password.
```
<configuration>
	<driver>org.postgresql.Driver</driver>
	<username>lend_it_spend_it_admin</username>
	<password>${password}</password>
</configuration>
```
With this change applied, the following command is run to perform a clean deployment as before.
```
mvn clean compile -Pdev -Dpassword=pass
```

#### Unit Testing
Whilst outside the scope of this blog post, a future one may well look at how unit tests can be added to a database build and the Maven build amended accordingly.

# The Next Release
Future releases for the development team are much simpler as no additional configuration is required.  Instead, all that is required is some housekeeping of the previously deployed SQL files.

The first step for preparing the next release is to ensure that a [Git tag](https://git-scm.com/book/en/v2/Git-Basics-Tagging) or other way of identifying a point-in-time for the source code has been established.

## POM Changes
Because the project team decided to use filename masks with wildcards, it means that unless a new type of DDL/DML is added, then they won’t have to make any changes (triggers, index’s and other database objects have been omitted from the POM so far).

Additionally, files that existed in the first release and not the subsequent will not cause any issues.  This is because the SQL-Maven-Plugin will simply try to find any match’s for the provided filename masks and skip to the next sequence if it can’t find any. 

## Preparing the File Structure
Once a point-in-time has been recorded, the rollback directory should be cleared out, and the content of the deploy directory copied into it.  At this point a little care needs to be taken to ensure that the code included is able to roll back the database to a state consistent with the successful deployment of the previous release.  This means that when a rollback is run on version 1.1.0 of the database it will return it to version 1.0.0.

At this same time, the deploy directory should be cleared out so that only DDL objects that can be safely re-deployed (stored procedures/functions, triggers etc…) without affecting existing data remain.
```
├── pom.xml
├── src
	└── main
		└── deploy
		|	└── example
		└── rollback
			└── example
				└── functions
					└── add_comment_fnc.sql
```
From here any new SQL files or modifications to existing code can be added and tracked by source control.  The maven commands described previously for deploying and rolling back databases remain the same, only the code they are deploying changes.

# Example Files
A complete example project can be found [here](https://github.com/amelja/maven-database-build-example) which gives you everything you need to get a basic database build working.  You may need to swap the JDBC driver if you plan to use something other than PostgreSQL.
