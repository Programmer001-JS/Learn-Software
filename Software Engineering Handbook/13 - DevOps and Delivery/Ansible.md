# Ansible

> **In one line —** agentless configuration management over SSH, describing the state a machine should be in; still the right tool when you have servers, and increasingly a sign that maybe you should not.

| | |
|---|---|
| **Category** | Configuration Management |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Terraform](Terraform.md) · [Docker](Docker.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) · [Kernel](../02%20-%20Computer%20Science%20Fundamentals/Kernel.md) · [CI CD](CI%20CD.md) |

---

## 1. Short Definition

*What is it?*

Ansible connects to machines over SSH and runs **modules** that enforce a described state: this package installed, this file present with this content, this service running. No agent is installed on the target — Python and SSH are enough.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Terraform created twelve servers
    ↓
Now what is INSIDE them?
    packages, users, config files, services, certificates, log rotation
    ↓
By hand: twelve slightly different machines, and a wiki page nobody trusts
By shell script: not idempotent, no reporting, fails halfway and leaves a mess
    ↓
Configuration drift, and no way to prove what any machine contains
```

> [!IMPORTANT]
> **Ansible's design choice that made it win was being agentless.** Puppet and Chef required an agent, a certificate authority and a central server before you could configure anything. Ansible needed SSH — which you already had. The lowered barrier to entry mattered more than any feature, and it is why Ansible survived while heavier tools contracted.

---

## 3. Architecture Position

```text
   TERRAFORM  ──creates──►  the machines exist
        │
        ▼
   ANSIBLE    ──configures──►  what is inside them
        │
        ▼
   your application runs

   CONTROL NODE (your laptop or CI)
        │  SSH, in parallel
        ├──► web-1   ──┐
        ├──► web-2     ├── inventory groups
        └──► db-1    ──┘
```

> [!TIP]
> **The division of labour is worth stating plainly: Terraform owns the resource, Ansible owns the inside.** Using Terraform to write config files onto instances, or Ansible to create cloud resources, is possible and consistently unpleasant. Each tool is bad at the other's job in a way that only becomes obvious after you have committed to it.

---

## 4. The pieces

```text
INVENTORY     which hosts exist, in which groups (static file or dynamic plugin)
PLAYBOOK      YAML: which tasks run on which groups
TASK          one module invocation
MODULE        the unit of work — apt, copy, template, service, user, ...
ROLE          a reusable, structured bundle of tasks, files, templates, handlers
HANDLER       runs only if a task reported "changed" — e.g. restart nginx
VARS          per host, per group, per role, with a precedence order
TEMPLATE      a Jinja2 file rendered with variables
VAULT         encrypted variable files, safe to commit
```

```yaml
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - common
    - nginx
  tasks:
    - name: Deploy the application config
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/app/app.conf
        owner: app
        mode: "0640"
      notify: Restart app          # the handler runs ONLY if this changed
```

---

## 5. Idempotence is the whole point

```text
DECLARATIVE (correct)                 IMPERATIVE (a shell script in YAML)
- apt: name=nginx state=present       - shell: apt-get install -y nginx
- lineinfile: ...                     - shell: echo "..." >> /etc/config
- service: name=nginx state=started   - shell: systemctl start nginx
    ↓                                     ↓
running twice changes nothing         running twice appends twice
reports changed: 0                    always reports changed
```

> [!CAUTION]
> **Every `shell` or `command` task is a small hole in Ansible's guarantees.** It cannot report whether anything changed, so handlers fire wrongly and `--check` mode becomes meaningless. Sometimes you genuinely need one — then add `creates:`, `removes:` or a `changed_when:` condition so it at least reports honestly. A playbook that always says "changed" is a playbook nobody can read.

---

## 6. Roles, and keeping it maintainable

```text
roles/nginx/
  tasks/main.yml         what to do
  handlers/main.yml      restart triggers
  templates/nginx.conf.j2
  defaults/main.yml      overridable defaults (LOW precedence)
  vars/main.yml          internal values (HIGH precedence)
  meta/main.yml          dependencies
```

```text
GOOD ROLE        one concern, sensible defaults, few required variables
BAD ROLE         conditionals for six operating systems and every possible option
```

> [!TIP]
> **Put overridable settings in `defaults/`, not `vars/`.** Variables in `vars/` sit near the top of the precedence order and are awkward to override from a playbook — which is the opposite of what a reusable role needs. This single distinction accounts for a large share of "why is my variable being ignored" confusion; the precedence list is worth reading once, properly.

---

## 7. Ansible versus immutable infrastructure

```text
MUTABLE (Ansible on live servers)      IMMUTABLE (bake and replace)
change the running machine              build a new image, replace the instance
fast, and works on existing fleets      slower to build, perfectly reproducible
drift accumulates over years            every machine is identical by construction
"why is web-3 different?"               no such question exists
```

```text
THE PRAGMATIC SYNTHESIS
    Ansible builds the IMAGE (with Packer), not the running server
    → declarative configuration, immutable result
```

> [!IMPORTANT]
> **If you are launching new instances anyway, configuring them at boot with Ansible is slower and less reliable than baking an image.** Boot-time configuration means every scale-out event depends on a package repository being available and a playbook succeeding — at exactly the moment you needed capacity. Use Ansible with Packer to produce the image, then launch from it. That keeps Ansible's readable, declarative model and gains immutability.

---

## 8. Where Ansible still clearly wins

```text
STRONG FIT
    existing server fleets that will not be containerised
    network devices, appliances, and anything with only SSH
    on-premises and bare metal
    one-off orchestrated operations across many hosts
    hardening and compliance baselines (CIS benchmarks)
    building images with Packer
    databases and stateful systems you run yourself

WEAK FIT
    anything already containerised
    deploying application code (use CI/CD and immutable artefacts)
    a substitute for Terraform
    a substitute for Kubernetes
```

> [!TIP]
> **Ansible is also excellent as a controlled remote-execution tool, and this is underrated.** "Run this check on all 200 hosts and show me which ones differ" is a genuinely useful capability that has nothing to do with configuration management, and it is far safer than an ad-hoc loop over SSH.

---

## 9. Real World Example

- **Image building with Packer** — the most defensible modern use.
- **On-premises and bare metal**, where cloud-native tooling does not apply.
- **Network and appliance configuration**, where nothing else can be installed.
- **Compliance hardening** — applying and re-verifying a CIS baseline across a fleet.
- **Self-managed database clusters** — PostgreSQL with replication, where a managed service is not an option.
- **Legacy fleets** that will be replaced eventually, and need to be consistent meanwhile.
- **Bootstrapping a Kubernetes cluster** on your own hardware (kubespray is Ansible).

---

## 10. Communication and Dependencies

- **SSH access and a privilege escalation path** (`become`)
- **Python on the target** — the one real requirement
- **An inventory**, ideally dynamic from your cloud provider rather than a stale file
- **Ansible Vault or an external secret store** for sensitive variables
- **[Terraform](Terraform.md)** to create what Ansible then configures
- **Packer**, if you are producing images rather than mutating servers
- **CI**, so playbooks are linted and run from a known state rather than a laptop

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Ansible when you have machines whose interiors you must manage: bare metal, on-premises, network devices, self-managed databases, or golden images built with Packer. Its readability and lack of agents make it the easiest configuration tool to adopt.

> [!CAUTION]
> - **Not for containerised workloads** — a Dockerfile is the configuration
> - **Not for application deployment** — CI/CD with immutable artefacts is better; see [CI CD](CI%20CD.md)
> - **Not to provision cloud resources** — Terraform has state, plans and dependency graphs
> - **Not as a general-purpose programming language** — YAML with Jinja2 loops and conditionals becomes unreadable quickly
> - **Not at boot time for autoscaled instances** — bake the image instead
> - **Not without linting and a check-mode run** on anything that touches production

---

## 12. Advantages and Disadvantages

**Advantages**
- Agentless — SSH and Python are the only requirements
- Readable YAML that non-specialists can follow
- Genuinely idempotent when you use modules rather than shell
- Huge module and collection library
- Works on anything with SSH, including devices you cannot install software on
- Excellent for one-off orchestrated operations across many hosts
- Combines well with Packer for immutable images

**Disadvantages**
- **Slow** — SSH round trips per task, per host
- YAML is a poor language for control flow, and it shows
- Variable precedence is genuinely confusing
- No real state file, so it cannot tell you what it does *not* manage
- Mutable-server model allows drift to accumulate
- `--check` mode is unreliable once shell tasks are involved
- Debugging Jinja2 templating errors is unpleasant
- Easy to write playbooks that are shell scripts in disguise

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Per-task latency** | An SSH round trip per task per host; this dominates everything |
| **`forks`** | Default 5 concurrent hosts — raise it substantially for large fleets |
| **Pipelining** | Reduces SSH operations per task; a large win, enable it |
| **Fact gathering** | Seconds per host; disable or cache it when not needed |
| **Loops** | Prefer a module's native list support over looping the module |
| **Large fleets** | Hundreds of hosts take real time; consider pull mode or images |

> [!TIP]
> **Three settings routinely cut playbook time by more than half: `pipelining = True`, a higher `forks`, and `gathering = smart` with a fact cache.** They are configuration file changes, not rewrites. If a playbook takes forty minutes across a fleet, start there before restructuring anything.

---

## 14. Security Considerations

> [!CAUTION]
> **The Ansible control node holds SSH access with root escalation to your entire fleet, which makes it one of the most valuable targets you own.** Running playbooks from engineers' laptops means those laptops are effectively production infrastructure. Run from CI with a dedicated key, restrict which hosts that key can reach, and log every run.

- **Ansible Vault for secrets**, with the vault password itself in a secret manager — not in the repository
- **Never plaintext credentials in playbooks, inventories or group vars**
- **`no_log: true`** on tasks handling sensitive values, or they appear in output
- **Dedicated SSH keys per environment**, short-lived where possible, and certificate-based if you can
- **`become` only where needed**, rather than at the play level by default
- **Run `--check --diff` first** on production changes
- **Review playbooks like production code** — they execute as root everywhere
- **Pin collection and role versions**; a Galaxy role is code that runs with root privileges
- **Prefer immutable images**, which remove the need for standing SSH access to running machines at all

---

## 15. Mental Model

> [!NOTE]
> **Ansible is a checklist walked by a very fast, very literal technician.**
>
> Each line says what the machine should look like, not what to do — "nginx installed", not "run the installer". The technician checks each item, changes only what is wrong, and reports what they touched. Run the checklist again and they should change nothing. The trouble starts when you write "run this command" instead of "this should be true", because then the technician cannot tell whether anything needed doing — and neither can you.

---

## 16. Mini Architecture Diagram

```text
   TERRAFORM ──► instances exist (or bare metal already does)
                          │
   ┌──────────────────────┼──────────────────────────────────┐
   │  BEST PRACTICE: configure the IMAGE, not the server      │
   │                                                           │
   │     PACKER + ANSIBLE ──► golden AMI ──► launch template   │
   │           │                                               │
   │      declarative config, immutable result                 │
   └───────────┼───────────────────────────────────────────────┘
               │
   ┌───────────▼─── OR, for existing fleets ──────────────────┐
   │  CONTROL NODE (in CI, not a laptop)                       │
   │    inventory (dynamic, from the cloud API)                 │
   │    playbook → roles → tasks → modules                      │
   │         │ SSH, forks=50, pipelining on                     │
   │    ┌────┼────┬────────┬────────┐                           │
   │    ▼    ▼    ▼        ▼        ▼                           │
   │  web-1 web-2 web-3   db-1    cache-1                       │
   │  packages · files · services · users · certs               │
   │         handlers fire only on actual change                │
   └───────────────────────────────────────────────────────────┘
```

---

## 17. Complete Request Flow

```text
A configuration change is needed: a new nginx setting on all web servers
    ↓
Engineer edits roles/nginx/templates/nginx.conf.j2
    ↓
Pull request: ansible-lint and yamllint run in CI
    ↓
Reviewed and merged
    ↓
CI runs the playbook with --check --diff against production first
    ↓
Diff shows one file changing on three hosts, and nothing else
    ↓
Real run: dynamic inventory queries the cloud API for hosts tagged web
    ↓
Facts gathered (cached), then tasks run across 50 hosts in parallel
    ↓
The template task reports "changed" on all three
    ↓
The `Restart nginx` handler fires ONCE per host, at the end — not per task
    ↓
Serial batching keeps two hosts in service while one restarts
    ↓
Recap: changed=3, failed=0
    ↓
─────────────── the better version of this flow ───────────────
The same role runs inside Packer instead
    ↓
A new AMI is produced and tested
    ↓
The launch template is updated; the ASG rolls instances gradually
    ↓
No SSH into production, no drift, and rollback is the previous AMI
    ↓
─────────────── the failure mode Ansible prevents ───────────────
Six months later, someone asks why web-3 behaves differently
    ↓
Run the playbook in check mode → it reports what is out of line
    ↓
On a hand-configured fleet, that question has no answer at all
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Ansible is for the inside of machines — use modules rather than shell so it stays idempotent, keep overridable values in `defaults/`, and prefer building images with Packer over configuring running servers.

---

## 19. Common Mistakes

- **`shell` and `command` everywhere**, destroying idempotence and check mode
- **Using it to create cloud resources**, where Terraform belongs
- **Configuring instances at boot** for an autoscaling group, so scaling depends on a playbook succeeding
- **Plaintext secrets** in group vars or inventories
- **Overridable settings in `vars/`** instead of `defaults/`, then fighting precedence
- **Default `forks=5`** on a large fleet, then concluding Ansible is slow
- **Fact gathering never disabled or cached**
- **Handlers that never fire**, because every task reports changed
- **Playbooks run from laptops**, making laptops production access
- **Unpinned Galaxy roles**, running third-party code as root
- **No `--check --diff` before production**
- **Using it to deploy application code** instead of an immutable artefact
- **Enormous playbooks with no roles**, unmaintainable within a year

---

## 20. Open Source Technologies

- **Ansible** (and `ansible-core` plus collections)
- **Packer** — the pairing that turns Ansible into immutable infrastructure
- **ansible-lint**, **yamllint** — catch most of section 19 automatically
- **Molecule** — actually test roles, in containers or VMs
- **ansible-vault**, **SOPS** — encrypted variables
- **Dynamic inventory plugins** — AWS, GCP, Proxmox; better than static files
- **AWX** — the open-source Ansible Tower: scheduling, RBAC, run history
- **kubespray** — a real-world example of Ansible at serious scale
- **OpenSCAP / CIS collections** — compliance baselines

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run your main playbook twice. The second run should report `changed=0`. If not, find the offending tasks.
- [ ] Count your `shell` and `command` tasks and replace the two easiest with real modules.
- [ ] Enable pipelining and raise `forks`; measure the difference.
- [ ] Check whether any secret sits in plaintext in your inventory or group vars.
- [ ] Take one role and run it inside Packer to produce an image instead.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Terraform (machines exist) → Ansible over SSH (what is inside) → running application
                    or, better: Packer + Ansible → immutable image → launch
```

## 2. Request Flow

```text
Input       an inventory of hosts and a declared desired state
    ↓
Processing  modules run over SSH in parallel, changing only what is wrong
    ↓
Output      consistent machines, a change report, and handlers fired only on real change
```

## 3. Real-World Usage

Ansible's centre of gravity moved. Where it once configured long-lived production servers, its strongest remaining roles are **image building with Packer**, **bare metal and on-premises**, **network devices**, and **compliance baselines**. That is not a decline in usefulness so much as a correction: the industry decided that replacing machines beats repairing them, and Ansible turned out to be very good at producing the thing you replace them with.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Agentless configuration management over SSH, declared in YAML |
| **Why does it exist?** | Because machine interiors drift, and hand configuration cannot be verified |
| **Where does it belong?** | Inside machines — after Terraform creates them, or inside Packer building an image |
| **When should I use it?** | Bare metal, on-premises, devices, self-managed data stores, and golden images |
