# CI/CD and Build Security — TryHackMe Walkthrough

A hands-on walkthrough documenting my progress through the **CI/CD and Build Security** room on TryHackMe. This room explores real-world misconfigurations across every stage of a software build pipeline — from the source code repository, to the build server and agents, to the pipeline logic itself, to environment segregation and secrets management. Each task builds on the last, showing how a single overlooked setting can cascade into a full compromise of production infrastructure.

---

## 1. Network Setup

**The Task:**
Before any exploitation could begin, I needed to connect to the room's isolated lab network and get access to the internal GitLab and Jenkins instances used throughout the room.

**How it was done:**
- Launched the Web-based AttackBox directly from the room page, which automatically connects to the lab's VPN network — no manual OpenVPN configuration required.
- Verified connectivity by pinging the GitLab host's IP address from the AttackBox terminal.
- Ran `ip a` to identify the dedicated `cicd` network interface and noted its assigned IP. This IP became important later, as it's the address I'd use as my "attacker IP" whenever setting up reverse shell listeners.
- Since the lab uses internal hostnames (`gitlab.tryhackme.loc`, `jenkins.tryhackme.loc`) rather than public DNS, I manually added both hostnames and their corresponding IPs to `/etc/hosts` so my browser and tools could resolve them.
- Registered with "Mother" — the lab's SSH-based tracking system — which is used throughout the room to log in, submit proof of each successful compromise, and receive flags as rewards for completing objectives.

<img width="1732" height="897" alt="01-network-diagram" src="https://github.com/user-attachments/assets/22c6254b-d1fb-4a62-9b38-a525b9787e13" />

<img width="1917" height="851" alt="02-mother-registration" src="https://github.com/user-attachments/assets/67c6275b-62c2-4087-8b2f-b13a275158e9" />


---

## 2. Building a CI/CD Pipeline from Scratch

**The Task:**
To understand how a real CI/CD pipeline operates before trying to break one, I first built and ran a working pipeline of my own.

**How it was done:**
- Registered a new account on the lab's GitLab instance.
- Forked a starter project called `BasicBuild`, which included a `.gitlab-ci.yml` file defining three pipeline stages: **build**, **test**, and **deploy**.
- Reviewed the CI configuration to understand GitLab's pipeline syntax — how jobs are assigned to stages, how stages run in sequence while jobs within a stage run in parallel, and how the deploy stage copied a small PHP web application to a directory and served it using PHP's built-in web server.
- Installed the GitLab Runner application on the AttackBox and registered it against my forked project using a project-specific registration token, selecting the `shell` executor so jobs would run directly on the AttackBox.
- Configured the runner to accept **untagged jobs**, since the sample pipeline didn't use job tags, and removed a known problematic `.bash_logout` file that can break runner functionality due to version mismatches.
- Triggered the pipeline by committing a small change to the README file, then watched the pipeline execute through all three stages in GitLab's Pipelines view.
- Verified successful deployment by browsing to the locally-hosted PHP application, which confirmed the entire pipeline — from commit to live deployment — worked end-to-end.

<img width="1390" height="865" alt="03-runner-registration-terminal" src="https://github.com/user-attachments/assets/ca15b78c-3e7b-4810-9cd1-23afca761741" />
<img width="1392" height="910" alt="04-runner-untagged-jobs" src="https://github.com/user-attachments/assets/2b6baac7-5c8b-484d-86dc-c62223ba1a18" />

<img width="1387" height="911" alt="05-pipeline-passed" src="https://github.com/user-attachments/assets/eb900342-a1dd-417a-acf7-81cd68ab0052" />


---

## 3. Source Code Security — Enumerating Exposed Repositories

**The Task:**
This task explored a subtle but common organizational mistake: leaving GitLab registration open to anyone on the internal network, combined with repositories that are technically "private" to the company but effectively public to any employee who registers an account.

**How it was done:**
- Generated a Personal Access Token from my GitLab account settings, scoped with `api`, `read_api`, and `read_repository` permissions — required because GitLab's API doesn't accept plain username/password credentials.
- Wrote and ran a Python script using the `python-gitlab` library that authenticated with the token, listed every project visible to my account, and downloaded a zipped copy of each one automatically. This simulated how a real attacker would perform bulk, automated reconnaissance rather than manually browsing each repo.
- Extracted all downloaded repositories and searched their contents recursively for common secret-related keywords (`secret`, `key`, `api`, etc.).
- Found a hardcoded API key inside a `Dockerfile` belonging to a "Mobile App" project — a secret that should have been injected securely via CI/CD variables, not committed directly into source code.
- This demonstrated how source code intended to be internal-only can still leak sensitive credentials the moment access control is even slightly too permissive.

<img width="1917" height="867" alt="07-access-tokens-page" src="https://github.com/user-attachments/assets/6b7a8e45-0ed6-4b62-892e-9d8bf5598b10" />

---

## 4. Build Process Security — Exploiting an On-Merge Build

**The Task:**
Here I explored a "toxic combination" in CI/CD configuration: pipelines that automatically build code the moment a merge request is opened — before any human review takes place.

**How it was done:**
- Forked a target repository (`Merge-Test`) that used a `Jenkinsfile` to define its pipeline. This file is executed by Jenkins whenever GitLab sends it a webhook notification, such as when a merge request is created.
- Since I had write access to my own fork — including the Jenkinsfile itself — I replaced the pipeline's build step with a payload that downloads and executes a reverse shell script via `curl`.
- Hosted the reverse shell script on a simple Python HTTP server and started a `netcat` listener to catch the incoming connection.
- Committed the modified Jenkinsfile and opened a brand-new merge request back to the original repository, which triggered the on-merge build webhook.
- Within seconds, Jenkins pulled my malicious Jenkinsfile, executed it on its build agent, and I received a fully interactive reverse shell — proving that CI/CD automation is effectively remote code execution as a built-in feature, and that any user able to influence pipeline code (even just in their own fork, prior to merge) can weaponize it.

<img width="1416" height="502" alt="09-jagent-shell-received" src="https://github.com/user-attachments/assets/14d9765f-ca79-46ff-bf84-b3e5f64c2c45" />

<img width="1917" height="306" alt="10-jagent-whoami-id" src="https://github.com/user-attachments/assets/5f6f8dc2-0b73-446c-ba61-e4fd6154810e" />


---

## 5. Build Server Security — Attacking Jenkins Directly

**The Task:**
Rather than exploiting the pipeline logic, this task targeted the Jenkins build server itself, focusing on a classic but still surprisingly common weakness: default credentials.

**How it was done:**
- Accessed the exposed Jenkins web interface directly over the network and successfully authenticated using the well-known default credential pair `jenkins:jenkins`.
- Switched to Metasploit and loaded the `exploit/multi/http/jenkins_script_console` module, which abuses Jenkins' built-in Groovy scripting console (accessible to authenticated users) to achieve remote code execution.
- Configured the module with the correct target OS, a Linux-compatible payload (`linux/x64/meterpreter/bind_tcp`), the Jenkins credentials, and connection details (host, port, target URI).
- Ran the exploit, which logged into Jenkins, obtained a CSRF token, uploaded a staged payload through the script console, and opened a fully-privileged Meterpreter session directly on the Jenkins server — demonstrating that weak build server credentials can lead to complete compromise of the infrastructure orchestrating every pipeline in the organization.

<img width="1102" height="862" alt="11-jenkins-dashboard-login" src="https://github.com/user-attachments/assets/b498ed13-8973-46f1-9a12-6acbcf00d50f" />

---

## 6. Pipeline Security — Bypassing Access Gates and the Two-Person Rule

**The Task:**
This task examined a scenario where an organization believed their `main` branch was protected because direct pushes were disabled — but the actual approval process wasn't technically enforced.

**How it was done:**
- Authenticated to GitLab using a lower-privileged developer account (`anatacker`) that had been granted access to a specific project.
- Attempted to edit a file directly on the protected `main` branch; GitLab blocked the direct push and instead forced the change into a merge request.
- Reviewing the merge request page revealed that although company policy stated a manager must approve changes, GitLab's actual configuration marked approvals as **optional** — meaning there was no technical control preventing the same user who opened the merge request from also approving and merging it.
- Approved and merged my own merge request, completely bypassing the intended two-person review process.
- Since the project's GitLab Runner was configured to execute jobs only on the `main` branch, gaining the ability to push to `main` also gave me the ability to modify `.gitlab-ci.yml` and trigger arbitrary code execution on that runner — repeating the reverse shell technique from the previous task, this time by having my own malicious merge request self-approved and merged.
- This highlighted a key lesson: **policy without technical enforcement is not real security.**

<img width="1401" height="912" alt="12-merged-by-ana-tacker" src="https://github.com/user-attachments/assets/0de43fe1-a912-4e62-a4cd-960c44970113" />
<img width="657" height="210" alt="13b-merge-immediately" src="https://github.com/user-attachments/assets/63ac946c-1a49-47dd-b183-2ded477e31cf" />

<img width="1020" height="460" alt="14b-grunner-shell-received" src="https://github.com/user-attachments/assets/0753beca-61e5-4857-9bf2-1e533c6baae2" />


---

## 7. Environment Security — Compromising a Shared Build Runner

**The Task:**
This task looked at what happens when development (DEV) and production (PROD) environments — despite being logically separated — still share the exact same underlying build infrastructure.

**How it was done:**
- Reviewed the pipeline history for both the DEV (`staging`) and PROD (`production`) environments under the project's Environments view in GitLab.
- Inspected individual job details for each environment's most recent build and compared the runner metadata shown in each job log.
- Confirmed that both DEV and PROD builds were executing on the **exact same GitLab Runner** — meaning access limited to a "less sensitive" DEV branch could still be leveraged to interact with the infrastructure directly responsible for production deployments.
- Since I had direct commit access to the DEV branch (even though it still required a self-approvable merge request, similar to the previous task), I modified its `.gitlab-ci.yml` to run the same reverse-shell payload technique used earlier.
- Once the merge request was approved and merged, the shared runner executed my payload, and I obtained a shell on the same host responsible for production deployments — despite only having "development" level access.
- This demonstrated why environment segregation must extend to the infrastructure layer, not just source code branches and permissions.

<img width="1402" height="866" alt="13-environments-page" src="https://github.com/user-attachments/assets/666d222a-ca10-4437-8f34-ecf72dfed3f2" />

<img width="1397" height="862" alt="14-production-job-log" src="https://github.com/user-attachments/assets/a015aaa8-232a-4ac2-ac19-207aefc7da8e" />

---

## 8. Build Secrets Security — Exploiting Improperly Scoped CI/CD Variables

**The Task:**
The final task examined whether CI/CD variables — GitLab's built-in mechanism for storing secrets like API keys — were properly scoped to only the environments that should have access to them.

**How it was done:**
- Reviewed the DEV branch's `.gitlab-ci.yml` file and noted it referenced a variable named `API_KEY_DEV` in its deployment script.
- Compared this against the PROD branch's pipeline, which referenced a differently-named variable, `API_KEY`, and echoed its value into the job log as part of the deployment step.
- As a developer with only DEV-level access, I didn't have permission to view the CI/CD variables list directly in project settings. However, since pipeline **execution** on DEV wasn't restricted from referencing *any* variable name, I edited the DEV branch's CI file to instead echo `${API_KEY}` — the production variable — rather than the DEV-scoped one.
- Committed the change through the same self-mergeable merge request process used in prior tasks, then triggered the pipeline and inspected the resulting job log.
- The production `API_KEY` value was printed directly in the DEV pipeline's console output — confirming that naming a variable differently per environment provides no actual security if the CI system doesn't enforce which branches or environments are permitted to access which variables.
- This tied together the core lesson of the entire room: **every layer of the pipeline — source, build process, build server, pipeline logic, environments, and secrets — must be secured independently, because a weakness in any single layer can compromise all the others.**

<img width="1397" height="910" alt="15-dev-branch-ci-file" src="https://github.com/user-attachments/assets/6ac11297-adba-40b6-aa4a-90c69590c3e5" />


---

## Key Takeaways

- **Source code exposure** often stems from overly permissive internal access, not just external threats — "internal only" is not the same as "secure."
- **Pipeline automation is effectively remote code execution as a feature.** Any weakness in *what* triggers a build, *who* can trigger it, and *where* it runs directly expands the attack surface.
- **Policy without technical enforcement is not security.** A written approval policy means nothing if the platform technically allows self-approval.
- **Shared infrastructure (runners, secrets) between environments of different trust levels** creates a direct path from a lower-privilege compromise to a higher-privilege one.
- **Secrets management requires explicit scoping and masking** — simply naming variables differently per environment does nothing if the CI system doesn't enforce which jobs can access which variables.
- **Defense in depth applies to pipelines just as much as networks.** Securing the source code but not the build server, or securing the build server but not the pipeline logic, still leaves the door open.

---

*This write-up documents my personal learning process working through TryHackMe's DevSecOps / CI-CD Security learning path.*
