#Once maven is installed, clone this project.

#change your current working directory to the project directory where this project was cloned, and run the below command.

mvn package


This post aims to provide a quick and easy method to packaging files as RPMs using RPM Maven plugin. The steps are introduced in such a way that one with no prior knowledge of maven should still be able to start using this tool for packaging files into RPMs.
Clone demo from github
fork-git
Quick Introduction to Maven

Maven, at a very basic level, is a build tool for packaging files into different formats such as zip, tar, jar, war etc, from a project, and is commonly used in packaging Java projects as jar files, in Software development. Though Maven can do more that just building packages, in this post, we will only walk you through the steps towards building RPMs using Maven, so that we do not go off track.
Installing Maven

Once you’ve enabled EPEL repository on your system, Installing maven is as simple as running the command “yum install maven“. You may also install maven using the steps recommended on the Apache Maven website.
What is pom.xml?

To achieve our simple goal of just packaging files as RPMs, all we need to know about this file is that, It is an XML file that contains information about the project and configuration details used by Maven to build the project. And, a project is nothing but a directory that contains the files that you’re trying to package into an RPM.

To better understand what goes in pom.xml, we will work out an example where in, we build an RPM with the following objectives.

    Upon installation of this RPM on a system, a sample .ini file (let’s just call it sample.ini) should get installed as /opt/myapp.ini.
    The user and group for this file should be myappid and myappgroup respectively
    The file should have 755 permissions.
    A script called “preinstall.sh” should run before
    /opt/myapp.ini is installed. We should use this script to check if the user and group exists and if not, we should create one.
    A script called “postinstall.sh” should run after
    /opt/myapp.ini is installed. We should use this script to restart httpd service.

NOTE: when you trigger the command “mvn package” to begin packaging, mvn (binary of Maven) looks for instruction from pom.xml in the current working directory. Therefore, a pom.xml is being expected in the current working directory. The same directory will ideally contain the project files that you’re trying to package into an RPM.

Our project directory looks like this before the RPM is built.

[admin@unixutils project]# tree
.
├── ini
│   └── sample.ini
├── pom.xml
└── scripts
    ├── postinstall.sh
    └── preinstall.sh

2 directories, 4 files

To achieve our objectives, we will write our objectives as instructions in the form of XML tags in our project file pom.xml. pom.xml typically begins with a <project> tag and ends with a </project> tag. This tag specifies the XML namespace for the document and it does not change. So does the <modelVersion>4.0.0</modelVersion>, which is the descriptor version used in the file for maven to be able to understand the contents of pom.xml. This does not change either. There is no need to give much thought into how these two tags affects the build, however, you can always look further on the Apache’s Maven – Introduction to POM.

<groupId>, <artifactId> & <version> are the three tags that uniquely identifiy a package after it is being built. Every package built with Maven will have these. A package is generally called an Artifact, and companies/communities build Artifacts and store them in a private repository such as Artifactory or the public Maven repository. Suppose a company named company.com builds packages, each of the packages will have an unique artifactId, version and a common groupId to indicate that it was built by the company’s group. The groupId is generally the company’s domain name in reverse (com.company). Thus, every package that is put in the repository is referenced uniquely using these three tags. This wraps up the basics of pom.xml. For building a java project and packaging .jar files, this is all it takes. At this point if the mvn build command is triggered, mvn automatically looks for source files under (src/main/java) in the working directory and compiles them and packages them as .jar. However we neither have a (src/main/java) directory structure nor java files to compile and build. All we have is some other files (that does not need compilation), that we intend to package as rpm. for this, we will make use of the RPM maven plugin. Plugins like these are again nothing but packages or artifacts that was built and made available on the Official Maven repository by communities or companies as an open source. As we just learnt, an artifact is uniquely identified by the three tags namely <groupId>, <artifactId> & <version>, In order to plug-in and use an external artifact/package, we need to call the plugin by its groupId, artifactId and version and include it under the <build></build> tags.

In the below pom.xml project file, we will be using the RPM Maven plugin, for which the <groupId>, <artifactId> & <version> values are “org.codehaus.mojo”, “rpm-maven-plugin” & “2.0.1” respectively (lines #12 to #14 in the below pom.xml). The tags that goes in after this, is whatever the developer of the plugin has defined. The usage of this plug-in can be found on the Plugin developer’s official page. Based on the usage document, we define the configuration (lines #15 to #38 in the below pom.xml), explained further below.
pom.xml

    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                          http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <groupId>com.unixutils</groupId>
        <artifactId>unixutils-test-rpm</artifactId>
        <version>1.0.0</version>
        <build>
          <plugins>
            <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>rpm-maven-plugin</artifactId>
            <version>2.0.1</version>
            <executions>
             <execution>
             <goals>
             <goal>rpm</goal>
             </goals>
            </execution>
           </executions>
              <configuration>
                        <copyright>2019, UnixUtils</copyright>
                        <group>Development/Tools</group>
                        <release>1.0.0</release>
                        <mappings>
                            <mapping>
                                <directory>/opt/myapp.ini</directory>
                                <username>myuser</username>
                                <groupname>mygroup</groupname>
                                <filemode>755</filemode>
                                <sources>
                                    <source>
                                        <location>${project.basedir}/ini/sample.ini</location>
                                    </source>
                                </sources>
                            </mapping>
                        </mappings>
                        <preinstallScriptlet>
                            <scriptFile>${project.basedir}/scripts/preinstall.sh</scriptFile>
                        </preinstallScriptlet>
                       <postinstallScriptlet>
                            <scriptFile>${project.basedir}/scripts/postinstall.sh</scriptFile>
                            <fileEncoding>utf-8</fileEncoding>
                       </postinstallScriptlet>
              </configuration>
             </plugin>
           </plugins>
         </build>
    </project>

Under <execution>, the <goal> defines that an RPM has be created, which will be the artifact/package (the end result of this build). <mapping> block defines that, the files from the project location/working directory specified in the <source> section gets packaged into an rpm, and when the rpm is installed on a system, it is deployed in the path specified in the <directory> section. username, group-name and permissions are specified under <username>, <groupname> & <filemode> sections respectively.

Note that ${project.basedir} is a predefined maven variable that denotes the current working directory where the project files are located. Optionally, <preinstallScriplet> can be used to define path to a script from the project location, that will be run before the rpm is installed. <postinstallScriptlet> can be used to define path to a script from the project location, that will be run after the rpm is installed.

mvn package

The above command starts building the RPM package, and dumps them into the below location

project/target/rpm/unixutils-test-rpm/RPMS/noarch/unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm

Now let’s try to install the package that was created.

[admin@unixutils project]# cd target/rpm/unixutils-test-rpm/RPMS/noarch
[admin@unixutils noarch]# yum localinstall unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm
Loaded plugins: fastestmirror
Examining unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm: unixutils-test-rpm-1.0.0-1.0.0.noarch
Marking unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package unixutils-test-rpm.noarch 0:1.0.0-1.0.0 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

====================================================================================================================================================================
 Package                                Arch                       Version                         Repository                                                  Size
====================================================================================================================================================================
Installing:
 unixutils-test-rpm                     noarch                     1.0.0-1.0.0                     /unixutils-test-rpm-1.0.0-1.0.0.noarch                      60

Transaction Summary
====================================================================================================================================================================
Install  1 Package

Total size: 60
Installed size: 60
Is this ok [y/d/N]: y
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
running command [id -u myuser]
id: myuser: no such user
myuser does not exist..
creating myuser..
running command [getent group mygroup]
mygroup does not exist..
creating mygroup..
  Installing : unixutils-test-rpm-1.0.0-1.0.0.noarch                                                                                                            1/1
restarting HTTPD services
  Verifying  : unixutils-test-rpm-1.0.0-1.0.0.noarch                                                                                                            1/1

Installed:
  unixutils-test-rpm.noarch 0:1.0.0-1.0.0

Complete!
[admin@unixutils noarch]#

The RPM is installed successfully. The postinstall and preinstall scripts also had run. Here we also see that the ini file is deployed as intended with the correct permissions and ownership.

[admin@unixutils project]# ll /opt/myapp.ini/
total 4
-rwxr-xr-x. 1 myuser mygroup 60 Jan  5 17:39 sample.ini
[admin@unixutils project]#

In this post, we explored the basics of using maven as a tool to create RPMs with “RPM Maven Plugin”. In the future posts, we will further explore the different features available with in this plugin and see different scenarios such as how to deploy an RPM update.
