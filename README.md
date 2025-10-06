# Insured Assurance CI/CD Pipeline Project

## Project Overview

This repository implements a fully automated CI/CD pipeline for a Java web application using GitHub Actions for Continuous Integration and Jenkins for Continuous Deployment to an Apache Tomcat server. The pipeline is designed to be reusable and follows best practices suitable for cloud environments such as AWS.

---

## Architecture

- Code pushed to GitHub triggers GitHub Actions.
- GitHub Actions build the application (WAR) using Maven and upload the artifact to an AWS S3 bucket.
- Jenkins server running on an AWS EC2 instance fetches the artifact from S3.
- Jenkins deploys the WAR file to the Tomcat server hosted on another EC2 instance.
- The application is hosted live on Tomcat, accessible via its public IP and configured port.

---

## AWS Infrastructure

- Two EC2 instances:
  - Jenkins Server (with Jenkins installed and configured)
  - Tomcat Server (hosting the Java web application)
- Private S3 bucket for storing WAR artifacts
- Proper AWS IAM roles and policies configured to allow secure access

---

## Prerequisites

- AWS Free Tier account with access to EC2, S3, IAM
- Two EC2 instances launched and accessible:
  - Jenkins Server (Java, Jenkins installed)
  - Tomcat Server (Java, Tomcat installed and running)
- AWS CLI installed on GitHub Actions runner
- SSH keys for Jenkins → Tomcat communication
- Jenkins configured with required plugins (GitHub, Maven, Publish Over SSH)
- GitHub repository with necessary secrets set

---

## Setup Instructions

### 1. AWS Setup

- Create an S3 bucket to store WAR files.
- Launch Jenkins and Tomcat EC2 instances with proper security groups.
- Configure Security Groups to allow SSH, Jenkins (8080), and Tomcat (8080) access as per needs.

### 2. Jenkins Setup

- Install Jenkins on EC2 following documented commands.
- Add credentials and SSH keys for deployment.
- Create Jenkins Freestyle job `TomcatDeployment`:
  - Parameterized with `PRESIGNEDURL` (String)
  - Build step to download artifact and deploy to Tomcat via SSH.

### 3. Tomcat Setup

- Install Java and Tomcat.
- Configure Tomcat to run as a systemd service.
- Set up deployment user and SSH key authentication from Jenkins server.

### 4. GitHub Repository

- Add repository secrets:
  - `AWSACCESSKEYID`
  - `AWSSECRETACCESSKEY`
  - `AWSREGION`
  - `S3BUCKET`
  - `JENKINSURL`
  - `JENKINSUSER`
  - `JENKINSTOKEN`
- Include `.github/workflows/maven.yml` to automate build, S3 upload, and Jenkins trigger.

---

## GitHub Actions Workflow Example

```
name: CI - build war - upload to S3 - trigger Jenkins
on:
  push:
    branches: [main]

env:
  WARNAME: insured-assurance.war

jobs:
  build-and-trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Java 11
        uses: actions/setup-java@v3
        with:
          distribution: temurin
          java-version: 11

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Build with Maven
        run: mvn -B clean package

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWSACCESSKEYID }}
          aws-secret-access-key: ${{ secrets.AWSSECRETACCESSKEY }}
          aws-region: ${{ secrets.AWSREGION }}

      - name: Upload WAR to S3
        run: |
          ARTIFACTKEY=releases/${{ github.sha }}-${{ env.WARNAME }}
          aws s3 cp target/${{ env.WARNAME }} s3://${{ secrets.S3BUCKET }}/${ARTIFACTKEY}
          echo "s3://${{ secrets.S3BUCKET }}/${ARTIFACTKEY}" > artifactpath.txt

      - name: Create presigned URL (1 hour)
        id: presign
        run: |
          ARTIFACTPATH=$(cat artifactpath.txt)
          PRESIGNEDURL=$(aws s3 presign ${ARTIFACTPATH} --expires-in 3600)
          echo "url=${PRESIGNEDURL}" >> $GITHUB_OUTPUT

      - name: Trigger Jenkins job
        uses: appleboy/jenkins-action@v0.3.0
        with:
          url: ${{ secrets.JENKINSURL }}
          user: ${{ secrets.JENKINSUSER }}
          token: ${{ secrets.JENKINSTOKEN }}
          job: TomcatDeployment
          parameters: |
            PRESIGNEDURL=${{ steps.presign.outputs.url }}
```

---

## Jenkins Job Configuration

- Freestyle job with parameter `PRESIGNEDURL`.
- Build step shell script example:

```
#!/bin/bash
set -euo pipefail

echo "Downloading WAR from presigned URL..."
wget -O $WORKSPACE/app.war "$PRESIGNEDURL"

echo "Copying WAR to Tomcat server..."
scp -i ~/.ssh/tomcat_id_rsa $WORKSPACE/app.war tomcat@TOMCAT_PUBLIC_IP:/opt/tomcat/webapps/insured-assurance.war

echo "Restarting Tomcat service..."
ssh -i ~/.ssh/tomcat_id_rsa tomcat@TOMCAT_PUBLIC_IP "sudo systemctl restart tomcat"

echo "Deployment successful."
```

---

## Validation

- Push changes to `main` branch.
- Monitor GitHub Actions for successful build and artifact upload.
- Verify Jenkins triggers and deploys artifact.
- Confirm web app access on Tomcat server at `http://TOMCAT_PUBLIC_IP:8080/insured-assurance`.

---

## Troubleshooting

- Ensure Maven builds succeed locally.
- Verify SSH keys and permissions between Jenkins and Tomcat.
- Check AWS credentials in GitHub secrets.
- Review Jenkins logs and Tomcat logs for errors.
- Confirm security groups permit necessary traffic.

---

## License

This project is open-sourced under the MIT License.

---

## Contact

For questions, reach out on the repository issues page.

---

Replace `TOMCAT_PUBLIC_IP` and other placeholders with your actual environment details.

Happy CI/CD automation!