# Jenkins tutorial in Docker with Agents

## Installing Jenkins

Below is the compose file for Jenkins container. 

```YAML
version: '3.9'
services: 
  jenkins:
    image: jenkins/jenkins:latest
    privileged: true
    ports:
      - 8080:8080
      - 50000:50000
    container_name: jenkins_container
    user: jenkins    # user: jenkins, for more info see under official docker image -> tags -> latest -> USER variable
    environment:
      - JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /home/user/Documents/demo-jenkins-project:/home/demo-jenkins-project:ro
    networks:
      - jenkins-network

volumes:
  jenkins_home:
    name: "jenkins_home"
    # external: true # if volume already exists

networks:
  jenkins-network:
    name: "jenkins-network"
```

> [!NOTE]  
> The default `user` set by _Dokerfile_ is `jenkins`. This can be verified by going to _tags_ under the official docker image for Jenkins. 

The volume top level element with name set to `Jenkins_home`, declares a named volume which can be reused. This way Jenkins data will be persistent even if the container is shut down. 

Next we use a _bind mount_ to mount _demo-Jenkins-project_ directiory containing _Jenkinsfile_ to mount inside Jenkins container. 

We then set the environment variable `JAVA_OPTS` with the value `-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true` which allows us to checkout from a local git repository (You may checkout from github or other SCM repository e.g., Github also).

For more details on the other options checkout the compose file specification. 

Now to start Jenkins enter the command 
```bash
sudo docker compose up -d
```

follow the prompts and setup the initial configuration for Jenkins. 

## Setup agents

Next we will configure the agents.  

**Dockerfile**
```dockerfile
FROM ubuntu:jammy

RUN apt-get update

RUN apt-get install -y openssh-client openssh-server openjdk-11-jdk
```
In the above *Dockerfile* the base image for the agent is *ubuntu jammy jellyfish* (personal choice). You may choose whichever distro you want. But the main idea is the same.  

We will follow the steps mentioned in [Ssh from one container to another container](https://stackoverflow.com/questions/53984274/ssh-from-one-container-to-another-container)

To build image from the Dockerfile navigate to the folder containing it and run the following command
```bash
sudo docker build -t name:tag .
```

In my case it is `jammy:v2`. 

Let's create an agent named `agent1`.  

```bash
sudo docker run --rm -ti -p 22 --network jenkins-network --name agent1 jammy:v2 bash
```

It will create `agent1` with port 22 bound to a random port in host. It will also add the agent to `jenkins-network` that we declared before. The `--rm` flag ensures that as soon as we exit the container the container will be removed. The `-ti` flag opens a pseudo tty interactive shell inside the container.  

start the ssh daemon 
```bash
mkdir /var/run/sshd
chmod 0755 /var/run/sshd
/usr/sbin/sshd
```

Create a user along with it's password
```bash
useradd --create-home --shell /bin/bash --groups sudo u1
passwd u1
```

also note down the hostname by executing the `hostname` command in the agent.  

Enter into jenkins container using the command
```bash
docker exec -it <container-id> bash
```

and try ssh'ing into the agent 
```bash
ssh -X u1@<hostname>
```

follow the prompt and enter the password when asked, if everything goes without error then it should successfully SSH from Jenkins master to agent. 

Now configure the agent in `Manage Jenkins` section and ensure that under Launch Method _Launch agents via SSH_ is selected. 

Now we have successfully added the agent. We can try out some jobs using the agent. 

For example below is a sample Jenkinsfile

**Jenkinsfile**
```groovy
pipeline {
    agent {
        label '<label>'
    }

    environment {
        VERSION = '1.0.0'
    }

    stages {
    
        stage("build") {
            
            steps {
                echo 'building the application'
                echo "application version ${VERSION}"
            }
        }

        stage("test") {
            
            steps {
                echo 'testing the application'
            }
        }

        stage("deploy") {
        
            when {
                expression {
                    BRANCH_NAME == 'main'
                }
            }
            
            steps {
                echo 'deploy the application'

                withCredentials([usernamePassword(
                    credentialsId: '<credentialsID>',
                    passwordVariable: 'PASSWORD',
                    usernameVariable: 'USERNAME' 
                )]) {
                    // Do something 
                }
            }
        }
    }
}
```

## Clean up
To stop the agents just type 
```bash
exit
```
in the command line.
To stop the jenkins container go the directory where compose file is located and run
```bash
sudo docker compose down
```
