# OWASP Dependency-Check (Manual Install) + Jenkins Pipeline (Ubuntu/Debian)

This guide installs **Java 17**, **Jenkins**, **Maven**, and **OWASP Dependency-Check v12.2.0** manually, downloads the NVD database to a Jenkins-owned directory, and runs scans from a Jenkins Pipeline.

---

## Prerequisites

- Ubuntu/Debian VM with sudo access
- Outbound internet access to:
  - Jenkins apt repo
  - GitHub releases (Dependency-Check)
  - NVD (database updates)

---

## 1) Install Java 17, Jenkins, Maven

> Run these commands on your server.

```bash
sudo apt update -y

# Java 17 + basic tools
sudo apt install -y openjdk-17-jdk unzip wget

# Jenkins repo key + repo
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update

# Jenkins
sudo apt install -y jenkins

# Maven
sudo apt-get install -y maven
```

### Quick verification

```bash
java -version
mvn -version
sudo systemctl status jenkins --no-pager
```

---

## 2) Install OWASP Dependency-Check (v12.2.0) manually

```bash
cd /opt

sudo wget https://github.com/dependency-check/DependencyCheck/releases/download/v12.2.0/dependency-check-12.2.0-release.zip
sudo unzip dependency-check-12.2.0-release.zip

# The zip extracts a folder named "dependency-check"
sudo mv dependency-check /opt/dependency-check

sudo chmod +x /opt/dependency-check/bin/dependency-check.sh
```

### Verify version

```bash
/opt/dependency-check/bin/dependency-check.sh --version
```

Expected output should include: `12.2.0`

---

## 3) Database directory (CRITICAL)

Dependency-Check stores its database (NVD cache) in a directory. For Jenkins jobs, the safest option is a Jenkins-owned path.

```bash
sudo mkdir -p /var/lib/jenkins/dependency-check-data
sudo chown -R jenkins:jenkins /var/lib/jenkins/dependency-check-data
sudo chmod -R 755 /var/lib/jenkins/dependency-check-data
```

---

## 4) Download / Update the NVD database (one-time or scheduled)

> **Important:** Do **not** hardcode your API key in scripts committed to Git.
> Prefer environment variables or Jenkins Credentials.

### Recommended (use an environment variable)

```bash
export NVD_API_KEY="YOUR_NVD_API_KEY_HERE"

sudo /opt/dependency-check/bin/dependency-check.sh \
  --updateonly \
  --data /var/lib/jenkins/dependency-check-data \
  --nvdApiKey "$NVD_API_KEY"
```

### If you must run without exporting

```bash
sudo /opt/dependency-check/bin/dependency-check.sh \
  --updateonly \
  --data /var/lib/jenkins/dependency-check-data \
  --nvdApiKey "YOUR_NVD_API_KEY_HERE"
```

This will download/update the database into:

- `/var/lib/jenkins/dependency-check-data`

---

## 5) Jenkins Setup (UI)

### A) Install the plugin

Go to:

- **Manage Jenkins → Plugins → Available plugins**
- Search: **OWASP Dependency-Check**
- Install it (restart if asked)

### B) Configure the tool path

Go to:

- **Manage Jenkins → Global Tool Configuration → Dependency-Check**

Add an installation:

- **Name:** `DP-Check`
- **Install automatically:** ❌ unchecked (manual install)
- **Dependency-Check home:** `/opt/dependency-check`

✅ Save.

---

## 6) Jenkins Pipeline (Declarative)

Create a Pipeline job and paste this `Jenkinsfile`:

```groovy
pipeline {
  agent any

  stages {

    stage('Checkout') {
      steps {
        git url: 'https://github.com/jaiswaladi246/Petclinic.git', branch: 'main'
      }
    }

    stage('Build (skip tests)') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('OWASP Dependency-Check') {
      steps {
        sh 'mkdir -p dependency-check-report'

        dependencyCheck(
          odcInstallation: 'DP-Check',
          additionalArguments: """
            --scan .
            --format XML
            --format HTML
            --out dependency-check-report
            --data /var/lib/jenkins/dependency-check-data
            --noupdate
            --disableOssIndex
          """
        )

        dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'dependency-check-report/**', fingerprint: true
    }
  }
}
```

### Output artifacts

After a successful run, you should see:

- `dependency-check-report/dependency-check-report.html`
- `dependency-check-report/dependency-check-report.xml`

They’ll be archived in Jenkins as build artifacts.

---

## Troubleshooting

### 1) Exit code 13 / “Unable to connect to the dependency-check database”
Most commonly caused by permissions on the `--data` directory.

Fix:

```bash
sudo chown -R jenkins:jenkins /var/lib/jenkins/dependency-check-data
sudo chmod -R 755 /var/lib/jenkins/dependency-check-data
```

Also make sure the Jenkins service user is actually `jenkins`:

```bash
getent passwd jenkins
```

### 2) “No documents exist”
Usually happens when Dependency-Check didn’t generate a report (scan failed early) or the publisher pattern is wrong.

Check:
- The scan step completed successfully
- The file exists in the workspace:
  - `dependency-check-report/dependency-check-report.xml`

### 3) Slow / failing updates
- Ensure you are using an NVD API key.
- Ensure outbound internet access is allowed from the server.

---

## Notes / Best Practices

- Keep the database directory persistent and Jenkins-owned:
  - `/var/lib/jenkins/dependency-check-data`
- Use **Jenkins Credentials** for the NVD API key and inject it as an env var in Pipeline.
- For production, consider running `--updateonly` via a scheduled job so pipelines can use `--noupdate`.

---

## Reference Paths Used

- Dependency-Check binary:
  - `/opt/dependency-check/bin/dependency-check.sh`
- Dependency-Check home (Jenkins tool config):
  - `/opt/dependency-check`
- Database directory:
  - `/var/lib/jenkins/dependency-check-data`
- Pipeline output folder:
  - `dependency-check-report/`
