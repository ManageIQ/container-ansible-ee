# container-ansible-ee

Container for the ManageIQ Ansible Execution Environment for use by Embedded Ansible.

[![CI](https://github.com/ManageIQ/container-ansible-ee/actions/workflows/ci.yaml/badge.svg?branch=master)](https://github.com/ManageIQ/container-ansible-ee/actions/workflows/ci.yaml)

## Building

```sh
bin/build_container_image
```

This will build a container tagged `docker.io/manageiq/ansible-ee` by default.

To override the tag, set the `TAG` env var:

```sh
TAG=localhost/my-ansible-ee bin/build_container_image
```

By default, this will build using the local architecture. To target a different architecture, use the `ARCH` env var:

```sh
ARCH=amd64 bin/build_container_image
```

## Usage

The container uses `/runner` as the ansible-runner [private data directory](https://ansible-runner.readthedocs.io/en/stable/intro/#runner-input-directory-hierarchy).
Its layout is:

```
/runner/
├── project/    ← playbooks, roles, vars  (mount your payload here)
├── inventory/  ← inventory files
├── env/        ← envvars, extravars, ssh_key, passwords, settings
├── artifacts/  ← job output written at runtime
└── .ansible/   ← Ansible controller scratch space (pre-created in the image)
```

Mount a private data directory (containing a `project/` subdirectory) to `/runner` and invoke
`ansible-runner`:

```sh
docker run --rm -it --platform=linux/amd64 \
  -v /path/to/your/private-data-dir:/runner \
  docker.io/manageiq/ansible-ee:latest \
  ansible-runner run /runner --ident result --playbook playbook.yml
```

The `--playbook` path is relative to `/runner/project`.

Artifacts (job events, stdout, rc, etc.) are written to
`/path/to/your/private-data-dir/artifacts/result/` on the host.

### ansible-galaxy role/collection installation

If your playbook or role requires additional roles from a `requirements.yml`, the caller should
chain `ansible-galaxy install` and `ansible-runner` together with `sh -c` in a single
`docker run`. The image provides `ansible-galaxy` but does not invoke it automatically.

**Playbook with a `requirements.yml` at `project/roles/requirements.yml`:**

```sh
docker run --rm --platform=linux/amd64 \
  -v /path/to/private-data-dir:/runner \
  docker.io/manageiq/ansible-ee:latest \
  sh -c "ansible-galaxy install -r /runner/project/roles/requirements.yml -p /runner/project/roles \
      && ansible-runner run /runner --ident result --playbook playbook.yml"
```

**Running a role directly with a `requirements.yml` at `roles/requirements.yml`:**

```sh
docker run --rm --platform=linux/amd64 \
  -v /path/to/private-data-dir:/runner \
  docker.io/manageiq/ansible-ee:latest \
  sh -c "ansible-galaxy install -r /runner/roles/requirements.yml -p /runner/roles \
      && ansible-runner run /runner --ident result --role my.role --roles-path /runner/roles"
```

## Testing

The `test/data` directory contains test playbooks and supporting files used by the Bats test suite.

To run a quick smoke test manually, copy a playbook into a temp runner directory and exec the EE:

```sh
RUNNER=$(mktemp -d)
mkdir -p "$RUNNER/project"
cp test/data/hello_world.yml "$RUNNER/project/"
docker run --rm --platform=linux/amd64 \
  -v "$RUNNER:/runner" \
  docker.io/manageiq/ansible-ee:latest \
  ansible-runner run /runner --ident result --playbook hello_world.yml
```

To run the Bats test suite:

```sh
# Install bats and plugins (if not already installed)
brew install bats-core
git clone https://github.com/bats-core/bats-support ~/.bats/libs/bats-support
git clone https://github.com/bats-core/bats-assert ~/.bats/libs/bats-assert

# Run tests
bats test/ee_tests.bats
```

To run a specific test:

```sh
bats --filter "runs a playbook" test/ee_tests.bats
```

With ansible-navigator using the execution environment:

```sh
cd test/data
ansible-navigator run hello_world.yml \
  --execution-environment-image docker.io/manageiq/ansible-ee:latest \
  --mode stdout --pull-policy missing
```

## License

This project is available as open source under the terms of the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).
