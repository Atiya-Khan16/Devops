# DevOps Practical 4 — Jenkins with AWS EC2

**Name:** Soham Sadanand Shetye  
**Roll No.:** K010

## Aim
Create a CI/CD pipeline by preparing an Ubuntu EC2 server on AWS, installing Git and Nginx, deploying a website, and configuring Jenkins to check out code from GitHub, build/check it, and deploy updates.

> **Important:** The supplied practical describes the EC2/Nginx setup and a Jenkins pipeline, but it does not fully document the Jenkins installation or secure remote deployment configuration. This guide separates the basic server deployment from the Jenkins pipeline and includes the extra configuration needed for a working setup.

## Architecture

1. Developer pushes code to GitHub.
2. Jenkins checks out the repository.
3. Jenkins runs validation/build steps.
4. Jenkins deploys the website to the web server.
5. Nginx serves the site to visitors.

---

## Part A — Create the GitHub repository

1. Create a GitHub repository, for example `devops_portfolio`.
2. Add your website files, such as `index.html` and `style.css`.
3. Copy the repository HTTPS URL. Replace the example URL below with your own.

```bash
https://github.com/YOUR_GITHUB_USERNAME/devops_portfolio.git
```

If you already have the repository, skip this section. Keep the repository public for the simplest classroom setup, or configure Git credentials in Jenkins for a private repository.

## Part B — Launch an AWS EC2 instance

1. Sign in to the AWS Console and open **EC2 → Instances → Launch instances**.
2. **Name:** `jenkins-server` (or another clear name).
3. **AMI:** Ubuntu Server 24.04 LTS.
4. **Instance type:** choose an eligible small instance for your lab and check current AWS pricing/free-tier eligibility before launching.
5. **Key pair:** create/download a key pair, e.g. `jenkins-key.pem`. Store it safely; AWS does not let you download the private key again.
6. **Network/Security Group:** allow these inbound rules:

| Type | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | My IP | Administration |
| HTTP | 80 | `0.0.0.0/0` | Public website |
| Custom TCP | 8080 | My IP only | Jenkins web UI, if Jenkins runs on this instance |

Do not expose port 8080 to everyone. For a production setup, prefer private access or a reverse proxy with HTTPS. Add HTTPS port 443 only when TLS is configured.

7. Launch the instance and wait until its status checks pass.
8. Copy the instance's **Public IPv4 address**.

## Part C — Connect to Ubuntu

### Option 1: EC2 Instance Connect (browser)

EC2 → Instances → select your instance → **Connect** → **EC2 Instance Connect** → **Connect**.

### Option 2: SSH from a terminal

On Linux/macOS/WSL, put the downloaded key in a safe location, then run:

```bash
chmod 400 ~/Downloads/jenkins-key.pem
ssh -i ~/Downloads/jenkins-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace `YOUR_EC2_PUBLIC_IP` with the actual address. On Windows PowerShell, use the correct local path to the `.pem` file. Do not upload or commit the private key to GitHub.

## Part D — Update Ubuntu and install Git + Nginx

Run these commands **inside the EC2 Ubuntu terminal**:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git nginx
git --version
nginx -v
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

The status should show `active (running)`. If it is not running, inspect the error with:

```bash
sudo journalctl -u nginx --no-pager -n 50
```

Open `http://YOUR_EC2_PUBLIC_IP` in a browser. You should see the Nginx welcome page. Make sure the EC2 security group permits inbound TCP port 80.

## Part E — Clone your website repository

Run on the EC2 instance. Replace the URL with your repository URL:

```bash
cd ~
git clone https://github.com/YOUR_GITHUB_USERNAME/devops_portfolio.git
cd devops_portfolio
ls -la
```

If the repository already exists on the server, update it instead:

```bash
cd ~/devops_portfolio
git pull origin main
```

If the default branch is not `main`, use the actual branch name.

## Part F — Deploy the website manually to Nginx

First confirm the repository contains `index.html`:

```bash
cd ~/devops_portfolio
test -f index.html && echo "index.html found" || echo "ERROR: index.html missing"
```

Copy the website files into Nginx's web root. This command copies the repository contents, including `.git`, so a cleaner option for a simple static site is to copy only the site files:

```bash
sudo find /var/www/html -mindepth 1 -maxdepth 1 -exec rm -rf {} +
sudo cp -r index.html style.css /var/www/html/
```

If your site has additional assets (for example an `images/` directory or JavaScript files), copy those too. If `style.css` does not exist, remove it from the command. For a generic directory copy, use care not to copy secrets or repository metadata into the public web root.

Check and restart Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost
```

Visit `http://YOUR_EC2_PUBLIC_IP` in a browser to verify the website.

## Part G — Install Java and Jenkins

> Jenkins needs a supported Java runtime. Follow the current Jenkins Linux installation instructions if package repository details change. The commands below use the Jenkins Debian package repository on Ubuntu.

Run on the EC2 Ubuntu instance:

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

Add the Jenkins signing key and package repository:

```bash
sudo install -d -m 0755 /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

If the Java package is unavailable on your Ubuntu image, check the current Jenkins documentation for the Java version supported by your Jenkins release rather than installing an unsupported runtime.

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

1. Open `http://YOUR_EC2_PUBLIC_IP:8080` only if your security group allows port 8080 from **your IP**.
2. Paste the initial password.
3. Choose **Install suggested plugins**.
4. Create the Jenkins administrator account.
5. Complete the setup wizard.

If the page does not open, check the service and port:

```bash
sudo systemctl status jenkins --no-pager
sudo journalctl -u jenkins --no-pager -n 80
sudo ss -lntp | grep 8080
```

**Security note:** Do not expose Jenkins directly to the whole internet. Restrict port 8080 to your own IP, and avoid storing AWS access keys or private SSH keys in scripts or a public repository.

## Part H — Create a Jenkins Pipeline job

1. Open Jenkins.
2. Select **New Item**.
3. Enter job name: `Portfolio-CI-CD`.
4. Select **Pipeline**, then click **OK**.
5. Under **Build Triggers**, choose one:
   - **GitHub hook trigger for GITScm polling** if you will configure a GitHub webhook; or
   - **Poll SCM** for periodic checking. Example schedule: `H/5 * * * *` (approximately every five minutes).
6. Under **Pipeline → Definition**, select **Pipeline script**.
7. Paste the script below, changing the repository URL and branch as needed.
8. Click **Save**, then **Build Now**.
9. Open the build number → **Console Output** to inspect the result.

### Pipeline script (basic checkout and validation)

This pipeline checks out the GitHub repository, lists files, and verifies that `index.html` exists. It uses Unix shell commands and is intended for a Jenkins controller running on Linux.

```groovy
pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/YOUR_GITHUB_USERNAME/devops_portfolio.git'
        BRANCH_NAME = 'main'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BRANCH_NAME}", url: "${REPO_URL}"
            }
        }

        stage('List Files') {
            steps {
                sh 'pwd && ls -la'
            }
        }

        stage('Validate HTML') {
            steps {
                sh 'test -f index.html'
                echo 'index.html found'
            }
        }

        stage('Validate CSS') {
            steps {
                script {
                    if (fileExists('style.css')) {
                        echo 'style.css found'
                    } else {
                        echo 'style.css not present; skipping optional CSS check'
                    }
                }
            }
        }

        stage('Build Result') {
            steps {
                echo 'Validation completed successfully'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully'
        }
        failure {
            echo 'Pipeline failed. Review Console Output.'
        }
    }
}
```

**Why this script differs from the screenshot:** The source practical's sample uses Windows `bat` commands (`dir`, `if exist`). An Ubuntu Jenkins agent must use Linux `sh` commands (`ls`, `test`). Do not mix the two command styles.

## Part I — Configure automatic GitHub triggers

### Option 1: Poll SCM (simplest)

In job **Configure → Build Triggers**, select **Poll SCM** and enter:

```text
H/5 * * * *
```

Save. Jenkins will periodically check for repository changes. This is polling, not an instant webhook.

### Option 2: GitHub webhook

1. In Jenkins, ensure the GitHub integration plugin is installed if needed.
2. In the GitHub repository, open **Settings → Webhooks → Add webhook**.
3. Payload URL: `http://YOUR_JENKINS_PUBLIC_HOST:8080/github-webhook/`
4. Content type: `application/json`.
5. Select **Just the push event**, then add the webhook.
6. Keep Jenkins access restricted and use HTTPS for a real deployment. A public HTTP Jenkins URL is not recommended for production.

A webhook will not work if the Jenkins endpoint is unreachable from GitHub or blocked by the network/security group.

## Part J — Deploy automatically (secure setup required)

The pipeline above only validates the code. To copy files to `/var/www/html` automatically, Jenkins needs a deliberately configured deployment path and permission. Do **not** solve permission errors by running Jenkins as root or giving it unrestricted `sudo`.

For a lab, a safer design is:
1. Run Jenkins on a separate controller or use a separate deployment agent.
2. Configure SSH credentials in **Jenkins → Manage Jenkins → Credentials**.
3. Add a narrowly scoped deployment command/script on the target EC2 host that copies only approved static-site files.
4. Add a pipeline deployment stage that invokes that script using the configured credential.
5. Verify the result at `http://YOUR_EC2_PUBLIC_IP`.

If Jenkins and Nginx are on the **same EC2 instance**, a local deployment stage can be configured with a carefully scoped directory permission or root-owned deployment helper. The source practical does not specify a safe permission configuration, so do not paste a broad `sudo cp` command into a Jenkins job without configuring permissions intentionally.

## Part K — Test the practical

1. Confirm the website loads at `http://YOUR_EC2_PUBLIC_IP`.
2. In Jenkins, click **Build Now**.
3. Open **Console Output** and confirm the stages pass.
4. Edit `index.html` locally and push the change to GitHub.
5. Wait for Poll SCM or trigger a build manually.
6. Confirm Jenkins checks out the latest commit. After adding a deployment stage, verify the updated website in the browser.

Example local Git commands (run on your own computer in the project directory):

```bash
git status
git add index.html style.css
git commit -m "Update portfolio page"
git push origin main
```

If you are using a different branch, replace `main`.

## Troubleshooting

### `Permission denied (publickey)` when connecting to EC2
- Confirm the correct `.pem` key and username (`ubuntu` for Ubuntu AMIs).
- On Linux/WSL/macOS, run `chmod 400 /path/to/jenkins-key.pem`.
- Confirm port 22 is allowed from your current public IP.

### Website cannot be reached
- Confirm the instance is running and use its current public IPv4 address.
- Confirm security-group inbound TCP port 80 is open.
- Check Nginx: `sudo systemctl status nginx --no-pager`
- Check configuration: `sudo nginx -t`
- Check logs: `sudo tail -n 50 /var/log/nginx/error.log`

### Jenkins cannot be reached on port 8080
- Confirm the security group allows TCP 8080 from your IP only.
- Run `sudo systemctl status jenkins --no-pager`.
- Inspect logs with `sudo journalctl -u jenkins --no-pager -n 80`.
- Check listening ports with `sudo ss -lntp | grep 8080`.

### Jenkins pipeline fails at `bat`
`bat` is for Windows agents. Use `sh` for Linux agents, as in the pipeline above.

### `index.html` not found
- Check the repository root with `ls -la`.
- Confirm the file is committed and pushed.
- If the HTML is in a subfolder, update the pipeline to use the correct path.

### `git clone` says repository not found
- Check the URL and repository name.
- For a private repository, configure a GitHub credential in Jenkins.

### Nginx returns the default page
- Confirm your files were copied to `/var/www/html`.
- Run `ls -la /var/www/html`.
- Reload Nginx after validating configuration: `sudo nginx -t && sudo systemctl reload nginx`.

### Jenkins service fails to start
```bash
sudo journalctl -u jenkins --no-pager -n 100
java -version
df -h
free -h
```
Check Java compatibility, available disk/RAM, and the service logs.

## Cleanup (avoid unexpected AWS charges)

When finished with the practical:
1. In AWS EC2, stop or terminate the instance if you no longer need it. **Terminating deletes the instance**.
2. Check and delete unused EBS volumes, snapshots, Elastic IPs, and other billable resources as appropriate.
3. Do not delete your `.pem` key until you have confirmed you no longer need it, and never share it.

## Result
An Ubuntu EC2 instance was prepared with Git and Nginx, a website was deployed, and a Jenkins Pipeline job was configured to check out code from GitHub and validate the website files. Automatic deployment after each push requires the additional secure deployment configuration described above.
