# ansible-datadog — RepoDocs
_Generated on 2026-05-11_

## Summary

### Overview
`ansible-datadog` is an Ansible role that installs and configures the Datadog monitoring Agent (versions 5 and 6) and its checks on Linux (Debian, RHEL/CentOS, SUSE/SLES) and Windows hosts. It is a fork of the upstream `DataDog/ansible-datadog` Galaxy role (Apache 2.0), vendored into the org so playbooks can reference it locally without pulling from Ansible Galaxy. Its role in the org is to provide a reusable Datadog-agent provisioning building block consumed by other Ansible playbook repos.

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | YAML (Ansible) | min_ansible_version 2.5 |
| Language | Jinja2 (templates) | (Ansible-bundled) |
| Framework | Ansible role | 3.1.0 (per CHANGELOG) |
| Build Tool | ansible-lint | _Not pinned_ |
| CI/CD | CircleCI | config v2 |
| CI/CD | GitHub Actions | scheduled secrets scan |
| Cloud/Infra | Datadog Agent (5 & 6, Windows MSI from S3) | n/a |

### Consumers
| Consumer | Type | How They Use It |
|----------|------|----------------|
| CircleCI | CI | Runs the role under ansible-lint and end-to-end install/upgrade/downgrade jobs across Debian and CentOS images (`.circleci/config.yml`). |
| GitHub Actions | CI | Scheduled TruffleHog secrets scan and Slack notifier (`.github/workflows/secrets-scan.yml`). |
| Ansible playbook repos in this org (e.g. `ansible-playbooks`, `devops-playbooks`, `sfdevops-aws-stacks`, `legacy-puppetmaster` consumers) | Ansible role caller | Include `Datadog.datadog` / this role to install the Datadog agent on target hosts. (Inferred from the role's purpose; no direct in-repo reference to specific sibling repos.) |
| Datadog APT/YUM/Zypper repositories (`apt.datadoghq.com`, `yum.datadoghq.com`) | External package source | Agent packages and GPG keys are pulled from these endpoints. |
| `s3.amazonaws.com/ddagent-windows-stable` | External package source | Windows MSI installer download URL. |
| `keyserver.ubuntu.com`, `pool.sks-keyservers.net` | External key server | Pulls Datadog APT GPG key (with backup keyserver fallback). |

### Dependencies on Org Repos
| Repo | Reason |
|------|--------|
_None — `meta/main.yml` declares `dependencies: []`. The role has no in-repo references to other org repos._

### External Integrations
| Service | Purpose | Integration Type |
|---------|---------|-----------------|
| Datadog (agent, APT/YUM/Zypper repos, Windows MSI on S3) | Install and configure the Datadog monitoring Agent and its checks | SDK (package install + agent config) |
| Slack (`rtCamp/action-slack-notify`) | Notify `github-token-scan` channel when secrets are detected | Webhook (outbound) |
| TruffleHog GitHub Action (`edplato/trufflehog-actions-scan`) | Scheduled secrets scanning | SDK (GitHub Action) |
| Ubuntu keyserver / SKS keyserver pool | Fetch Datadog APT GPG key | REST (HKP) |

### Async & Scheduled Work
| Channel / Job | Type | Direction | Purpose |
|--------------|------|-----------|---------|
| `secret-scan` GitHub Actions workflow (cron `0 14 * * 1-5`) | Scheduled CI job | N/A | TruffleHog secrets scan weekdays at 14:00 UTC; Slack notify on failure. |
| `datadog-agent` / `datadog-agent-process` / `datadog-agent-trace` (systemd) services managed by role | Background daemons (managed) | N/A | Long-running Datadog Agent processes started/stopped by the role on the target host. |

### Upgrade Alerts
| Dependency | Current Version | Issue | Severity |
|-----------|----------------|-------|----------|
| Ansible | min 2.5 (test matrix 2.5–2.8) | All Ansible 2.x releases are EOL (Ansible 2.9 was the final 2.x; community Ansible has moved to 4.x/2.10+ collections). | Severe |
| Datadog Agent 5 | supported by role | Datadog Agent 5 is end-of-life; Datadog dropped support years ago. | Severe |
| Target OS platforms | Ubuntu trusty/xenial/artful, Debian wheezy/jessie/stretch, EL 6/7, openSUSE 11/12, Windows 2008 | Most platforms listed in `meta/main.yml` are EOL (trusty, xenial, wheezy, jessie, stretch, EL6, openSUSE 11/12, Windows 2008). | Severe |
| `actions/checkout@master`, `edplato/trufflehog-actions-scan@master` (GitHub Actions) | floating `@master` refs | Floating refs to third-party actions = supply-chain risk; the action publisher could push malicious code. | Critical |
| CircleCI Docker images `datadog/docker-library:ansible_*_2_5..2_8` | unpinned tags on EOL Ansible | Built on EOL Ansible versions; images themselves unmaintained. | Severe |

## API Reference

This is an Ansible role, not a service. Its "public API" is the set of role variables, tasks, handlers, and the configuration-file schema it manages.

### Role Variables (consumer-facing inputs)

Defined in `defaults/main.yml` and `meta/main.yml`. Override at playbook/inventory level.

| Variable | Default | Purpose |
|---|---|---|
| `datadog_api_key` | _(unset; falls back to `'youshouldsetthis'` in template)_ | Datadog API key written to `datadog.yaml`. |
| `datadog_site` | _(unset)_ | Datadog intake site (`datadoghq.com` / `datadoghq.eu`); requires Agent ≥ 6.6.0. |
| `datadog_agent_version` | `""` | Pin a specific Agent version. Empty = `state: latest`. |
| `datadog_agent5` | `no` | Install Agent 5 instead of Agent 6 (Linux only). |
| `datadog_agent_allow_downgrade` | `no` | Allow downgrade (apt `force`, yum `allow_downgrade`, zypper `oldpackage`). |
| `datadog_enabled` | `yes` | Whether `datadog-agent` service should run. |
| `datadog_skip_running_check` | `false` | Skip the service start/stop step (for sysvinit users). |
| `datadog_config` | `{}` | Map merged into the main config file. |
| `datadog_config_ex` | _(unset)_ | Extra INI sections for `datadog.conf` (Agent 5 only). |
| `datadog_checks` | `{}` | Map of `check_name -> check YAML`, expanded to per-check config files. |
| `datadog_user` / `datadog_group` | `dd-agent` / `root` | File ownership for config files. |
| `datadog_additional_groups` | `{}` | Additional Linux groups for `datadog_user`. |
| `datadog_apt_repo` | `deb https://apt.datadoghq.com/ stable 6` | APT repo line. |
| `datadog_apt_cache_valid_time` | `3600` | APT cache TTL (seconds). |
| `datadog_apt_key_retries` | `5` | Retries when fetching APT key. |
| `datadog_apt_keyserver` / `datadog_apt_backup_keyserver` | `hkp://keyserver.ubuntu.com:80` / `hkp://pool.sks-keyservers.net:80` | Keyservers. |
| `use_apt_backup_keyserver` | `false` | Force use of backup keyserver. |
| `datadog_apt_key_url_new` | _(unset)_ | Override APT GPG key URL (key ID `382E94DE`). |
| `datadog_yum_repo` | `https://yum.datadoghq.com/stable/6/{{ ansible_userspace_architecture }}/` | YUM baseurl. |
| `datadog_yum_gpgkey` | `https://yum.datadoghq.com/DATADOG_RPM_KEY.public` | YUM GPG key URL (key ID `4172A230`). |
| `datadog_yum_gpgkey_e09422b3` | `…DATADOG_RPM_KEY_E09422B3.public` | YUM GPG key for Agent 6 ≥ 6.14. |
| `datadog_yum_gpgkey_e09422b3_sha256sum` | `694a2ffe…0c812085` | Pinned checksum of the above key. |
| `datadog_zypper_repo` | `https://yum.datadoghq.com/suse/stable/6/{{ ansible_userspace_architecture }}` | Zypper baseurl. |
| `datadog_zypper_gpgkey` / `_sha256sum` | `…DATADOG_RPM_KEY.public` / `00d6505c…` | Zypper GPG key & checksum. |
| `datadog_zypper_gpgkey_e09422b3` / `_sha256sum` | `…E09422B3.public` / `694a2ffe…` | Zypper GPG key (Agent 6.14+). |
| `datadog_agent5_apt_repo` / `_yum_repo` / `_zypper_repo` | Agent 5 repo URLs | Used when `datadog_agent5: yes`. |
| `datadog_windows_download_url` | `https://s3.amazonaws.com/ddagent-windows-stable/datadog-agent-6-latest.amd64.msi` | MSI download URL. |
| `datadog_windows_versioned_url` | `https://s3.amazonaws.com/ddagent-windows-stable/ddagent-cli` | Versioned MSI URL prefix. |
| `datadog_windows_ddagentuser_name` / `_password` | `""` / `""` | Windows service user credentials. |
| `win_install_args` | `" "` | MSI install args (set by `pkg-windows-opts.yml`). |

### Role Tasks (entry points)

`tasks/main.yml` dispatches by OS family:

| File | Trigger | Purpose |
|---|---|---|
| `pkg-debian.yml` | `ansible_os_family == "Debian"` | Install `apt-transport-https`, import GPG key (id `A2923DFF56EDA6E76E55E492D3A80E30382E94DE`), remove deprecated HTTP repos, add Datadog APT repo, install `datadog-agent` (pinned or latest). |
| `pkg-redhat.yml` | `ansible_os_family == "RedHat"` | Download/import RPM key `E09422B3` with sha256 verify, install `datadog` (or `datadog_5`) yum repo, install agent. |
| `pkg-suse.yml` | `ansible_os_family == "Suse"` | Download/import both RPM keys (`4172A230`, `E09422B3`) with SLES11 SNI workaround, install zypper repo via templated `/etc/zypp/repos.d/datadog.repo`, install agent. |
| `pkg-windows.yml` | `ansible_os_family == "Windows"` | Fail if Agent 5, choose download URL (latest or versioned), download MSI to `%TEMP%`, install via `win_package`, delete MSI. |
| `agent5-linux.yml` | `datadog_agent5 and not Windows` | Create `/etc/dd-agent`, render `datadog.conf` (INI), per-check `*.yaml`, manage `datadog-agent` service. |
| `agent6-linux.yml` | `not datadog_agent5 and not Windows` | Add user groups, create `/etc/datadog-agent`, render `datadog.yaml`, per-check `conf.d/<name>.d/conf.yaml`, render `trace-agent.conf`/`process-agent.conf`, manage `datadog-agent` + `…-process` + `…-trace` services. |
| `agent6-win.yml` | `not datadog_agent5 and Windows` | Render the same set of files under `%ProgramData%\Datadog\`, manage the `datadogagent` Windows service. |
| `win_agent_6_latest.yml` / `win_agent_version.yml` | Helpers | Set `dd_download_url` fact. |
| `pkg-windows-opts.yml` | Helper | Build the `win_install_args` string from optional user/password vars. |
| `post_tasks/*.yml`, `pre_tasks/*.yml` | If `post_tasks` / `pre_tasks` defined | User-supplied hook directories (3.1.0 feature). |

### Handlers (`handlers/main.yml`)

| Handler | Action |
|---|---|
| `restart datadog-agent` | `service: name=datadog-agent state=restarted` (Linux, when enabled, not check-mode). |
| `restart datadog-agent-win` | `win_service: name=datadogagent state=restarted force_dependent_services=true` (Windows). |

### Templates (`templates/`)

| File | Renders to | Content |
|---|---|---|
| `datadog.yaml.j2` | `/etc/datadog-agent/datadog.yaml` (Linux Agent 6) or `%ProgramData%\Datadog\datadog.yaml` (Windows) | `site`, `dd_url`, `api_key`, plus full `datadog_config` via `to_nice_yaml`. |
| `datadog.conf.j2` | `/etc/dd-agent/datadog.conf` (Agent 5 INI), also reused for `trace-agent.conf` / `process-agent.conf` | INI-format config. |
| `checks.yaml.j2` | per-check config file | `{{ datadog_checks[item] | to_nice_yaml }}`. |
| `zypper.repo.j2` | `/etc/zypp/repos.d/datadog.repo` | Zypper repo file (because Ansible's `zypper_repository` module did not allow setting `repo_gpgcheck`). |

## Architecture

### System-context diagram

```
                       ┌────────────────────────┐
                       │   Ansible Controller   │
                       │  (operator's machine,  │
                       │   CI runner, etc.)     │
                       └───────────┬────────────┘
                                   │ ansible-playbook
                                   ▼
            ┌──────────────────────────────────────────┐
            │   ansible-datadog role (this repo)       │
            │   - tasks/main.yml dispatches by OS      │
            │   - templates render Datadog config      │
            │   - handlers restart service             │
            └─────────┬────────────────────────┬───────┘
                      │                        │
            (Linux: apt/yum/zypper)   (Windows: win_get_url + MSI)
                      │                        │
                      ▼                        ▼
        ┌──────────────────────────┐  ┌──────────────────────────┐
        │ apt.datadoghq.com        │  │ s3.amazonaws.com/        │
        │ yum.datadoghq.com        │  │   ddagent-windows-stable │
        │ keyserver.ubuntu.com     │  │                          │
        └──────────────────────────┘  └──────────────────────────┘
                      │                        │
                      ▼                        ▼
            ┌──────────────────────────────────────────┐
            │     Managed Host (target server/VM)      │
            │  /etc/datadog-agent/  or  C:\ProgramData │
            │   datadog-agent service ─────────────────┼──► Datadog SaaS
            └──────────────────────────────────────────┘
                                                     (datadoghq.com / .eu)

   CI:                                       Scheduled scan:
   CircleCI runs ansible-lint + dry-run +    GitHub Actions cron
   real install in datadog/docker-library    (0 14 * * 1-5)
   containers for Debian/CentOS × Ansible    runs TruffleHog →
   2.5/2.6/2.7/2.8.                          Slack on failure.
```

### Key components

- `tasks/main.yml`: dispatcher that includes the per-OS-family package task file and then the appropriate Agent-version task file (Agent 5 vs Agent 6, Linux vs Windows). Trailing `include_tasks` for optional `pre_tasks/*.yml` and `post_tasks/*.yml` user hooks.
- `pkg-debian.yml` / `pkg-redhat.yml` / `pkg-suse.yml` / `pkg-windows.yml`: per-OS installer logic. Adds Datadog package repos, GPG keys, and installs the agent package.
- `agent5-linux.yml` / `agent6-linux.yml` / `agent6-win.yml`: render configuration files for the chosen Agent version and manage the agent service (start/stop, enable/disable).
- `templates/`: Jinja2 templates that produce `datadog.yaml`, `datadog.conf`, per-check `conf.yaml`, and `datadog.repo`.
- `handlers/main.yml`: agent restart handler for Linux (`service`) and Windows (`win_service`).
- `defaults/main.yml`: declares every overridable variable with safe defaults.
- `meta/main.yml`: Ansible Galaxy metadata (author, supported platforms, `dependencies: []`).

### Data flow

1. Operator/CI invokes a playbook that lists this role with vars (`datadog_api_key`, `datadog_checks`, …).
2. `tasks/main.yml` selects the right include based on `ansible_os_family` and `datadog_agent5`.
3. The OS-specific include adds the Datadog package repo, trusts the GPG key, and installs `datadog-agent` (pinned or latest).
4. The agent-version include writes config files from the templates using the operator-provided maps, then ensures the `datadog-agent` service (or `datadogagent` on Windows) is started/enabled (or stopped if `datadog_enabled: false`).
5. Any template change notifies the restart handler at the end of the play.
6. At runtime the installed agent ships metrics/traces/logs to Datadog SaaS (`datadoghq.com` / `datadoghq.eu`) — outside the role's scope.

### CI/CD tooling

Two tools detected:

- **CircleCI** (`.circleci/config.yml`, version 2). Workflow `test_datadog_role` runs:
  1. `ansible_lint` job — `pip install ansible-lint` then `ansible-lint -v $(find . -name *yml) -x ANSIBLE0006,ANSIBLE0010,602`.
  2. A matrix of `{debian, centos} × ansible {2.5, 2.6, 2.7, 2.8} × agent {5, 6}` jobs (16 jobs). Each job checks out the repo, runs the appropriate playbook in `--check` mode (dry run), then real installs, then upgrade/downgrade scenarios using the `datadog/docker-library:ansible_<distro>_<ver>` images. Verifies via `dd-agent info` / `datadog-agent version` / `ps aux | grep datadog-agent`.
  No deploy stage — this is a library/role, not a service.
- **GitHub Actions** (`.github/workflows/secrets-scan.yml`). Scheduled at `0 14 * * 1-5`. Steps: `actions/checkout@master` → `edplato/trufflehog-actions-scan@master` (`--regex --entropy=False --max_depth=1`) → on failure, `rtCamp/action-slack-notify@v2.0.2` to `github-token-scan` Slack channel.

### Test architecture

- `tests/` — minimal Ansible Galaxy test scaffold: a one-line `Vagrantfile` (Ubuntu trusty), an `inventory`, and four playbooks (`test_5_default.yml`, `test_5_full.yml`, `test_6_default.yml`, `test_6_full.yml`) that invoke the role with default or full variable sets.
- `ci_test/` — playbooks used by CircleCI: `install_agent_5.yaml`, `install_agent_6.yaml`, `downgrade_to_5_centos.yaml`, `downgrade_to_5_debian.yaml`, plus a `ci.ini` inventory. These exercise install/upgrade/downgrade flows against the role using `/root/project/` as the role path inside the CI container.
- No unit tests (Molecule, testinfra) — verification is purely end-to-end via `ansible-playbook` runs + post-run shell assertions (`dd-agent info`, `ps aux | grep datadog-agent`).

### Data model / database schema

Not applicable — no database. The role's configuration "schema" is the operator-supplied dict structure (`datadog_config`, `datadog_checks`, `datadog_config_ex`) plus the on-disk Datadog agent config layout (`/etc/datadog-agent/datadog.yaml`, `conf.d/<name>.d/conf.yaml`, etc.).

### Auth & trust boundaries

- Inbound auth: _Not applicable._ This is an Ansible role; it has no listening surface. Trust is whatever the calling playbook establishes (SSH/WinRM).
- Outbound auth:
  - APT GPG key id `A2923DFF56EDA6E76E55E492D3A80E30382E94DE` (key ID `382E94DE`) trusted via `apt_key`.
  - YUM/Zypper RPM keys `4172A230` and `E09422B3` trusted via `rpm_key`; the `E09422B3` key download is checksum-verified with a pinned sha256.
  - The Datadog API key (`datadog_api_key`) is written to `datadog.yaml` so the installed agent can authenticate to Datadog SaaS at runtime.
  - GitHub Actions uses `secrets.ACCESS_TOKEN` for TruffleHog and `secrets.SLACK_WEBHOOK` for notifications.
- Authorization model: _Not applicable._ The role runs with whatever privileges the calling playbook grants (`become: yes` is recommended in README examples).

### Data ownership

_Not applicable._ This role does not own any datastore. It configures an agent that emits telemetry to Datadog SaaS.

### Deployment topology

_Deployment topology not in this repo._ This is an Ansible role consumed by other playbooks; it does not deploy itself. The role itself runs on whatever Ansible controller invokes it and targets whatever hosts the calling inventory specifies.

## Repo Activity

Derived from git history; current HEAD is `89e0499`.

- **Created**: 2014-06-10 (initial commits dated 2014-06-10, with bulk of original activity through 2014-07).
- **Last meaningful change**: 2019-08-30 — `4cc9a1d` "Release 3.1.0", which bundled SUSE RPM-key rotation, Windows `ddagentuser` name/password support, and `pre_tasks`/`post_tasks` user hooks. The only commits since then are 2020-07-07 `1a2e2d6` "Create secrets-scan.yml" and 2020-07-08 `89e0499` "Update secrets-scan.yml".
- **Activity level**: 0 commits in the last 90 days (none since 2020-07-08). The repo is effectively unmaintained on this fork.
- **Hot spots** (top files by all-time churn — used as a proxy since no commits in the last 6 months):
  - `README.md` (52 commits)
  - `defaults/main.yml` (32 commits)
  - `tasks/pkg-debian.yml` (24 commits) / `tasks/main.yml` (24 commits)
- **Recent major changes**: _No major changes in the last 6 months._ (No commits at all in the last 6 months; the last substantive functional change predates that window by several years.)
