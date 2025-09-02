# DevOps Architecture

This repo demonstrates our CI/CD architecture for the Python app.

<p align="center">
  <!-- Display SVG (sharp & scalable) -->
  <a href="./diagrams/dynamic devops_architecture_final.drawio">
    <img src="./diagrams/devops_architecturl.svg" alt="DevOps Architecture" width="900"/>
  </a>
</p>

**Flow:** Developer → GitHub → Jenkins (on Azure VM) → Build Docker → JFrog / Docker Hub → Unit tests & SonarCloud → Kubernetes → Ingress (NGINX) → Incoming Traffic → Services/Pods

---

### Edit the diagram
- Open the file `diagrams/devops_architecture.drawio` in [diagrams.net](https://app.diagrams.net/).  
- Or click the diagram above (links to the `.drawio` file in the repo) and choose **Device** to import.

