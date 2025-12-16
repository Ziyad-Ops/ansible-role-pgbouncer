# Molecule Testing for PgBouncer Ansible Role

This directory contains [Molecule](https://molecule.readthedocs.io/) test scenarios for the `ansible-role-pgbouncer` role. Molecule is a testing framework designed to help develop and test Ansible roles.

## What is Molecule?

Molecule provides a comprehensive testing framework for Ansible roles, allowing you to:
- Test your role against multiple platforms and scenarios
- Verify idempotence (running the role multiple times produces the same result)
- Validate the final state of the system after role execution
- Automate the entire test lifecycle

## Prerequisites

Before running Molecule tests, ensure you have the following installed:

1. **Python 3.8+**
2. **Docker** - Required for running test containers
3. **Molecule and dependencies**:
   ```bash
   pip install molecule molecule-plugins[docker] ansible-core
   ```
4. **Ansible Collections**:
   ```bash
   ansible-galaxy collection install community.docker
   ```

## Test Infrastructure

This scenario uses Docker-in-Docker (DinD) to test the PgBouncer role:

- **Driver**: Docker
- **Test Platform**: `ziayd/ansible-molecule:v1.0.0` (custom image with Docker support)
- **Test Database**: PostgreSQL 15 container
- **Privileges**: Runs in privileged mode to support Docker-in-Docker

## Running Tests

### Run Complete Test Sequence

To run the full test suite (destroy, create, converge, idempotence, verify, destroy):

```bash
molecule test
```

### Run Individual Test Phases

You can run specific phases of the test sequence:

```bash
# Create the test instance
molecule create

# Run the role (converge)
molecule converge

# Verify the results
molecule verify

# Test idempotence (ensure role doesn't make unnecessary changes on second run)
molecule idempotence

# Destroy the test instance
molecule destroy
```

### Login to Test Instance

To manually inspect the test environment:

```bash
molecule login
```

## Test Sequence Explained

The test sequence defined in `molecule.yml` consists of:

1. **Destroy** - Removes any existing test instances
2. **Create** - Spins up the test container using `create.yml`
3. **Converge** - Runs the PgBouncer role against the test instance using `converge.yml`
4. **Idempotence** - Runs the role again to ensure no changes are made (validates role stability)
5. **Verify** - Executes verification tests defined in `verify.yml`
6. **Destroy** - Cleans up test instances

## Test Files Overview

| File | Purpose |
|------|---------|
| `molecule.yml` | Main configuration file defining the test scenario, driver, platforms, and test sequence |
| `create.yml` | Playbook to create the test infrastructure |
| `converge.yml` | Playbook that applies the role to the test instance and sets up PostgreSQL for testing |
| `verify.yml` | Playbook containing assertions to validate the role worked correctly |
| `destroy.yml` | Playbook to tear down the test infrastructure |

## What Gets Tested

The verification phase (`verify.yml`) checks:

1. **PgBouncer Container Status**
   - Validates that the PgBouncer container is running
   - Ensures the container is in a healthy state

2. **Configuration Directory**
   - Verifies that `/opt/pgbouncer` directory exists
   - Ensures proper file structure is created

## Test Configuration

The `converge.yml` playbook sets up:

- **PostgreSQL 15** container with:
  - Database: `db1`
  - User: `postgres`
  - Password: `admin`
  - Port: `5432`

- **PgBouncer** role execution with default variables

## Interpreting Test Results

### Successful Test Output

```
PLAY RECAP *********************************************************************
molecule                   : ok=X    changed=0    unreachable=0    failed=0
```

### Failed Tests

If tests fail, Molecule will show detailed error messages. Common issues:

- Container creation failures
- Role execution errors
- Verification assertion failures
- Idempotence violations (changes on second run)

## Debugging Failed Tests

1. **Keep the test instance alive**:
   ```bash
   molecule converge
   # Instance stays running for inspection
   ```

2. **Login and investigate**:
   ```bash
   molecule login
   # Check container status
   docker ps
   # Check logs
   docker logs pgbouncer
   # Inspect configuration
   cat /opt/pgbouncer/config/pgbouncer.ini
   ```

3. **Check Molecule logs**:
   ```bash
   molecule --debug test
   ```

4. **Cleanup when done**:
   ```bash
   molecule destroy
   ```

## Customizing Tests

To modify the test scenario:

1. **Change test platform**: Edit `platforms` section in `molecule.yml`
2. **Add more databases**: Modify `converge.yml` to include additional PostgreSQL instances
3. **Add custom variables**: Update `converge.yml` with role variables for different test scenarios
4. **Extend verification**: Add more assertions in `verify.yml`

## CI/CD Integration

This Molecule scenario can be integrated into CI/CD pipelines:

### GitHub Actions Example

```yaml
name: Molecule Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install dependencies
        run: |
          pip install molecule molecule-plugins[docker] ansible-core
          ansible-galaxy collection install community.docker
      - name: Run Molecule tests
        run: molecule test
```

## Additional Resources

- [Molecule Documentation](https://molecule.readthedocs.io/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Docker Documentation](https://docs.docker.com/)
- [PgBouncer Documentation](https://www.pgbouncer.org/)

## Contributing

When contributing to this role, ensure all Molecule tests pass before submitting pull requests:

```bash
molecule test
```

All tests must pass and the role must be idempotent.
