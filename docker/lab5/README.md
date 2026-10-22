# Lab 5: Docker Image Vulnerability Scanning with Trivy

## 1. Clone the Repository
Clone the repository containing the application and Dockerfile, then navigate to the project directory.

```bash
git clone https://github.com/Ibrahim-Adel15/Trivylmage.git
cd Trivylmage
ls
```

## 2. Build the Docker Image
Build the image using the provided Dockerfile and tag it as `trivy-lab:v1`.

```bash
docker build -t trivy-lab:v1 .
```

Verify that the image was created successfully:

```bash
docker images
```

**Explanation:** `-t` assigns a name and tag to the image, while `.` specifies the current directory as the build context.

## 3. Install Trivy
Install Trivy on Ubuntu/WSL using the official installation instructions:

```bash
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt-get update
sudo apt-get install trivy
```

Verify the installation:

```bash
trivy --version
```

## 4. Scan the Image for All Vulnerabilities
Scan the Docker image and display vulnerabilities of all severities.

```bash
trivy image trivy-lab:v1
```

**Explanation:** Trivy examines the image's operating system packages and supported application dependencies for known vulnerabilities.

## 5. Scan Only HIGH and CRITICAL Vulnerabilities
Filter the scan results to show only high- and critical-severity issues.

```bash
trivy image --severity HIGH,CRITICAL trivy-lab:v1
```

**Explanation:** This command helps prioritize vulnerabilities that may require urgent attention.

## 6. Generate an HTML Vulnerability Report
Download the official Trivy HTML template:

```bash
wget https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl -O html.tpl
```

Generate an HTML report containing all vulnerability severities:

```bash
trivy image --format template --template "@html.tpl" -o trivy-report.html trivy-lab:v1
```

Generate a second report containing only HIGH and CRITICAL vulnerabilities:

```bash
trivy image --severity HIGH,CRITICAL --format template --template "@html.tpl" -o trivy-high-critical-report.html trivy-lab:v1
```

Open the reports in your browser:

```bash
explorer.exe trivy-report.html
explorer.exe trivy-high-critical-report.html
```

**Explanation:** The HTML reports provide a readable view of the detected vulnerabilities and their details. The second report focuses on HIGH and CRITICAL findings.

## 7. Review the Results
Review the reports and identify:

- **Vulnerability ID:** The identifier of the security issue.
- **Severity:** The risk level, such as LOW, MEDIUM, HIGH, or CRITICAL.
- **Affected Package:** The vulnerable software package.
- **Installed Version:** The version currently included in the image.
- **Fixed Version:** The version that addresses the vulnerability, when available.

Prioritize HIGH and CRITICAL vulnerabilities. Where possible, update the affected packages or base image, rebuild the Docker image, and run Trivy again to verify the improvements.

## Conclusion
In this lab, I built a Docker image, installed Trivy, scanned the image for known vulnerabilities, filtered HIGH and CRITICAL findings, and generated HTML security reports to review the results.

**Tools Used:** Docker, Trivy, Ubuntu/WSL, Git, and HTML reports.
