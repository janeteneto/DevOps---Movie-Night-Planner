# DevOps - Movie Night Planner Project

**This repo is a detailed documentation of processes used in the DevOps lifecycle of the Movie Night Planner Project. This document includes the following contents:**
- Project Description
- DevOps Objectives
- CI/CD Pipeline

  ## Project Description

  The Movie Night Project had as its primary goal to provide a website that allows users to plan their selected movies' streaming time. The website extracts data from the TMdb database service and the JustWatch database that stores multiple data points for each title (movie/TV show). The different API Routes help to make requests to receive the data as efficiently as possible. This is how the home page of the website looks like:

  ![2023-11-07](https://github.com/janeteneto/DevOps---Movie-Night-Planner/assets/129942042/4d49a979-330a-4f84-8fcd-33c228f4666b)

The website has a few core functionalities:
- The user can register an account
- The user can log in and logout
- The user can 'Add to plan'
- The user can select a date to add a movie to its plan

## DevOps Objectives

For this project, I was the DevOps consultant for two different teams, and the objectives were:
- Enable fast and automated testing of code
- Enable fast and automated deployment of code
- Manage Cloud infrastructure **(AWS)**
- Manage Gitlab Pipeline
- Manage Version Control
- Manage production and staging environments

## CI/CD Pipeline

To achieve the objectives above, I used different tools, which were all necessary to create a Continuous Integration and Continuous Deployment pipeline. For the pipeline itself, I used **Gitlab**. I chose this tool because of the friendly user interface, familiarity with the tool, and easy integration with the project's central repository, located in **GitHub**.
For the cloud service, I selected **AWS** for the same reasons, specifically, the **EC2 Service**. These were the steps to create an efficient system and pipeline from start to finish:

1. Cloned the Github repo to Gitlab
2. Enabled the 'Mirroring' feature in Gitlab, which enabled the automatic syncing of the repo. This way, each time there are changes in Github, Gitlab pulls and mirrors these changes.
3. In AWS, created a secure VPC and subnet
4. Create an EC2 instance with that VPC and with the following configurations:
- t2.micro
- AWS Linux AMI
- Command to install and enable Java in the user data

5. Attach Elastic IP address to the instance
**6. Build Gitlab pipeline:**

1. Define stages:

````
image: maven:3.8.3-openjdk-17

stages:
    - build
    - package
    - test
    - deploy
````
- The image contains maven and java dependencies needed to run the app
- `build` - installs and packages the app into a `.jar` file snapshot
- `package` - packages the app while skipping tests, in case of previous job failure
- `test` - verifies the code and runs JUnit tests
- `deploy` - deploys code into the ec2 instance

2. Include SAST tool:
````
include:
- template: Security/SAST.gitlab-ci.yml
````
- This line of code includes the Gitlab template to run SAST tests, ensuring security and finding vulnerabilities in the code using the `semgrep` command.

3. `Build` job:

````
build:
    stage: build
    environment: staging
    tags: 
        - docker
    script:
        - echo "Maven build started"
        - "mvn clean package"
    artifacts:
        paths:
            - target/*.jar
    only:
        - dev
    allow_failure: true
````

4. `Package` job:

````
package:
    stage: package
    tags:
        - docker
    script:
        - echo "Maven packaging started (skipping tests)"
        - "mvn clean package -DskipTests"
    artifacts:
        paths:
          - target/*.jar
    rules:
      - when: on_failure
    dependencies:
        - build
````

5. `Test` job:

````
test:
    stage: test
    script:
        - echo "Maven test started"
        - "mvn verify"
````

6. `Deploy` job:

````
deploy staging:
    stage: deploy
    needs: [build]
    script:
        - echo "Maven deploy to staging started"
        - chmod 600 $SSH_PRIVATE_KEY
        - scp -i $SSH_PRIVATE_KEY -o StrictHostKeyChecking=no $JAR_FILE $IP_ADDRESS
        - ssh -i $SSH_PRIVATE_KEY -o StrictHostKeyChecking=no $IP_ADDRESS "fuser -k 8080/tcp || true; java -jar /home/ec2-user/MovieNightPlanner-0.0.1-SNAPSHOT.jar > /home/ec2-user/application.log 2>&1 &"
    only:
        - dev
````
