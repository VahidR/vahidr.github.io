---
title: "Securing Kubernetes: Avoiding Hard-Coded Passwords in Manifests and Helm Charts"
date: 2024-02-29T11:48:57+01:00
type: "post"
tags: ["kubernetes", "security"]
---

![Credit: growtika](https://images.unsplash.com/photo-1667372459470-5f61c93c6d3f?q=80&w=2664&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D)

#### Background
Kubernetes has become the de facto container orchestration platform, empowering organizations to deploy, scale, and manage
containerized applications seamlessly. However, with great power comes great responsibility, especially when it comes to security.
One common pitfall in Kubernetes manifests and Helm charts is the presence of hard-coded passwords, which can expose
your applications to serious security risks.

#### The Problem
The Problem is that hard-coded passwords in manifests and Helm charts are a potential security vulnerability.
These credentials are often stored in plaintext, making it easier for unauthorized users or malicious actors to access
sensitive information and compromise the security of your Kubernetes clusters.

Let's take a look at an example of hard-coded passwords in a Kubernetes manifest:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  username: YWRtaW4=  # base64-encoded 'admin'
  password: cGFzc3dvcmQ=  # base64-encoded 'password123'
```

In this example, the username and password for a database are encoded in base64, but they are still easily decipherable.
This practice poses a security risk if the manifest is exposed or inadvertently shared.

#### Best Practices to Avoid Hard-Coded Passwords
##### 1. Use Kubernetes Secrets
Leverage Kubernetes Secrets to store sensitive information securely. This ensures that credentials are not exposed in
plain text in the manifest files. You can get inspired from this code snippet:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  username: <base64-encoded-username>
  password: <base64-encoded-password>
```

##### 2. Externalize Configuration:
Keep sensitive information, including passwords, out of manifests and Helm charts. Instead, use configuration files or
environment variables to provide such information during deployment.

##### 3. Utilize Helm Values:
Parameterize your Helm charts using values, allowing users to input sensitive information during deployment.
This separates configuration from the chart, promoting re-usability and reducing security risks.
```yaml
# values.yaml
database:
  username: myuser
  password: mypassword
```
```yaml
# values in deployment.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  username: {{ .Values.database.username | b64enc | quote }}
  password: {{ .Values.database.password | b64enc | quote }}
```

##### 4. Encrypted Secrets:
Consider using tools like Sealed Secrets or external solutions to encrypt sensitive information within Kubernetes Secrets.
This adds an extra layer of security, especially when managing secrets in version control systems.


That is all I know about the problem and the techniques to mitigate the threads against it, up to now.  
