# Lab 5: Docker Image Vulnerability Scanning with Trivy

## 1. Clone the Repository

Clone the repository containing the application and Dockerfile, then navigate to the project directory.

```bash
git clone https://github.com/Ibrahim-Adel15/TrivyImage.git
cd TrivyImage
ls
```

## 2. Build the Docker Image

Build the Docker image and tag it as `trivy-lab:v1`.

```bash
docker build -t trivy-lab:v1 .
```

Verify that the image was created successfully:

```bash
docker images
```

**Explanation:** The `-t` option assigns a name and tag to the image, while `.` specifies the current directory as the build context.

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

## 4. Scan the Image for Vulnerabilities

Scan the Docker image for vulnerabilities of all severities:

```bash
trivy image trivy-lab:v1
```

**Explanation:** Trivy checks the image's operating system packages and supported application dependencies for known vulnerabilities.

## 5. Scan HIGH and CRITICAL Vulnerabilities

Display only high- and critical-severity vulnerabilities:

```bash
trivy image --severity HIGH,CRITICAL trivy-lab:v1
```

Save the filtered results as a text report:

```bash
mkdir -p reports

trivy image --severity HIGH,CRITICAL trivy-lab:v1 \
  -o reports/high-critical.txt
```

## 6. Generate HTML and JSON Reports

Download the official Trivy HTML template:

```bash
wget https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl \
  -O reports/contrib-html.tpl
```

Generate an HTML report:

```bash
trivy image --format template \
  --template "@reports/contrib-html.tpl" \
  -o reports/trivy-report.html \
  trivy-lab:v1
```

Generate a JSON report:

```bash
trivy image --format json \
  -o reports/trivy-report.json \
  trivy-lab:v1
```

The reports are saved in the `reports/` directory:

- `contrib-html.tpl` — the HTML report template.
- `high-critical.txt` — HIGH and CRITICAL vulnerability findings.
- `trivy-report.html` — the HTML vulnerability report.
- `trivy-report.json` — the machine-readable JSON report.

To open the HTML report in Windows:

```bash
explorer.exe reports/trivy-report.html
```

**Explanation:** HTML reports are useful for reviewing scan results in a browser, while JSON reports are suitable for further processing and automation.

## 7. Review the Results

Review the generated reports and identify:

- **Vulnerability ID:** The identifier of the security issue.
- **Severity:** The risk level, such as LOW, MEDIUM, HIGH, or CRITICAL.
- **Affected Package:** The vulnerable software package.
- **Installed Version:** The version currently included in the image.
- **Fixed Version:** The version that addresses the vulnerability, when available.

Prioritize HIGH and CRITICAL vulnerabilities. Where possible, update affected packages or the base image, rebuild the Docker image, and scan it again to verify the improvements.

## Conclusion

In this lab, I built a Docker image, installed Trivy, scanned the image for known vulnerabilities, filtered HIGH and CRITICAL findings, and generated HTML, JSON, and text reports.

**Tools Used:** Docker, Trivy, Ubuntu/WSL, Git, HTML, and JSON.
