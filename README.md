# Ansible Role: PgBouncer

An Ansible role to deploy and manage a containerized instance of [PgBouncer](https://www.pgbouncer.org/) - a lightweight connection pooler for PostgreSQL.

## Table of Contents

- [About PgBouncer](#about-pgbouncer)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Supported Platforms](#supported-platforms)
- [Role Variables](#role-variables)
- [Dependencies](#dependencies)
- [Example Playbook](#example-playbook)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Author Information](#author-information)

---

## About PgBouncer

PgBouncer is a lightweight connection pooler for PostgreSQL that sits between your application and the PostgreSQL database server. It reduces the performance overhead of opening new connections to PostgreSQL by maintaining a pool of reusable connections.

### Why Use PgBouncer?

- **Reduce connection overhead**: PostgreSQL creates a new process for each connection, which can be resource-intensive
- **Handle connection spikes**: Allow many clients to connect while maintaining a smaller pool of actual database connections
- **Improve performance**: Reuse existing connections instead of creating new ones
- **Resource management**: Limit the number of connections to your database server

This role deploys PgBouncer as a Docker container using the `edoburu/pgbouncer` image, making it easy to integrate into containerized environments.

---

## Features

- Containerized deployment using Docker
- Configurable connection pooling modes (session, transaction, statement)
- Support for multiple databases and users
- Flexible authentication methods (md5, scram-sha-256, plain, trust)
- Customizable pool sizes and connection limits
- Easy integration with existing PostgreSQL deployments
- Automated testing with Molecule

---

## Requirements

- **Ansible**: `2.1` or higher (tested with `2.14.3`)
- **Docker**: Installed and running on target hosts
- **Ansible Collections**:
  - `community.docker` - Install with: `ansible-galaxy collection install community.docker`

---

## Installation

### Install from Ansible Galaxy

```bash
ansible-galaxy install ziyad-ops.pgbouncer
```

### Install from Git Repository

```bash
# Using requirements.yml
cat << EOF > requirements.yml
- src: https://github.com/ziyad-ops/ansible-role-pgbouncer.git
  name: pgbouncer
  version: main
EOF

ansible-galaxy install -r requirements.yml
```

### Install Manually

```bash
git clone https://github.com/ziyad-ops/ansible-role-pgbouncer.git roles/pgbouncer
```

---

## Supported Platforms

This role has been tested on:

- **Debian**: 11 (Bullseye), 12 (Bookworm)
- **Ubuntu**: 22.04 (Jammy)

---

## Role Variables

### **1. Docker Configuration**

This section defines variables related to Docker container configuration.

| Variable                  | Description                                                        | Default Value       |
|---------------------------|--------------------------------------------------------------------|---------------------|
| `pgbouncer_docker_image`  | The Docker image for PgBouncer.                                    | `edoburu/pgbouncer` |
| `pgbouncer_docker_tag`    | The image tag/version.                                             | `v1.24.1-p1`        |
| `pgbouncer_container_name` | The name of the PgBouncer container.                              | `pgbouncer`         |
| `pgbouncer_restart_policy` | Container restart policy.                                         | `always`            |
| `pgbouncer_host_port`     | Host port to expose PgBouncer.                                     | `6432`              |
| `pgbouncer_listen_port`   | Internal container listen port.                                    | `6432`              |

---

### **2. User and Permissions**

| Variable             | Description                                 | Default Value |
|----------------------|---------------------------------------------|---------------|
| `pgbouncer_user`     | The system user for managing config files.  | `postgres`    |
| `pgbouncer_uid`      | User ID for the PgBouncer user.             | `70`          |
| `pgbouncer_group`    | The system group for PgBouncer files.       | `postgres`    |
| `pgbouncer_gid`      | Group ID for the PgBouncer group.           | `70`          |

---

### **3. Configuration Paths**

| Variable                   | Description                                           | Default Value                           |
|----------------------------|-------------------------------------------------------|-----------------------------------------|
| `pgbouncer_home_path`      | Base directory for PgBouncer configuration.           | `/opt/pgbouncer`                        |
| `pgbouncer_config_path`    | Configuration directory for PgBouncer files.          | `/opt/pgbouncer/{{ container_name }}`   |

**Note**: Log directory configuration is available but not actively used in this role's default configuration.

---

### **4. PgBouncer Settings**

| Variable                     | Description                                                  | Default Value    |
|------------------------------|--------------------------------------------------------------|------------------|
| `pgbouncer_pool_mode`        | Pooling mode (`session`, `statement`, `transaction`).        | `transaction`    |
| `pgbouncer_max_client_conn`  | Maximum client connections.                                  | `200`            |
| `pgbouncer_default_pool_size`| Default pool size per database.                              | `30`             |
| `pgbouncer_auth_type`        | Authentication type (`md5`, `scram-sha-256`, `plain`, `trust`). | `scram-sha-256` |
| `pgbouncer_admin_users`      | Admin users for PgBouncer management console.                | `pgbouncer`      |
| `pgbouncer_stats_users`      | Users allowed to access statistics.                          | `pgbouncer_monitor` |

#### Pool Mode Explanation

- **session**: One server connection per client connection (most compatible)
- **transaction**: Server connection returned to pool after transaction (recommended for most use cases)
- **statement**: Server connection returned to pool after each statement (most aggressive, requires no transactions)

---

### **5. Authentication Configuration**

| Variable                  | Description                                           | Default Value                                    |
|---------------------------|-------------------------------------------------------|--------------------------------------------------|
| `pgbouncer_auth_query`    | SQL query to fetch authentication credentials.        | `SELECT username, password FROM pgbouncer.get_auth($1)` |
| `pgbouncer_auth_user`     | User for authentication queries.                      | `pgbouncer_auth`                                 |
| `pgbouncer_auth_dbname`   | Database for authentication queries.                  | `postgres`                                       |

---

### **6. Database Configuration**

| Variable             | Description                                     | Default Value |
|----------------------|-------------------------------------------------|---------------|
| `pgbouncer_databases`| List of databases to expose through PgBouncer.  | See below     |

**Default Configuration:**
```yaml
pgbouncer_databases:
  - name: '*'  # Wildcard: allows connections to any database
    host: "{{ ansible_default_ipv4.address }}"
    port: 5432
```

**Custom Configuration Example:**
```yaml
pgbouncer_databases:
  - name: db1
    host: postgres1.example.com
    port: 5432
    dbname: production_db1

  - name: db2
    host: postgres2.example.com
    port: 5432
    dbname: production_db2
```

---

### **7. User Authentication**

| Variable           | Description                              | Default Value |
|--------------------|------------------------------------------|---------------|
| `pgbouncer_users`  | List of users for PgBouncer authentication. | See below  |

**Default Users:**
```yaml
pgbouncer_users:
  - username: postgres          # Admin console user
    password: admin

  - username: pgbouncer_monitor # Monitoring user
    password: Monitor

  - username: pgbouncer_auth    # Auth query user
    password: auth
```

**Custom Configuration Example:**
```yaml
pgbouncer_users:
  - username: app_user
    password: "{{ vault_app_password }}"  # Use Ansible Vault for secrets

  - username: readonly_user
    password: "{{ vault_readonly_password }}"
```

**Security Note**: Always use Ansible Vault or external secret management for passwords in production.

---

## Dependencies

This role has no hard dependencies on other Ansible roles. However, it works well with:

- **PostgreSQL roles** (optional): For deploying PostgreSQL alongside PgBouncer
- **Docker roles** (optional): For ensuring Docker is installed on target hosts

---

## Example Playbook

### Basic Usage

```yaml
- hosts: database_servers
  become: true
  roles:
    - role: ziyad-ops.pgbouncer
```

### Custom Configuration

```yaml
- hosts: database_servers
  become: true
  roles:
    - role: ziyad-ops.pgbouncer
      vars:
        pgbouncer_host_port: 6432
        pgbouncer_pool_mode: transaction
        pgbouncer_max_client_conn: 500
        pgbouncer_default_pool_size: 50

        pgbouncer_databases:
          - name: app_db
            host: postgres-primary.internal
            port: 5432
            dbname: application

        pgbouncer_users:
          - username: app_user
            password: "{{ vault_app_password }}"
```

### Multiple Databases

```yaml
- hosts: database_servers
  become: true
  roles:
    - role: ziyad-ops.pgbouncer
      vars:
        pgbouncer_databases:
          - name: db1
            host: postgres1.internal
            port: 5432
            dbname: database1

          - name: db2
            host: postgres2.internal
            port: 5432
            dbname: database2

          - name: db3
            host: postgres3.internal
            port: 5432
            dbname: database3

        pgbouncer_users:
          - username: user1
            password: "{{ vault_user1_pass }}"
          - username: user2
            password: "{{ vault_user2_pass }}"
```

---

## Testing

This role includes comprehensive testing using [Molecule](https://molecule.readthedocs.io/).

### Prerequisites

```bash
# Install testing dependencies
pip install molecule molecule-plugins[docker] ansible-core

# Install required collections
ansible-galaxy collection install community.docker
```

### Run Tests

```bash
# Run complete test suite
molecule test

# Run individual test phases
molecule create    # Create test instance
molecule converge  # Apply the role
molecule verify    # Run verification tests
molecule destroy   # Clean up
```

### Test Scenarios

The Molecule tests validate:
- PgBouncer container deployment
- Configuration file generation
- Container health and running state
- Network connectivity
- Idempotency (role can be run multiple times safely)

For more details, see [molecule/default/README.md](molecule/default/README.md).

---

## Troubleshooting

### Container Connectivity Issues

**Problem**: Cannot connect to PgBouncer

**Solutions**:
1. Verify the PgBouncer container is running:
   ```bash
   docker ps | grep pgbouncer
   ```

2. Check PgBouncer logs:
   ```bash
   docker logs pgbouncer
   ```

3. Ensure both PgBouncer and PostgreSQL containers are on the same Docker network:
   ```bash
   docker network inspect bridge
   ```

### Database Connection Issues

**Problem**: PgBouncer cannot connect to PostgreSQL

**Solutions**:
1. Test connectivity from PgBouncer container:
   ```bash
   docker exec -it pgbouncer psql -h <postgres_host> -p 5432 -U postgres
   ```

2. Verify PostgreSQL allows connections from PgBouncer's IP:
   - Check `pg_hba.conf` on PostgreSQL server
   - Ensure firewall rules allow traffic on port 5432

### Client Connection Testing

Test connection through PgBouncer:
```bash
# From client machine
psql -h <pgbouncer_host> -p 6432 -U <username> -d <database>
```

### Admin Console Access

Connect to PgBouncer admin console:
```bash
psql -h localhost -p 6432 -U postgres pgbouncer

# Useful admin commands:
SHOW POOLS;       # Show pool statistics
SHOW DATABASES;   # Show configured databases
SHOW CLIENTS;     # Show connected clients
SHOW SERVERS;     # Show server connections
RELOAD;           # Reload configuration
```

### Performance Issues

If experiencing slow connections:
1. Increase `pgbouncer_default_pool_size`
2. Adjust `pgbouncer_max_client_conn` based on your workload
3. Consider changing `pgbouncer_pool_mode` to `transaction` for better connection reuse
4. Monitor pool statistics with `SHOW POOLS;` in admin console

---

## Contributing

Contributions are welcome! Please follow these guidelines:

1. **Fork** the repository
2. **Create a feature branch**: `git checkout -b feature/your-feature-name`
3. **Make your changes** and ensure they follow the existing code style
4. **Run tests**: `molecule test` - All tests must pass
5. **Run linting**: `ansible-lint`
6. **Commit your changes**: Use clear, descriptive commit messages
7. **Push to your fork**: `git push origin feature/your-feature-name`
8. **Submit a Pull Request**

### Testing Requirements

- All changes must pass Molecule tests
- Role must remain idempotent
- Follow Ansible best practices

---

## License

This project is licensed under the **MIT License** - see the LICENSE file for details.

---

## Author Information

This role was created by **ziyad-ops** 

### Contact & Support

- **Issues**: Report bugs or request features via the issue tracker
- **Pull Requests**: Contributions are welcome!

### Related Projects

- [PgBouncer Official Documentation](https://www.pgbouncer.org/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Docker Community Collection](https://docs.ansible.com/ansible/latest/collections/community/docker/)
