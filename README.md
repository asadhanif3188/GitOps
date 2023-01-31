# End to end  Deployment in Kubernetes clusters using Jenkins, GitOps and GitHub Pipeline

### What are we going to achive in this project?
We have an application running in kubernetes, that application is saying *Hello, My Docker project!*, and the magic starts happening when we change the code and push it to github. As soon as you commit the changes, a jenkins job get submitted automatically, builds a new image, pushes the image to Dockerhub, changes the deployment file with the latest image id, the new image automatically gets deployed to the kubernetes cluster using gitops, and our application starts pointing to the new pod.

### What will be covered in this project:
* GitOps Workflow
  * Difference with DevOps Workflow
* Dockerfile and Jenkinsfile Walkthrough
* Jenkins Installation
* Jenkins Jobs Setup
* ArgoCD (GitOps) Installation
* ArgoCD (GitOps) Setup
* Automating GitHub to jenkins using Webhook
* Zero touch end to end (nirvana!)


![image](https://user-images.githubusercontent.com/115881685/209542342-7247a5d7-f6cd-43c0-b419-7d8b82ab7be6.png)

### In Summary GitOps:
* Periodically syncs the running cluster with the desired state in Git Repo
* Works with both vanilla manifest files or Helm charts
* Reduced learning curve than Devops
* Increased security
  * CI (Developer) and CD (Ops) permissions are seperated
* GitOps doesn't mean getting rid of DevOps

### Dockerfile and Jenkinsfile Walkthrough

Now lets jump in the Github repository first.

![repo](https://user-images.githubusercontent.com/115881685/215599012-c12f4fcc-7ff2-40c3-8d61-1d9f37850885.png)
This is the kubernetescode repository where we have our applicationfile and dockerfile.


![repo 2](https://user-images.githubusercontent.com/115881685/215599843-cb39bd01-d67f-409d-b036-74d09f934678.png)
Our application code is app.py. It is a very simple python program, which is importing the library flask and just returning Hello, Docker project!


![repo 3](https://user-images.githubusercontent.com/115881685/215601234-51d90fb6-7a68-4694-82e4-685522b55f7d.png)
The file requirements.txt list the external library flask, and in this case we are specifically using the version 2.1.0


![repo 4](https://user-images.githubusercontent.com/115881685/215602130-f844cc27-61ae-4476-a4b6-e3ce6ffe84ac.png)
The dockerfile dockerizes that python program and creates a container image. It is using the base python 3.8 docker image, and then its copying over the requirement file running a pip install of the flask. There it is running the python program accepting incoming connection.



```
node {
    def app

    stage('Clone repository') {
      

        checkout scm
    }

    stage('Build image') {
  
       app = docker.build("georgenal/test")
    }

    stage('Test image') {
  

        app.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        
        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
            app.push("${env.BUILD_NUMBER}")
        }
    }
    
    stage('Trigger ManifestUpdate') {
                echo "triggering updatemanifestjob"
                build job: 'updatemanifest', parameters: [string(name: 'DOCKERTAG', value: env.BUILD_NUMBER)]
        }
}
```

This the jenkinsfile which is for the job that is creating the container image.

In the first stage it clones this repository into the jenkins enviroment and then it builds the container image.

The next stage is a dummy place holder.

In the next stage is where i push the image to dockerhub.

And in the last stage, we trigger another jenkins job to update the deployment file, and the name of the this job is *updatemanifest* 



![repo 5](https://user-images.githubusercontent.com/115881685/215608198-fa93e034-8d2f-4d77-94dc-469ae813a54b.png)
This is the kubernetesmanifest repository which contains a jenkinsfile and a deploymentfile for the jenkins job to update the deployment.



![repo 7](https://user-images.githubusercontent.com/115881685/215611245-8e9c0a0b-a9ab-4cff-a76c-991db0a8b607.png)
![repo 6(1)](https://user-images.githubusercontent.com/115881685/215611039-54c76f75-771e-45a4-9013-fda83771aec3.png)
If we go to the deployment.yaml, the container image is referencing to the latest tag.

And in the next step, we are creating a loadbalancer service to talk to the container.



```
node {
    def app

    stage('Clone repository') {
      

        checkout scm
    }

    stage('Update GIT') {
            script {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        //def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                        sh "git config user.email onalotech7@gmail.com"
                        sh "git config user.name georgeonalo"
                        //sh "git switch main"
                        sh "cat deployment.yaml"
                        sh "sed -i 's+georgenal/test.*+georgenal/test:${DOCKERTAG}+g' deployment.yaml"
                        sh "cat deployment.yaml"
                        sh "git add ."
                        sh "git commit -m 'Done by Jenkins Job changemanifest: ${env.BUILD_NUMBER}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/kubernetesmanifest.git HEAD:main"
      }
    }
  }
}
}
```

This is the jenkinsfile for updating the deployment file.

The first step is similar, it clones this repository in the jenkins enviroment, and in the second stage, it updates the file.


### Jenkins Installation
For detailed explanation on how to install jenkins on ec2 (not on your local machine because we are going to need webhook) see the aws official [documentation](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/) page.

![5](https://user-images.githubusercontent.com/115881685/215617093-ab07ac8e-f62a-483a-947c-17f0d189c805.png)
*My ec2 machine*


![6](https://user-images.githubusercontent.com/115881685/215617278-c13bc81d-0584-4176-b3f2-e386fe3489df.png)
*Sshed into the machine*


![7](https://user-images.githubusercontent.com/115881685/215617634-7e83b4df-73b7-428d-9fa3-fd3354cab749.png)
*installed jenkins*

![8](https://user-images.githubusercontent.com/115881685/215617848-6c7ffe47-74b9-40d4-9db2-c6745cf632ac.png)
*accessed jenkins url*

#### Next step setup credentials for dockerhub and github

To do this, on the jenkins home page, click *manage jenkin, manage credntials, under Global click jenkins, and then add credentials*
scroll down to the id field, label it "dockerhub".

add another credentials again, scroll down to the id field and label it "github".

Pls note the username here respectively is the username for your github and dockerhub accounts not the email id used for login.

And also, the password is not the login password, instead it should be a personal access token that is generated respectively from both github and dockerhub accounts.

![9](https://user-images.githubusercontent.com/115881685/215621053-3314b92c-89a6-481d-ac61-22cdc4cc4666.png)



### Jenkins Jobs Setup


To create a jenkins job, *click new item*, and enter the the name "buildimage", *select pipeline* and click "ok"

![new](https://user-images.githubusercontent.com/115881685/215684894-18a6ed8f-a313-4c81-9b0f-6a2cc8d895bc.png)


Scroll down to pipeline, select pipeline script from SCM, and then Git under SCM

Go back to kubernetescode repo, click code and copy the http url and then paste it in the repository url.

Under branch specifier, change it to *main*

and finally click *save*

![code](https://user-images.githubusercontent.com/115881685/215686480-2774f3fe-629c-4f2e-adcc-5c8ef943e32b.png)
![image](https://user-images.githubusercontent.com/115881685/215686957-b0bf5014-f42a-45b4-9f3a-546b7c578aef.png)
![image](https://user-images.githubusercontent.com/115881685/215687471-bf10851e-db0d-499f-b1c3-7c468bd0ba85.png)


#### Next we setup the jenkins manifestupdate job.

like before, *click new item*, and enter the the name "updatemanifest", *select pipeline* and click "ok"
![image](https://user-images.githubusercontent.com/115881685/215690369-b9871562-533b-4b3b-aa0a-5feeb8f01c6f.png)


Select *this project is parametalized* and then add a *string parameter*. Name of the parameter is *DOCKERTAG*

Default value is latest but it will be overide from the jenkins job.
![image](https://user-images.githubusercontent.com/115881685/215690717-192f40f3-5512-434b-84d4-9d0a8bba027b.png)


Again like before, Scroll down to pipeline, select pipeline script from SCM, and then Git under SCM

Go back to kubernetesmanifest repo, click code and copy the http url and then paste it in the jenkins repository url.

Under branch specifier, change it to *main*

and click *save*
![code 1](https://user-images.githubusercontent.com/115881685/215690974-1aeb8b34-4841-4ea2-8903-1b76bc9c7a19.png)
![image](https://user-images.githubusercontent.com/115881685/215691219-20fc52d7-6fac-4318-b0a5-83885d87bb83.png)

Now lets try to manually run the jobs that we have just created, go to the dashboard and select the "buildimage" job
![image](https://user-images.githubusercontent.com/115881685/215691732-651b3ccf-d84b-4067-9c5b-8a13433cfae3.png)

Click "build now"

![image](https://user-images.githubusercontent.com/115881685/215692104-83ad7d75-93ce-494a-9c91-c2b80f6a730a.png)


Our job gets built and this automatically triggers the updatemanifest job

![image](https://user-images.githubusercontent.com/115881685/215692895-a5930d0e-2745-4606-a66b-5ee396cf8607.png)


Going to our dockerhub we see that our very first image is now in the repository.

![image](https://user-images.githubusercontent.com/115881685/215693532-04433b44-8a5c-4531-b2ee-cc2b0a35b8cc.png)


### ArgoCD (GitOps) Installation












Click this link for instructions on how to impliment this project: [Tutorial](https://youtu.be/o4QG_kqYvHk)



