\# Cloud Infrastructure \& Linux Production Incident Case Study



A comprehensive diagnostic and remediation runbook documenting multi-layer infrastructure failures encountered on an AWS EC2 Ubuntu instance hosting a Python microservice behind an Nginx reverse proxy.



\## Architecture Overview

\* \*\*Hosting:\*\* AWS EC2 (Ubuntu 24.04 LTS)

\* \*\*Backend Runtime:\*\* Python 3.12 (`BaseHTTPRequestHandler`)

\* \*\*Process Supervisor:\*\* `systemd`

\* \*\*Reverse Proxy / Ingress:\*\* Nginx 1.24

\* \*\*Configuration Storage:\*\* AWS S3



\---



\## The Incident

Upon deployment, the external web application was completely unreachable via standard HTTP (port 80). Systematic triaging across IAM, local filesystem permissions, network sockets, and reverse proxy routing identified four distinct cascading failures.



\### Cascading Failure Points \& Root Causes



1\. \*\*Linux Filesystem Permissions (`\[Errno 13]`)\*\*

&#x20;  \* \*Symptom:\* `app.service` crashed immediately on bootstrap.

&#x20;  \* \*Root Cause:\* The systemd service was executed by unprivileged user `ubuntu`, which lacked write permissions to initialize logging directories under `/var/log/app\_log/`.

&#x20;  \* \*Remediation:\* Created target directories and granted recursive ownership:

&#x20;    ```bash

&#x20;    mkdir -p /var/log/app\_log

&#x20;    chown -R ubuntu:ubuntu /var/log/app\_log

&#x20;    chmod 755 /var/log/app\_log

&#x20;    ```



2\. \*\*Cloud IAM Authorization (`AccessDenied`)\*\*

&#x20;  \* \*Symptom:\* Application failed runtime initialization when fetching dynamic configurations from AWS S3 (`botocore.exceptions.ClientError`).

&#x20;  \* \*Root Cause:\* The instance profile IAM role lacked `s3:GetObject` permissions for the configuration bucket.

&#x20;  \* \*Remediation:\* Attached `AmazonS3ReadOnlyAccess` policy to the instance's IAM role.



3\. \*\*Port 8000 Socket Collision (`\[Errno 98]`)\*\*

&#x20;  \* \*Symptom:\* Backend failed with `Address already in use`.

&#x20;  \* \*Root Cause:\* An unauthorized service (`rogue.service`) holding `Restart=always` monopolized TCP port 8000 using Python's `http.server`.

&#x20;  \* \*Remediation:\* Terminated process conflicts, disabled the unit, and deleted the configuration:

&#x20;    ```bash

&#x20;    systemctl stop rogue.service

&#x20;    systemctl disable rogue.service

&#x20;    rm -f /etc/systemd/system/rogue.service

&#x20;    systemctl daemon-reload

&#x20;    ```



4\. \*\*Nginx Reverse Proxy Misrouting\*\*

&#x20;  \* \*Symptom:\* Port 80 served the default static Ubuntu landing page instead of upstream backend responses.

&#x20;  \* \*Root Cause:\* Default site configurations took precedence, and `/etc/nginx/sites-available/app` lacked the `proxy\_pass` upstream directive.

&#x20;  \* \*Remediation:\* Unlinked the default host and injected upstream forwarding:

&#x20;    ```bash

&#x20;    rm -f /etc/nginx/sites-enabled/default

&#x20;    sed -i '/location \\/ {/a \\        proxy\_pass \[http://127.0.0.1:8000](http://127.0.0.1:8000);' /etc/nginx/sites-available/app

&#x20;    nginx -t \&\& systemctl reload nginx

&#x20;    ```



\---



\## Verification \& Health Checks

\* \*\*Internal Backend Health:\*\* `curl -v http://127.0.0.1:8000/` returned `HTTP/1.0 200 OK`.

\* \*\*Ingress Verification:\*\* `curl -v http://<INSTANCE\_IP>/` returned `HTTP/1.1 200 OK` routed via Nginx.

\* \*\*Process Persistence:\*\* Verified `systemctl is-enabled app.service nginx` both returned `enabled`.



\### Live Deployment Verification


https://github.com/Mcakohli/cloud-infra-troubleshooting-case-study/commit/8e6bb6698b8e5693e63b449684c5ce35c5ee1df2

<img src="Docs/live\_verification.png" alt="Application Live Verification" width="100%" />

