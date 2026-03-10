# DevSecOps Pipeline - Grupo 05

This repository contains a Jenkins DevSecOps pipeline that performs automated security analysis and build.

## Tools Used

- Jenkins
- Semgrep (SAST)
- Snyk (Container Security)
- OWASP ZAP (DAST)
- Docker
- Maven

## Pipeline Stages

1. Checkout source code
2. Static Security Analysis (Semgrep)
3. Docker Image Build
4. Container Security Scan (Snyk)
5. Dynamic Security Test (OWASP ZAP)
6. Build with Maven
7. Unit Testing

## Security Reports

The pipeline generates:

- semgrep-report.json
- snyk-report.json
- zap-report.json

These reports are archived in Jenkins artifacts.

## Source Code Scanned

https://github.com/michix14/lab4
