# CI/CD System for the Petclinic Springboot application

This documentation provides an overview of the CI/CD (Continuous Integration/Continuous Deployment) system I implemented for the Springboot Petclinic project. The system follows a specific workflow to build, test, and deploy the application.

## 1. How It Works (overview)

<DIV align="centre"><img src="resources/0_CICD_System.png" width="1024"></DIV><br/>

The CI/CD system operates as follows:

0. I forked the Springboot Petclinic project from the original GitHub repository to my own repository. The original repository link is: [https://github.com/spring-projects/spring-petclinic](https://github.com/spring-projects/spring-petclinic).

1. We'll use Jenkins as the CI/CD tool. I created multiple Freestyle Jobs in Jenkins to add granularity and better control over the pipeline, once a job is done, it triggers the logically following jobs. Of course it could've been possible to write one regular Jenkins pipeline and separate these tasks into several stages.
Jenkins will checkout the project and perform the following steps then.

2. Run a Snyk security scan (Snyk Open Source) to identify and address any vulnerabilities and generate a reporting based on the findings.
In this step we leverage the JFrog REST API to create a local repo in Artifactory for the `pom.xml` file. Naming conventions are essential, I used the `local/remote-technology-project` schema.

3. Build the application using Maven.
In this step we also leverage the JFrog REST API to create a local repo in Artifactory for the `.jar` file(s)

4. Build a Docker image and push it to Artifactory (like before: before pushing, a separate local repository is created in Artifactory using the JFrog REST API.) and DockerHub

This workflow ensures that the project is thoroughly tested, built, and stored in Artifactory and DockerHub for deployment.

# 2. Prerequisites

Before setting up the CI/CD system, ensure that the following prerequisites are met:

- The system can be implemented on any host. For this documentation, I implemented on `localhost`. However, an optional Terraform script is provided to spin up an EC2 instance on AWS and install Jenkins there.

- If you're a Mac user like me, I recommend using Homebrew for installing the required tools. You can download Homebrew from [https://brew.sh](https://brew.sh).

- Install the following tools using Homebrew:
  - JFrog CLI (`jfrog`)
  - Jenkins LTS (`jenkins-lts`)
  - Snyk (`snyk`)
  - Rosetta (required for Snyk)

- Install Docker Desktop by downloading it from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop).

- Install Java (OpenJDK). For example, you can use OpenJDK version 20.0.1, as follows (Make sure the installed Java version is compatible with the Springboot Petclinic project): 
```bash 
$ brew install openjdk@20.0.1
```
- I wrote several bash scripts for the pipelines, a good IDE of your choice is also recommended to highlight the syntax.

:rotating_light: IMPORTANT!
- You can use the following commands to start, restart and stop Jenkins:
```bash
brew services stop jenkins-lts
brew services start jenkins-lts
brew services restart jenkins-lts  
```

- By default, Docker is not going to be available from Jenkins is you installed it with Homebrew, make sure you add it the `.plist` file.
``` /opt/homebrew/Cellar/jenkins-lts/<version>/homebrew.mxcl.jenkins-lts.plist```

Then restart Jenkins:
```bash
brew services restart jenkins-lts
```

<details>
<summary>:hammer_and_wrench: You can use this code below to extend jenkins-lts.plist </summary><br/>
  
```bash
<key>EnvironmentVariables</key>
    <dict>
      <key>PATH</key>
      <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Applications/Docker.app/Contents/Resources/bin/:/Users/<yourUsername>/Library/Group\ Containers/group.com.docker/Applications/Docker.app/Contents/Resources/bin</string>
    </dict>
  </dict>
```
</details>

# 3. What do the Jenkins jobs do?

As mentioned above, instead of creating a tremendous pipeline that consist of many stages, I structured the jobs into multiple freestyle pipeline jobs that trigger each other.
The workflow:
<DIV align="centre"><img src="resources/1_Jenkins_workflow.png" width="1024"></DIV><br/>

<details>
<summary>:hammer_and_wrench: Bash script to check if a local repo exists in Artifactory and if not it invokes the JFrog REST API & push pom.xml to Artifactory </summary><br/>
  
```bash
#!/bin/bash

artifactoryUrl='https://artre.jfrog.io/artifactory'
localRepoName='local-maven-petclinic-pom'
pomFilePath="${WORKSPACE}/pom.xml"

createLocalRepoJson=$(cat << EOF
{
  "key": "${localRepoName}",
  "rclass": "local",
  "packageType": "maven",
  "xrayIndex": true
}
EOF
)

credentialsId='artifactory-credentials'

# Retrieve Artifactory username and password from Jenkins credentials
read -r ARTIFACTORY_USERNAME ARTIFACTORY_PASSWORD <<< "$(curl -s "http://localhost:8080/credentials/store/system/domain/_/credential/${credentialsId}/api/json" | sed -n -E 's/.*"username" : "([^"]+)".*/\1/p' && curl -s "http://localhost:8080/credentials/store/system/domain/_/credential/${credentialsId}/api/json" | sed -n -E 's/.*"password" : "([^"]+)".*/\1/p')"

# Check if the local repository already exists in Artifactory
checkLocalRepoExistence() {
  local response=$(curl -s -o /dev/null -w '%{http_code}' -u "${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}" -X GET "${artifactoryUrl}/api/repositories/${localRepoName}")
  
  if [[ $response == "200" ]]; then
    return 0
  else
    return 1
  fi
}

# Create Artifactory local repository if it doesn't exist
if checkLocalRepoExistence; then
  echo "Local repository '${localRepoName}' already exists."
else
  # Create the local repository
  curl -u "${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}" -X PUT "${artifactoryUrl}/api/repositories/${localRepoName}" -H 'Content-Type: application/json' -d "${createLocalRepoJson}"
  echo "Local repository '${localRepoName}' created successfully."
fi

# Upload pom.xml to the local repository
curl -u "${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}" -X PUT "${artifactoryUrl}/${localRepoName}/pom.xml" --upload-file "${pomFilePath}"

echo "pom.xml uploaded successfully to '${localRepoName}' repository."

```
</details>

<details>
<summary>:hammer_and_wrench: Bash script build the app, login and push the image to DockerHub</summary><br/>
  
```bash
#!/bin/bash

# Change to the workspace directory
cd ${WORKSPACE}

# Retrieve DockerHub credentials from Jenkins credentials
JSON_RESPONSE=$(curl -sS "http://localhost:8080/credentials/store/system/domain/_/credential/docker-credentials/api/json")

DOCKERHUB_USERNAME=$(echo "$JSON_RESPONSE" | grep -oE '"username"\s*:\s*"[^"]+"' | cut -d '"' -f 4)
DOCKERHUB_PASSWORD=$(echo "$JSON_RESPONSE" | grep -oE '"password"\s*:\s*{"\w+"\s*:\s*"([^"]+)"' | cut -d '"' -f 4)

# Log in to DockerHub
echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin

# Run the Maven wrapper command to build the Docker image
./mvnw spring-boot:build-image

# Tag the built image with DockerHub repository name
DOCKERHUB_REPO="artredist/petclinic"
docker tag docker.io/library/spring-petclinic:3.1.0-SNAPSHOT "$DOCKERHUB_REPO:3.1.0-SNAPSHOT"

# Push the tagged image to DockerHub
docker push "$DOCKERHUB_REPO:3.1.0-SNAPSHOT"

```
</details>

<details>
<summary>:hammer_and_wrench: Bash script to check if a local repo exists in Artifactory and if not it invokes the JFrog REST API & push .jar to Artifactory </summary><br/>
  
```bash
#!/bin/bash

artifactoryUrl='https://artre.jfrog.io/artifactory'
localRepoName='local-maven-petclinic-jar'
jarFilesPath="${WORKSPACE}"

createLocalRepoJson=$(cat << EOF
{
  "key": "${localRepoName}",
  "rclass": "local",
  "packageType": "maven",
  "xrayIndex": true
}
EOF
)

credentialsId='artifactory-credentials'

# Retrieve Artifactory username and password from Jenkins credentials
response=$(curl -s "http://localhost:8080/credentials/store/system/domain/_/credential/${credentialsId}/api/json")
ARTIFACTORY_USERNAME=$(echo "$response" | grep -oE '"username" : "([^"]+)"' | sed -E 's/"username" : "([^"]+)"/\1/')
ARTIFACTORY_PASSWORD=$(echo "$response" | grep -oE '"password" : "([^"]+)"' | sed -E 's/"password" : "([^"]+)"/\1/')

# Check if the local repository already exists in Artifactory
checkLocalRepoExistence() {
  local response=$(curl -s -o /dev/null -w '%{http_code}' -u "${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}" -X GET "${artifactoryUrl}/api/repositories/${localRepoName}")

  if [[ $response == "200" ]]; then
    return 0
  else
    return 1
  fi
}

# Create Artifactory local repository if it doesn't exist
if checkLocalRepoExistence; then
  echo "Local repository '${localRepoName}' already exists."
else
  # Create the local repository
  curl -u "${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}" -X PUT "${artifactoryUrl}/api/repositories/${localRepoName}" -H 'Content-Type: application/json' -d "${createLocalRepoJson}"
  echo "Local repository '${localRepoName}' created successfully."
fi

# Upload JAR files to the local repository
find "$jarFilesPath" -name "*.jar" -type f -print0 | while IFS= read -r -d $'\0' jarFile; do
  jarFileName=$(basename "$jarFile")
  curl -u "${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}" -X PUT "${artifactoryUrl}/${localRepoName}/${jarFileName}" --upload-file "${jarFile}"
done

echo "The JAR files were uploaded successfully to the '${localRepoName}' repository."

```
</details>
  
