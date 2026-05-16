
# **Automated Vulnerability Discovery & Remediation Pipeline for Containerized WordPress**  

---

## 1. Environment Setup

### 1.1 Steps Taken

**Local development environment**  
- Created a `docker/docker-compose.yml` file defining two services:  
  - `wordpress:5.8` – the intentionally vulnerable WordPress instance  
  - `mysql:5.7` – the database backend  
- Used environment variable substitution (`${MYSQL_USER}`, etc.) to avoid hardcoding credentials in the compose file.  
- A local `.env` file (git‑ignored) supplies the actual values for manual testing.

**CI/CD environment (GitHub Actions)**  
- Created `.github/workflows/scan.yml` to fully automate deployment and scanning.  
- The workflow:  
  - Spins up a fresh MySQL and WordPress container on every run.  
  - Uses GitHub Secrets for all database and WordPress admin credentials.  
  - Installs WordPress via WP‑CLI.  
  - Pulls the `wpscanteam/wpscan` Docker image and executes a scan against the site.  
  - Handles WPScan exit codes intelligently (exit code 5 = vulnerabilities found) to always preserve scan artifacts.  
  - Uploads the JSON scan report as a GitHub Artifact and commits it to the repository under `/scans/`.  
  - Fails the pipeline only if WPScan reports vulnerabilities (exit code 5).  

- Created `.github/workflows/dockerfile-push.yml` to build the hardened WordPress image and push it to **both** Docker Hub and GitHub Container Registry (GHCR).  

### 1.2 Challenges Encountered & How They Were Solved

1. **WPScan exit code 5 breaking the pipeline**  
   WPScan returns exit code 5 when vulnerabilities are found. Initially, this caused the workflow step to fail immediately, preventing the scan report from being saved.  
   *Solution:* Captured the exit code in a file (`scan-exit-code.txt`), allowed the step to continue for exit codes 0 and 5, and later used that file as the security gate. This ensures the scan artifact is always preserved, and the pipeline fails **after** the evidence is recorded.

2. **Docker image pull progress polluting WPScan error logs**  
   Docker outputs image pull progress to stderr, which was redirected into `wpscan-errors.log`, making the log bloated and hard to read.  
   *Solution:* Added a separate `docker pull` step before the scan, then used `--pull=never` in the WPScan run. Now `wpscan-errors.log` contains only real WPScan warnings.

3. **Insufficient Docker Hub access token scopes**  
   The initial push to Docker Hub failed with “unauthorized: access token has insufficient scopes”.  
   *Solution:* Created a new Docker Hub access token with **Read & Write** permissions, stored it in the `DOCKER_PASSWORD` secret, and the push succeeded.

4. **GHCR repository name case sensitivity**  
   GitHub Container Registry requires repository names to be all lowercase. Using `${{ github.repository_owner }}` directly (which contained an uppercase letter) caused an “invalid tag” error.  
   *Solution:* Added a step to convert the owner name to lowercase using `tr` and used the output variable in the GHCR tag.

---

## 2. Findings Overview

### 2.1 Key Vulnerabilities (Before Remediation)

The baseline scan of **WordPress 5.8.3** revealed **36 core vulnerabilities**, including:

| Severity | Example Vulnerability | CVE / WPVDB ID |
|----------|----------------------|----------------|
| High | Unauthenticated Blind SSRF via DNS Rebinding | CVE-2022-3590 |
| High | SQLi via Link API | WPVDB 601b0bf9 |
| Medium | Multiple Reflected/Stored XSS (wp‑mail.php, Customizer, Gutenberg…) | WPVDB 622893b0, 3b1573d4, … |
| Medium | Prototype Pollution in jQuery | WPVDB 1ac912c1 |
| Low/Info | XML‑RPC enabled, user enumeration (`admin` found), readme.html exposed | — |

### 2.2 Risk Assessment

- **Critical impact:** Blind SSRF can allow an attacker to make the server perform requests to internal networks, potentially bypassing firewalls.  
- **High impact:** SQL injection in the Link API could lead to full database compromise.  
- **Medium impact:** Multiple XSS vulnerabilities could enable session hijacking, defacement, or malicious redirects.  
- **Information leakage:** Exposed version headers, readme, and enabled XML‑RPC provide attackers with precise targeting information and a brute‑force attack surface.

### 2.3 Exploitation Scenarios

- **XML‑RPC brute‑force:** With the `admin` user enumerated, an attacker could use `xmlrpc.php` to attempt thousands of password guesses per second.  
- **SSRF to internal services:** The blind SSRF could be used to probe internal cloud metadata endpoints (e.g., `169.254.169.254`) and extract sensitive credentials.  
- **Stored XSS via Comments:** An attacker could inject malicious JavaScript into a comment; when an admin views the comment moderation page, the script could steal the admin’s session cookie.  


---

## 3. Remediation Steps

### 3.1 Patching and Updating

- **WordPress core:** Updated from `5.8.3` to the **latest stable release**. After the initial hardening attempt using WordPress 6.4.2 (which left 8 outdated vulnerabilities), the base image was changed to `wordpress:latest`, ensuring all current core patches are applied automatically.  
- **Themes:** The default theme now uses the version bundled with the latest WordPress, containing no known vulnerabilities.  
- **Plugins:** No unnecessary plugins were present; none needed patching.  

### 3.2 Hardening Measures

All hardening was embedded in `Dockerfile.hardened`:

| Measure | Implementation |
|---------|----------------|
| Non‑root user | Created `wpuser`, reconfigured Apache to drop privileges to it at runtime (`APACHE_RUN_USER=wpuser`). |
| Remove unnecessary packages | Purged `telnet`, `ftp`, `netcat`. |
| Disable XML‑RPC | Apache `<Files xmlrpc.php>` block denying all access. |
| Hide server information | `ServerSignature Off`, `ServerTokens Prod`, `Header unset X-Powered-By`. |
| Block directory listing | `Options -Indexes`. |

These measures are applied on top of the latest WordPress base image, creating a truly hardened artifact.

### 3.3 Before/After Scan Evidence

**Before (WordPress 5.8.3)**  
- Core version: 5.8.3 (`status: "insecure"`)  
- Vulnerabilities: **36**  
- Scan results stored in `/scans/before-scan.txt`

**After (WordPress latest, fully hardened)**  
- WPScan exit code: **0** (no vulnerabilities found).    
- Scan results automatically stored in `/scans/scan-result.json` 

---

## 4. Fixed Image Build

- **GitHub Repository:**  
  `https://github.com/Vladutchi/wordpress-devsecops`  
  (Contains all Dockerfiles, compose files, workflow definitions, and scan artifacts.)

- **Docker Hub Image:**  
  `https://hub.docker.com/r/vladutchi/wordpress-hardened`  
  (Tag: `latest`)

- **GitHub Container Registry:**  
  `ghcr.io/vladutchi/wordpress-hardened:latest`  
  (Available under the repository’s Packages section.)


---

## 5. Tooling Justification

| Tool | Why Chosen |
|------|------------|
| **Docker** | Enables reproducible, isolated environments. The same container image is scanned locally and in CI, eliminating “it works on my machine” discrepancies. |
| **WPScan** | Industry‑standard WordPress vulnerability scanner. Provides a detailed JSON output with CVE/WPVDB references, perfect for automated analysis. |
| **GitHub Actions** | Free for public repositories, deeply integrated with the codebase. Supports secret management, conditional triggers, and artifact storage – all essential for a DevSecOps pipeline. |
| **Docker Hub / GHCR** | Public registries for sharing hardened images. Dual push demonstrates multi‑registry support, a common enterprise requirement. |
| **WP‑CLI** | Automates WordPress installation inside the container without manual GUI interaction, keeping the pipeline fully scripted. |

---

## 6. DevSecOps Strategy (Shift‑Left Security)

### 6.1 How the Workflow Embeds Security Early

1. **Every code change triggers a security scan**  
   The `scan.yml` workflow runs automatically after a successful hardened image build. A vulnerability found at this stage prevents the build from being considered safe – long before the image reaches a production registry.

2. **Security gates are automated**  
   The pipeline does not rely on human review; WPScan’s exit code directly determines whether the build passes or fails. A clean scan (exit code 0) confirms the hardened image contains no known vulnerabilities.

3. **Infrastructure as Code**  
   Hardening measures are scripted in the Dockerfile, making them version‑controlled, auditable, and reproducible. New team members can understand exactly what security configurations are applied.

4. **Secrets never touch source code**  
   All credentials are stored in GitHub Secrets and injected at runtime. This prevents accidental exposure in logs or commits, a fundamental DevSecOps practice.

5. **Semi‑automatic remediation loop**  
   - A developer updates `Dockerfile.hardened` to patch a vulnerability.  
   - The `dockerfile-push.yml` workflow rebuilds and pushes the new image.  
   - `scan.yml` automatically triggers, scans the updated image, and commits the new scan report.  
   - If the scan passes (exit 0), the image is considered “safe” for downstream use.  


```
[Change Dockerfile.hardened] 
      → (Build & Push workflow) → [New image on Docker Hub / GHCR]
      → (trigger on completion) → [Scan workflow] 
      → [WPScan] → [Commit results to /scans/] 
      → [Pass/Fail gate]
```


