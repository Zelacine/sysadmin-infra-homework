#  Nginx + PHP-FPM, Terraform, Ansible

This repository is a **template** for completing the test assignment. Full description is in [INSTRUCTIONS.md](./INSTRUCTIONS.md).

## What's already included
- Directory structure for Terraform/Ansible.
- Basic GitHub Actions workflows: `terraform.yml`, `ansible.yml`.
- Template files for `web` role (Ansible) and minimal Terraform configs.

## What the candidate needs to implement (briefly)
1. **Terraform**: describe `nginx` and `php-fpm` containers (docker provider), network and volume, variables and outputs.
2. **Ansible**: `web` role should deploy Nginx config, index.php, enable `nginx` and `php-fpm`, add logrotate.
3. Clean up and complete workflows (fmt/validate/plan + ansible-lint), optionally add Molecule tests for the role.
4. **README**: complete the steps below for running and testing (see "Local Run" section).

---

## Local Run
```
# example
cd /terraform
terraform init
terraform fmt -check
terraform plan 
terraform apply -auto-approve
```

```
cd /ansible
```
**Missing Docker modules**
   - 'community.docker.docker_container_copy' and the other Docker modules come from the 'community.docker' collection. Install it once on your control machine:
```bash   
     ansible-galaxy collection install community.docker
```    
Add a tiny inventory and point the playbook at it, e.g. create ansible/inventory containing:
```bash
     [local]
     localhost ansible_connection=local
```
```bash
     Then run 
     ansible-playbook -i inventory --syntax-check playbook.yml
```
  
   
(No sudo needed unless your Ansible is installed system wide.) If you prefer project-local collections, run that command inside the project and set `ANSIBLE_COLLECTIONS_PATHS` or add to `ansible.cfg`.

Run ansible-playbook:
```bash
ansible-playbook -i inventory --syntax-check playbook.yml
```
and the syntax check should pass.


### Testing
```
curl http://localhost:8080/healthz
# expected JSON:
# {"status":"ok","service":"nginx","env":"dev"}
```

## CI/CD
- **Actions** tab should be green: Terraform (fmt/validate/plan) and ansible-lint pass.
- Attach screenshots or links to successful runs:

Terraform
https://github.com/Zelacine/sysadmin-infra-homework/actions/runs/19681930283/job/56377821675


Ansible
https://github.com/Zelacine/sysadmin-infra-homework/actions/runs/19681930324/job/56377822001

## Useful Links
- Full task description: [INSTRUCTIONS.md](./INSTRUCTIONS.md)
- Solution rationale: See Answer [Decisions.md](./Decisions.md) 
