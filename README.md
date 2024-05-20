---
title: Gitness Workshop
authors: Dewan Ahmed & Jim Sheldon
version: 1.2
---

# Gitness Workshop

## Prerequisites

- [Docker installation](https://docs.docker.com/engine/install/) (tested with Docker Desktop 4.26.1 and docker CLI 24.0.7)
- [k3d](https://k3d.io/) installed (tested with 5.6.3)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) installed (tested with 1.28.2)
- (Optional) A slack workspace with admin access to create a webhook for notifications.

## Get Started

### Create a Docker Network

Create a common network for Gitness and the local Kubernetes cluster. This allows Gitness pipeline containers to communicate with the cluster.

```shell
docker network create gitness
docker network ls
```

### Create a local Docker registry

Create a k3d Docker registry to contain the built Docker image for deployment.

```shell
k3d registry create registry.localhost --port 5000
```

### Start Gitness

1. Use the following Docker command to run Gitness (observe the network this container uses):

```shell
docker run -d \
  -e GITNESS_PRINCIPAL_ADMIN_EMAIL=admin@example.com \
  -e GITNESS_PRINCIPAL_ADMIN_PASSWORD=adminpass1 \
  -e GITNESS_USER_SIGNUP_ENABLED=true \
  -e GITNESS_CI_CONTAINER_NETWORKS=gitness \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/gitness:/data \
  --network=gitness \
  --name gitness \
  --restart always \
  harness/gitness
```
2. Once the container is running, open `localhost:3000` in your browser.
3. Select **Sign Up**.
4. Enter a User ID (`developer`), Email (`developer@example.com`), and Password (`devpass1`).
5. Select **Sign Up**. (You might see a warning to change your password. You can ignore that warning.)
6. Log out from the developer account. Click the profile icon from the bottom-left corner and then click **Log out**. 
7. Log in using the admin User ID (`admin`) and Password (`adminpass1`). 

### Create a new Project

1. Select **New Project**.
2. Enter a project Name (devdays) and optional Description (Gitness is awesome).
3. Select **Create Project**.

Optionally, Gitness can [import projects](https://docs.gitness.com/administration/project-management#import-a-project) from external sources (such as GitLab groups or GitHub organizations).

### Create and configure a local Kubernetes Cluster

Create a k3d cluster, passing the Docker network, a specific API port, and a Docker registry.

```shell
k3d cluster create --network gitness --api-port 9090 --registry-use k3d-registry.localhost:5000 devdays
kubectl cluster-info
```

Create the gitness namespace.

```shell
kubectl create namespace gitness
```

Download the [service account manifest file](service-account.yml).

```shell
curl -sLO https://raw.githubusercontent.com/harness-community/gitness-lab/main/service-account.yml
ls service-account.yml
```

Apply the service account manifest.

```shell
kubectl apply -f service-account.yml
```

Get the service account token.

```shell
kubectl get secret gitness-sa-token -n gitness -o=jsonpath='{.data.token}' | base64 -d && echo
```

Save this token. You’ll use this later.

<details>
  <summary>(Optional) Create a Slack Webhook</summary>

This step is optional and is used to demonstrate how to send notifications from your Gitness pipeline. If you don’t have a Slack workspace set up already, you can skip this step.

<ol>
    <li> Once you have signed in to your Slack workspace, navigate to https://YOUR_SLACK_WORKSPACE_NAME.slack.com/apps and search for <b>Incoming WebHooks</b> under the apps.
    <li> Click <b>Add to Slack</b> and either choose an existing channel or create a new one where the notifications will be sent.
    <li>Click <b>Add Incoming Webhook Integration</b>.
    <li>Copy the Webhook URL under Setup Instructions. You’ll use this later.
</ol>

</details>

## Code Repo

### Create a Repository

1. In your project, select **Repositories**, and then select **New Repository**.
2. Enter a repository **Name** and optional **Description**. Let’s use `gitness-is-awesome` for the repository name.
3. Gitness repositories are initialized with a main branch, unless you specify a different name for the base branch.
4. Select your preference for visibility (**Public** or **Private**). Let’s keep the visibility **Private** for this example. 
5. Optionally, you can add a License, .gitignore, or README file to your repository. Check the box **Add a README file**.
6. Select **Create Repository**.

![gitness-is-awesome repo create](assets/gitness-is-awesome-repo-create.png)

Once the repository is created, click **+ New File**, give the file a name **app.py**, and paste the following:

```python
# app.py

def main():
    print("Gitness is awesome!")

if __name__ == "__main__":
    main()
```

7. Click **Commit Changes** and then **Commit**.

This is how you can create a new repository in Gitness. In the next section, you’ll import a repository from GitHub.

### Import Repository

1. Click on **Repositories**, click on the dropdown and then **Import Repository**.

![Import Repo](assets/import-repo.png)

2. You’ll import a popular Go repository called **podinfo**. Use **harness-community** for Organization and **podinfo** for Repository. Click **Import Repository**. The repository import should happen fast.

![Import Repo Detail](assets/import-repo-detail.png)

### Clone Repository

Next, you’ll clone the podinfo repository using VS Code. 

1. Open VS Code on your local machine and click **Clone Repository**.

![Clone Repo VS Code](assets/clone-repo-vs-code.png)

2. On the Gitness console, click **Clone** and copy the Git clone URL (there’s a copy to clipboard button).

![Git Clone URL](assets/git-clone-url.png)

3. Go back to VS Code and paste the Git clone URL. Select the directory where you want the repository to be copied. This will clone the remote repository in your local machine.

### Webhook

You can send data to HTTP endpoints from actions in your repository, such as opened pull requests, new branches, and more. For this exercise, you’ll use [webhook.site](webhook.site) - a website that offers unique, random URLs to instantly receive and inspect all incoming HTTP requests and webhooks in real-time, facilitating testing and debugging.

1. Navigate to webhook.site and copy your unique URL.

![Webhook Site Unique URL](assets/webhook-site-url.png)

2. On Gitness, click on **Webhooks** under the podinfo repository and then **+ New Webhook**.
3. Give this webhook a name: **trigger_on_pr**.
4. Paste the unique URL you copied under **Payload URL**. You can leave out the **Secret**.
5. Choose **Let me select individual events** and select **PR created**.
6. Click **Create Webhook**.

![Webhook PR created](assets/webhook-pr-created.png)

Continue to the next section to create a PR workflow. Once you raise a PR, you’ll see the trigger in action on this site.

### PR Workflow

1. For this section, let’s add **developer** user to the project. 
2. From the left navigation, click on **Members** and then **+ Add Member**. Find **developer** from the **User** dropdown, choose **Contributor** role, and select **Add member to this project**.

![Add developer to devdays project](assets/add-developer-member.png)

3. On the Gitness portal, log out from this profile by clicking the icon on the bottom-left corner and log back in using the developer credentials. 
4. Back in VS Code, create a new branch named **update-app-version** and commit a change. For instance, this commit updates the `pkg/version/version.go` file, resulting in the application version changing from **6.6.1** to **6.6.2**.

![Update image tag](assets/update-image-tag.png)

5. When you commit and push changes, you’ll be prompted to enter username and password for the remote repository. Click **Generate Clone Credential** under **Git Clone URL** and use the generated credentials.

![Generate Clone Credentials](assets/generate-clone-creds.png)

Once the push is complete, you should see the remote repository reflect the changes.

6. Click on **Pull Requests** from the left navigation menu and then click **+ New Pull Request**. Select your feature branch as the source branch, add a PR description, and click **Create Pull Request**. 

![Create PR](assets/create-pr.png)

7. Click **Add +** under Reviewers and find **Administrator** from the list.

8. Navigate to the webhook.site dashboard and you should see the POST request details triggered by your PR. 

![Webhook triggered PR created](assets/webhook-site-pr-created.png)

> [!NOTE] 

> Switch from the **developer** account to the **admin** account before moving to the next step, as the developer account doesn’t have permission to edit the webhook.

9. Toggle the webhook off from the podinfo repository under **Webhooks** → **trigger_on_pr**.

![Disable Webhook](assets/disable-webhook.png)

10. Now log out from the current user profile and log back in as the administrator user. 

**Don’t merge the PR yet**. You’ll do it later at the CI/CD section. 

### Secret Scanning

You can use Gitness’ built-in [Gitleaks](https://github.com/gitleaks/gitleaks) integration to prevent hardcoded secrets like passwords, API keys, and tokens from being introduced into your Git repository during a push. 

You can enable secret scanning for individual repositories. Once enabled on a repo, any push event to that repo that contains a commit matching a recognized secret pattern is denied.

> [!NOTE] 

> Gitness Secret Scanning scans only new/changed code in commits that users attempt to push **after** you enable Secret Scanning on a repo. Secrets in existing/unchanged code aren't detected.

1. Go to the podinfo repository where you want to enable secret scanning and select **Settings**.
2. Select the **Security** tab.
3. Enable **Secret Scanning**.
4. Select **Save**.

Now, from your VS Code, create a new file `config` and add the following:

```shell
AWS_ACCESS_KEY="AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

Now try to push this commit to the remote **podinfo** repository. You should see the following error on the git log:

```
remote: Push contains secret:        
remote: 
remote:   aws-access-token in config:1        
remote:   Secret:  AKIAIOSFODNN7EXAMPLE        
remote:   Commit:  1469e0435ac535dfd552ab443248493fc4fb1192        
remote:   Details: Identified a pattern that may indicate AWS credentials, risking unauthorized cloud resource access and data breaches on AWS platforms.
```

The above AWS credentials are sample ones from the AWS documentation and are not valid credentials 🙂.

## Pipelines

Pipelines in Gitness help you automate steps in your software delivery process, such as initiating code builds, running automated tests, and deploying to a staging or production environment. Gitness supports multiple pipelines per repository.

### Basic Pipeline

Let’s create a simple pipeline in Gitness that will print the build number and the git commit sha for the current running build using [expression variables](https://docs.gitness.com/reference/pipelines/expression_variables).

Under **podinfo** repository, select **Pipelines** from the left navigation menu and then **+ New Pipeline**. Give this pipeline a name `hello-pipeline` and click **Create**.

Replace the existing pipeline with the following:

```YAML
kind: pipeline
spec:
  stages:
  - type: ci
    spec:
      steps:
      - name: print-build-info
        type: run
        spec:
          container: alpine
          script: |-
            echo "Build number: ${{ build.number }}"
            echo "Build commit: ${{ build.commit }}"
```

Click Save and **Run** → **Run Pipeline**. 

You’ll see something like this for the **print-build-info** stage execution:

```
latest: Pulling from alpine
Digest: sha256:c15c54221625d866fb6a98e74dcb1c6c4a24ab7c6585632db07f45219ea17efd
Status: Image is up to date for alpine:latest
+ echo "Build number: 1"\necho "Build commit: 5369dca9d8365a2b3540d6581ab52b4744387aef"
Build number: 1
Build commit: 5369dca9d8365a2b3540d6581ab52b4744387aef
```

### Secrets

Gitness offers a built-in secret manager to store and manage sensitive information, such as passwords, tokens, and ssh keys. In Gitness, secrets are managed at the project level. 

For this lab, `kube_token` and `webhook_url` are considered secrets. Let’s add these two to Gitness secret manager.

1. From the left hand navigation, click **Secrets** → **+ New Secret**.
2. Name this secret `kube_token` and paste the service account token of the Kubernetes cluster from a previous step as the value.
3. Repeat this process to create another secret `webhook_url` and use the webhook URL from webhook.site. (Optional) If you have a Slack webhook, you can use that value instead.

### Pipeline Conditions

[Conditions](https://docs.gitness.com/pipelines/conditions) limit pipeline [step](https://docs.gitness.com/category/steps) execution at runtime. Gitness sets [variables](https://docs.gitness.com/reference/pipelines/expression_variables) that can be used in conditions.

> [!TIP]

> Gitness supports multiple pipelines per repository. Creating a pipeline per [trigger](https://docs.gitness.com/pipelines/triggers) (push, pull request, tag) can reduce the need for conditions. We will cover triggers more in the next section.

Let's create a new pipeline **webhook-pipeline**. Select **Pipelines** from the left navigation menu and then **+ New Pipeline**. Give this pipeline a name **webhook-pipeline** and click **Create**.
 
Replace the existing pipeline with the following:

```YAML
kind: pipeline
spec:
  stages:
    - name: build
      type: ci
      spec:
        steps:
          - name: test
            type: run
            spec:
              container: alpine
              script: |-
                go test -v ./...
          - name: webhook
            type: plugin
            when: failure()
            spec:
              name: webhook
              inputs:
                content_type: application/json
                template: |
                  {
                    "name": "BuildBot Notification",
                    Repo: {{ repo.name }},
                    Build Number {{ build.number }},
                    Build Event: {{ build.event }},
                    Build Status: {{ build.status }},
                  }
                urls: ${{ secrets.get("webhook_url") }}
```

The test step fails because go is not installed in the alpine docker image by default, which causes the notify step to run. Check the webhook.site dashboard to ensure that a notification is sent.


