# PumpFun Infrastructure as Code Documentation

## Table of Contents

1. [Introduction](#introduction)
2. [Project Overview](#project-overview)
3. [Architecture](#architecture)
4. [Requirements](#requirements)
5. [Directory Structure](#directory-structure)
6. [Installation and Setup](#installation-and-setup)
7. [Inventory Configuration](#inventory-configuration)
8. [Playbooks](#playbooks)
9. [Roles](#roles)
10. [Security Practices](#security-practices)
11. [Deployment Workflow](#deployment-workflow)
12. [Maintenance and Operations](#maintenance-and-operations)
13. [Troubleshooting](#troubleshooting)
14. [Contributing Guidelines](#contributing-guidelines)
15. [Ethical Considerations](#ethical-considerations)

## Introduction

This documentation provides a comprehensive guide to the Infrastructure as Code (IaC) implementation at PumpFun. Our infrastructure is managed using Ansible, following DevOps best practices to ensure consistency, reliability, and security across all environments.

PumpFun's infrastructure code emphasizes ethical principles in every aspect of our operations. This approach ensures that our technical foundation aligns with our organizational values of transparency, accountability, and responsible technology deployment.

## Project Overview

The PumpFun IaC project uses Ansible to automate the provisioning, configuration, and management of our infrastructure. This automation enables us to:

- Deploy and configure servers consistently and reliably
- Implement security best practices across all systems
- Set up reverse proxy services with Traefik for application routing
- Establish secure user access and management
- Ensure regular system maintenance and updates
- Maintain audit trails of infrastructure changes

Our infrastructure is version-controlled, allowing for transparent history, peer review, and accountability for all changes.

## Architecture

The infrastructure consists of the following components:

1. **Base System Configuration**: Common settings, packages, and security measures applied to all servers
2. **User Management**: Creation and management of administrative users with appropriate permissions
3. **Security Layer**: SSH hardening, firewall configuration, and intrusion prevention
4. **Container Runtime**: Docker installation and configuration for application deployment
5. **Reverse Proxy**: Traefik configuration for routing traffic to applications with TLS termination
6. **Logging and Monitoring**: Docker log rotation and system monitoring

The architecture follows a layered approach, with each layer building upon the previous one to create a complete, secure infrastructure stack.

## Requirements

To use this IaC project, you will need:

- Ansible 2.9 or newer
- Python 3.6 or newer
- Access to target servers (SSH keys configured)
- Vault password for encrypted secrets
- Docker and Docker Compose on the control node (for testing)

Required Ansible collections:

- community.docker
- ansible.posix
- community.general

## Directory Structure

```
├── .gitignore                    # Git ignore rules
├── ansible.cfg                   # Ansible configuration
├── inventories/                  # Environment-specific inventories
│   └── development/              # Development environment
│       ├── group_vars/           # Variables for development groups
│       │   ├── all.yml.example   # Example variables for all hosts
│       │   └── vault.yml.example # Example encrypted variables
│       └── hosts.yml.example     # Example inventory hosts file
├── playbooks/                    # Playbook definitions
│   ├── bootstrap.yml             # Initial system setup
│   ├── deploy.yml                # Application deployment
│   ├── maintenance.yml           # System maintenance tasks
│   └── site.yml                  # Main playbook combining roles
├── requirements.yml              # Required Ansible collections
└── roles/                        # Role definitions
    ├── common/                   # Base system configuration
    ├── root_setup/               # Initial root access configuration
    ├── traefik/                  # Traefik reverse proxy setup
    └── user_setup/               # User configuration and security
```

## Installation and Setup

### Prerequisites

1. Install Ansible on your control node:

   ```bash
   sudo apt update
   sudo apt install ansible python3-pip
   ```

2. Install required Ansible collections:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

3. Create a vault password file:
   ```bash
   echo "your-secure-password" > .vault_pass
   chmod 600 .vault_pass
   ```

### Initial Configuration

1. Copy example inventory files to create your environment:

   ```bash
   cp inventories/development/hosts.yml.example inventories/development/hosts.yml
   cp inventories/development/group_vars/all.yml.example inventories/development/group_vars/all.yml
   ```

2. Create an encrypted vault file:

   ```bash
   ansible-vault create inventories/development/group_vars/vault.yml
   ```

3. Add your server information to the inventory file (hosts.yml)

4. Update variable files with your specific configuration

## Inventory Configuration

### Host Configuration

The inventory defines the servers in your infrastructure. Edit `inventories/development/hosts.yml` to specify your servers:

```yaml
all:
  hosts:
    prod.server.1:
      ansible_host: 203.0.113.10
      ansible_python_interpreter: /usr/bin/python3
```

### Variable Configuration

Edit `inventories/development/group_vars/all.yml` to define environment-specific variables:

```yaml
ansible_user: ansible

system:
  environment: development
  timezone: UTC
  locale: en_US.UTF-8

security:
  ssh:
    permit_root_login: "no"
    password_auth: "no"
  firewall:
    enabled: true
    default_policy: deny
    rules:
      - { port: 22, proto: tcp, rule: allow }
      - { port: 80, proto: tcp, rule: allow }
      - { port: 443, proto: tcp, rule: allow }

docker:
  compose_version: "v2"
  networks:
    - name: traefik-public
      driver: bridge

traefik:
  acme_email: "admin@pumpfun.com"
  image: traefik:v2.9
```

### Vault Configuration

Store sensitive information in the encrypted `vault.yml` file:

```yaml
vault_root_setup_pass: "secure-admin-password"
vault_traefik_dashboard_auth: "admin:hashed-password"
```

To generate a hashed password for Traefik:

```bash
htpasswd -nb admin secure-password
```

## Playbooks

### bootstrap.yml

The bootstrap playbook performs initial system setup with root access:

```yaml
- name: "Bootstrap - Initial System Setup"
  hosts: all
  remote_user: root
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/group_vars/vault.yml"
  tags:
    - bootstrap
  roles:
    - role: root_setup
```

Run with:

```bash
ansible-playbook playbooks/bootstrap.yml -i inventories/development/hosts.yml
```

### site.yml

The main playbook that applies all configuration roles:

```yaml
- name: "Common Setup - Base System Configuration"
  hosts: all
  remote_user: "{{ ansible_user }}"
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/group_vars/vault.yml"
  tags:
    - common
  roles:
    - common

- name: "User Setup - Security Hardening and Service Preparations"
  hosts: all
  remote_user: "{{ ansible_user }}"
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/group_vars/vault.yml"
  tags:
    - user_setup
  roles:
    - user_setup

- name: "Traefik Setup - Reverse Proxy Deployment"
  hosts: all
  remote_user: "{{ ansible_user }}"
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/group_vars/vault.yml"
  tags:
    - traefik
  roles:
    - traefik
```

Run with:

```bash
ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml
```

### deploy.yml

Deploys applications (currently just Traefik):

```yaml
- name: "Deploy Applications"
  hosts: all
  remote_user: "{{ ansible_user }}"
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/group_vars/vault.yml"
  tags:
    - deploy
  roles:
    - traefik
```

Run with:

```bash
ansible-playbook playbooks/deploy.yml -i inventories/development/hosts.yml
```

### maintenance.yml

Performs system maintenance tasks:

```yaml
- name: "System Maintenance"
  hosts: all
  remote_user: "{{ ansible_user }}"
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/group_vars/vault.yml"
  tags:
    - maintenance
  tasks:
    - name: Update and upgrade system packages
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        autoremove: true
      tags:
        - system_update

    # Additional maintenance tasks...
```

Run with:

```bash
ansible-playbook playbooks/maintenance.yml -i inventories/development/hosts.yml
```

## Roles

### common

The common role establishes the base system configuration:

- Installs essential packages
- Sets up system timezone and locale
- Configures unattended security updates
- Applies basic security measures

Key files:

- `roles/common/defaults/main.yml`: Default variables
- `roles/common/tasks/main.yml`: Main tasks
- `roles/common/tasks/packages.yml`: Package installation
- `roles/common/tasks/security.yml`: Security configuration

### root_setup

The root_setup role performs initial system setup:

- Updates the system
- Creates an admin user
- Sets up SSH access
- Configures passwordless sudo

Key files:

- `roles/root_setup/defaults/main.yml`: Default variables
- `roles/root_setup/tasks/main.yml`: Main tasks
- `roles/root_setup/tasks/user_creation.yml`: Admin user creation
- `roles/root_setup/templates/public_key.j2`: SSH key template

### user_setup

The user_setup role handles security hardening and service preparation:

- SSH hardening
- Firewall configuration (UFW)
- Fail2Ban setup
- Docker installation and configuration
- Swap file management

Key files:

- `roles/user_setup/defaults/main.yml`: Default variables
- `roles/user_setup/tasks/main.yml`: Main tasks
- `roles/user_setup/tasks/ssh.yml`: SSH hardening
- `roles/user_setup/tasks/firewall.yml`: Firewall configuration
- `roles/user_setup/tasks/docker.yml`: Docker installation
- `roles/user_setup/tasks/docker_logging.yml`: Docker log configuration
- `roles/user_setup/tasks/swap.yml`: Swap file management

### traefik

The traefik role sets up the Traefik reverse proxy:

- Creates required directories
- Templates docker-compose configuration
- Sets up Docker networks
- Deploys Traefik container
- Configures TLS with Let's Encrypt

Key files:

- `roles/traefik/defaults/main.yml`: Default variables
- `roles/traefik/tasks/main.yml`: Main tasks
- `roles/traefik/tasks/deploy.yml`: Deployment tasks
- `roles/traefik/templates/traefik-docker-compose.yml.j2`: Docker Compose template

## Security Practices

PumpFun's infrastructure implements several security best practices:

### SSH Hardening

- Disabled root login
- Password authentication disabled
- Public key authentication only
- Fail2Ban for brute force protection

### Firewall Configuration

- Default deny policy
- Explicit allow rules for required ports
- UFW for firewall management

### System Security

- Automatic security updates
- Limited user privileges
- Secure password storage with Ansible Vault
- Regular system updates

### Container Security

- Docker log rotation to prevent disk space issues
- Non-root users for container operations
- Minimal exposure of Docker socket

### Secrets Management

- Sensitive data encrypted with Ansible Vault
- No secrets in version control
- Separate vault files for different environments

## Deployment Workflow

The deployment process follows these steps:

1. **Bootstrap**: Initialize new servers with the bootstrap playbook

   ```bash
   ansible-playbook playbooks/bootstrap.yml -i inventories/development/hosts.yml
   ```

2. **Base Configuration**: Apply common configuration and security measures

   ```bash
   ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml --tags=common,user_setup
   ```

3. **Reverse Proxy**: Deploy Traefik for application routing

   ```bash
   ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml --tags=traefik
   ```

4. **Application Deployment**: Deploy individual applications using the established infrastructure

   ```bash
   ansible-playbook playbooks/deploy.yml -i inventories/development/hosts.yml
   ```

5. **Maintenance**: Perform regular maintenance tasks
   ```bash
   ansible-playbook playbooks/maintenance.yml -i inventories/development/hosts.yml
   ```

## Maintenance and Operations

### Regular Maintenance

Schedule regular maintenance to keep systems secure and updated:

```bash
# Update system packages
ansible-playbook playbooks/maintenance.yml -i inventories/development/hosts.yml --tags=system_update

# Check disk space
ansible-playbook playbooks/maintenance.yml -i inventories/development/hosts.yml --tags=check_disk

# Check service status
ansible-playbook playbooks/maintenance.yml -i inventories/development/hosts.yml --tags=check_services
```

### Adding New Servers

To add new servers to the infrastructure:

1. Add the server to the appropriate inventory file
2. Run the bootstrap playbook for initial setup
3. Apply all roles to bring the server to the desired state

### Modifying Configuration

To modify the configuration:

1. Update the relevant variables or tasks
2. Test changes in a development environment
3. Apply changes to production using the appropriate playbook and tags
4. Document changes in version control commit messages

## Troubleshooting

### Common Issues

1. **SSH Connection Failures**

   - Check if the server is accessible
   - Verify SSH key permissions (should be 600)
   - Ensure the correct user is specified in the inventory

2. **Ansible Vault Issues**

   - Verify the vault password file exists and has correct permissions
   - Check if the vault password is correct

3. **Docker Deployment Failures**

   - Check if Docker is installed and running
   - Verify Docker permissions for the ansible user
   - Check network connectivity for image pulls

4. **Traefik Configuration Issues**
   - Verify domain DNS configuration
   - Check if certificates are being issued correctly
   - Inspect Traefik logs for specific errors

### Debugging

Use these techniques to debug issues:

```bash
# Enable verbose output
ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml -vvv

# Check syntax without making changes
ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml --syntax-check

# Run in check mode to see what would change
ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml --check

# Run with a specific tag
ansible-playbook playbooks/site.yml -i inventories/development/hosts.yml --tags=traefik
```

## Contributing Guidelines

### Code Style

- Follow Ansible best practices
- Use YAML syntax consistently (2-space indentation)
- Use descriptive variable and task names
- Include comments for complex tasks

### Version Control

- Create feature branches for new functionality
- Write clear commit messages
- Use pull requests for code review
- Tag releases with semantic versioning

### Documentation

- Update documentation for any changes
- Document variables and their purpose
- Include examples for complex configurations
- Maintain this documentation as code evolves

## Ethical Considerations

PumpFun is committed to ethical practices in all aspects of our operations. Our infrastructure code reflects this commitment through:

### Data Privacy and Security

- Strong encryption for sensitive data
- Minimal collection and storage of user data
- Regular security updates and patching
- Access controls and least privilege principles

### Transparency

- Open documentation of infrastructure practices
- Clear communication about security measures
- Infrastructure as code enables audit and review

### Responsible Resource Usage

- Efficient server utilization
- Log rotation to prevent resource exhaustion
- Responsible scaling practices

### Ethical Decision-Making Framework

When making infrastructure decisions, consider:

1. **Impact**: How does this change affect users, employees, and the environment?
2. **Alternatives**: Have we considered less invasive or resource-intensive options?
3. **Transparency**: Can we clearly explain and justify this decision?
4. **Long-term Consequences**: What are the potential long-term effects of this decision?

By following these ethical considerations, we ensure our technical infrastructure aligns with PumpFun's values and commitment to responsible technology practices.
