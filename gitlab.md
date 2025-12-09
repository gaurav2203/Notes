# 🚀 GITLAB CI/CD — INTERVIEW CHEAT SHEET
*(For DevOps Roles Working With GitLab Pipelines)*

## 🔵 1. Pipeline Structure

A GitLab pipeline is built from:

### **Stages**
* Define execution order.
* **Example:** `build` → `test` → `deploy`

### **Jobs**
* Atomic execution units inside stages.
* Jobs in the **same stage** run in **parallel**.
* The next stage runs only when **all jobs in the previous stage succeed**.
