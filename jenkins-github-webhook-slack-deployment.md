# Jenkins GitHub Webhook + Automated Deployment + Slack

This guide configures a Jenkins pipeline that automatically deploys a static website whenever changes are pushed to GitHub.

The workflow is:

```text
GitHub
   |
   | git push
   v
GitHub Webhook
   |
   v
Cloudflare Tunnel
   |
   v
Jenkins Controller
   |
   +--> Checkout
   +--> Get Server IP
   +--> Deploy Website
   +--> Verify Deployment
   |
   v
Nginx
   |
   v
Website
   |
   v
Slack Notification
```

---

## 1. Install Nginx

Install Nginx on the Jenkins controller:

```bash
sudo apt update
sudo apt install nginx -y
```

Check the service:

```bash
sudo systemctl status nginx
```

Test Nginx locally:

```bash
curl http://localhost
```

---

## 2. Give Jenkins Access to the Nginx Web Directory

When the pipeline runs on the built-in/controller node, its shell steps run as the `jenkins` Linux user.

For this controller-only lab, give Jenkins ownership of the web directory:

```bash
sudo chown -R jenkins:jenkins /var/www/html
sudo chmod -R 755 /var/www/html
```

Verify:

```bash
ls -ld /var/www/html
ls -l /var/www/html
```

The directory should show `jenkins jenkins` ownership.

---

## 3. Prepare the GitHub Repository

Create or use a GitHub repository containing the static website.

The repository should contain at least:

```text
index.html
Jenkinsfile
```

Use the `main` branch for the pipeline.

---

## 4. Create a Jenkins Pipeline Job

In Jenkins:

1. Select **New Item**.
2. Enter a job name.
3. Select **Pipeline**.
4. Create the job.

Under the Pipeline configuration, select:

```text
Pipeline script from SCM
```

Select:

```text
SCM: Git
```

---

## 5. Create a GitHub Fine-grained Personal Access Token

Create a **Fine-grained Personal Access Token (PAT)** in GitHub.

Go to:

```text
GitHub
→ Settings
→ Developer settings
→ Personal access tokens
→ Fine-grained tokens
```

Create a new token.

### Select Repository Access

Choose:

```text
Resource owner: <your GitHub account>
Repository access: Only select repositories
Repository: jenkins-webhook-demo
```

Restricting the token to a specific repository limits where Jenkins can use the token.

### Set Repository Permissions

Grant only the permissions required for this lab:

```text
Contents → Read-only
Metadata → Read-only
```

### Fine-grained vs Classic PAT

GitHub provides **Fine-grained** and **Classic** personal access tokens.

| Feature | Fine-grained PAT | Classic PAT |
|---|---|---|
| Repository access | Specific repositories | Broader scope |
| Permissions | Individual repository permissions | Predefined scopes |
| Access control | More granular | Less granular |
| Least privilege | Easier to implement | Usually broader |
| Example | Read-only access to one repository | `repo` scope |

For this Jenkins lab, use a Fine-grained PAT restricted to the `jenkins-webhook-demo` repository with read-only access.

### Why Contents → Read-only?

Jenkins needs to read the repository so it can retrieve files such as:

```text
index.html
Jenkinsfile
```

Jenkins does not need permission to modify the repository contents for this pipeline.

### Why Metadata → Read-only?

Repository metadata access allows GitHub repository information to be read during repository operations. GitHub requires this permission for repository access.

Create the token and copy it securely. Do not commit the token to Git.

---

## 6. Add the GitHub PAT to Jenkins Credentials

Go to:

```text
Manage Jenkins
→ Credentials
```

Add a credential.

Use:

```text
Kind: Username with password
Username: <GitHub username>
Password: <GitHub PAT>
ID: github-pat
Description: GitHub PAT for Jenkins
```

Save the credential.

---

## 7. Configure GitHub SCM

In the Jenkins Pipeline job, configure the Git repository.

Set:

```text
Repository URL:
https://github.com/<username>/<repository>.git
```

Select the GitHub credential created earlier.

Set the branch:

```text
main
```

Set the Jenkinsfile path:

```text
Jenkinsfile
```

Save the configuration.

---

## 8. Enable the GitHub Webhook Trigger

In the Jenkins job configuration, enable:

```text
GitHub hook trigger for GITScm polling
```

This allows a GitHub webhook event to trigger the Jenkins job.

---

## 9. Create the GitHub Webhook

Open the GitHub repository:

```text
Settings
→ Webhooks
→ Add webhook
```

Set the Payload URL to:

```text
https://<your-jenkins-domain>/github-webhook/
```

Example:

```text
https://jenkinslab.jenkinsdevops.space/github-webhook/
```

Set:

```text
Content type: application/x-www-form-urlencoded
```

For this lab, leave the secret empty.

Select:

```text
Just the push event
```

Enable:

```text
Active
```

Save the webhook.

---

## 10. Test the GitHub Webhook

Make a change to `index.html`.

Commit and push the change:

```bash
git add index.html
git commit -m "Update website"
git push origin main
```

The expected flow is:

```text
git push
   ↓
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Pipeline
```

---

## 11. Configure the Jenkins Controller Label

If the pipeline uses:

```groovy
agent {
    label 'control-built-in'
}
```

make sure the built-in/controller node has the same label.

Go to:

```text
Manage Jenkins
→ Nodes
→ Built-In Node
→ Configure
```

Add:

```text
control-built-in
```

Save the configuration.

---

## 12. Install Pipeline Stage View

Install the **Pipeline Stage View** plugin to visualize the pipeline stages.

Go to:

```text
Manage Jenkins
→ Plugins
```

Search for:

```text
Pipeline Stage View
```

Install the plugin.

---

## 13. Configure the Jenkinsfile

Use a declarative pipeline structure such as:

```groovy
pipeline {
    agent {
        label 'control-built-in'
    }

    options {
        skipDefaultCheckout(true)
    }

    stages {
        ...
    }
}
```

---

## 14. Control the Git Checkout

Use:

```groovy
options {
    skipDefaultCheckout(true)
}
```

This disables the default automatic SCM checkout.

Add an explicit Checkout stage:

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

This lets the pipeline control the checkout explicitly.

### What happens without skipDefaultCheckout?

If `skipDefaultCheckout(true)` is removed while `checkout scm` remains, Jenkins performs the automatic checkout and the explicit `checkout scm` stage performs another checkout.

So the repository can be checked out twice.

---

## 15. Get the Server IP

Determine the IP address of the node running the pipeline:

```groovy
stage('Get Server IP') {
    steps {
        script {
            env.SERVER_IP = sh(
                script: "hostname -I | awk '{print \$1}'",
                returnStdout: true
            ).trim()

            echo "Server IP: ${env.SERVER_IP}"
        }
    }
}
```

---

## 16. Deploy the Website

Deploy the checked-out website to the Nginx document root:

```groovy
stage('Deploy Website') {
    steps {
        sh '''
            echo "Deploying website..."
            echo "Running as user: $(whoami)"
            echo "Hostname: $(hostname)"

            rm -rf /var/www/html/*
            cp index.html /var/www/html/

            echo "Website deployed successfully"
        '''
    }
}
```

The deployment flow is:

```text
GitHub Repository
       |
       v
Jenkins Workspace
       |
       | copy
       v
/var/www/html
       |
       v
Nginx
       |
       v
Website
```

---

## 17. Verify the Website Deployment

Verify the deployed website with `curl`:

```groovy
stage('Verify Deployment') {
    steps {
        sh '''
            echo "Checking website..."

            curl -f http://localhost

            echo ""
            echo "Website is working!"
        '''
    }
}
```

The `-f` option makes `curl` return a failure status for HTTP errors.

---

## 18. Add Pipeline Post Actions

Add success and failure handling:

```groovy
post {
    success {
        echo 'Website deployment successful!'
    }

    failure {
        echo 'Website deployment failed!'
    }
}
```

---

# Slack Integration

## 19. Create a Slack App

Create a Slack app for Jenkins notifications.

Go to the Slack API and select:

```text
Create New App
→ From scratch
```

Give the app a Jenkins-related name.

Install the app in the Slack workspace.

---

## 20. Add the Slack Bot Permission

Open:

```text
OAuth & Permissions
→ Scopes
→ Bot Token Scopes
```

Add:

```text
chat:write
```

This allows the Jenkins bot to send messages to Slack.

---

## 21. Generate the Slack Bot Token

Install/authorize the Slack app in the workspace.

Copy the Bot User OAuth Token.

It will look similar to:

```text
xoxb-...
```

Keep the token secret.

---

## 22. Add the Slack Token to Jenkins Credentials

Go to:

```text
Manage Jenkins
→ Credentials
```

Create a credential:

```text
Kind: Secret text
Secret: <Slack Bot Token>
ID: slack-token
Description: Slack Bot Token
```

Save it.

---

## 23. Install the Jenkins Slack Notification Plugin

Go to:

```text
Manage Jenkins
→ Plugins
```

Search for:

```text
Slack Notification
```

Install the plugin.

---

## 24. Configure Slack in Jenkins

Go to:

```text
Manage Jenkins
→ System
```

Configure the Slack integration.

Use the Slack workspace and the Slack credential created earlier.

For the custom Slack app, enable the custom bot/app setting required by the plugin.

Run **Test Connection** and confirm that the connection succeeds.

---

## 25. Create the Slack Notification Channel

Create a channel for Jenkins notifications.

Example:

```text
#jenkins-builds
```

Add the Jenkins bot to the channel.

The bot must be a member of the channel before it can send messages there.

---

## 26. Configure Slack Notifications in the Jenkinsfile

Use the Jenkins Slack pipeline step with the stored credential.

Example:

```groovy
slackSend(
    channel: '#jenkins-builds',
    tokenCredentialId: 'slack-token',
    botUser: true,
    message: "Build ${env.BUILD_NUMBER} completed."
)
```

Add notifications to the appropriate `post` sections.

Example:

```groovy
post {
    success {
        slackSend(
            channel: '#jenkins-builds',
            tokenCredentialId: 'slack-token',
            botUser: true,
            message: "Build ${env.BUILD_NUMBER} succeeded."
        )
    }

    failure {
        slackSend(
            channel: '#jenkins-builds',
            tokenCredentialId: 'slack-token',
            botUser: true,
            message: "Build ${env.BUILD_NUMBER} failed."
        )
    }
}
```

---

## 27. Test a Successful Build

Push a valid change:

```bash
git add .
git commit -m "Update Jenkins pipeline"
git push origin main
```

Verify the complete flow:

```text
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Pipeline
   ↓
Website
   ↓
Slack
```

Confirm that Slack receives the successful build notification.

---

## 28. Test a Failed Build

Introduce a controlled error in the pipeline and push the change:

```bash
git add .
git commit -m "Test pipeline failure"
git push origin main
```

Verify that:

- Jenkins reports the build as failed.
- Jenkins logs show the failure.
- Slack receives the failure notification.

Restore the working Jenkinsfile after the test.

---

## 29. Verify the End-to-End Deployment

The completed controller-based deployment is:

```text
                    GitHub
                       |
                    git push
                       |
                       v
                GitHub Webhook
                       |
                       v
              Cloudflare Tunnel
                       |
                       v
              Jenkins Controller
                       |
                       v
                 Jenkins Pipeline
                       |
          +------------+------------+
          |            |            |
          v            v            v
       Checkout    Get Server IP   Deploy
                                    |
                                    v
                              /var/www/html
                                    |
                                    v
                                  Nginx
                                    |
                                    v
                                 Website
                                    |
                                    v
                                  Slack
```
