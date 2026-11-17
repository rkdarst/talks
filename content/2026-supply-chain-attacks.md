# Supply chain attacks

RSE seminar series
Richard Darst, 2026-11-16


Audience: Research Engineers
Purpose: Understand the problem, see possible solutions, discuss what to recommend to our users.


## Cybersecurity

Surely people know why security is important.

"CIA" triple:

* **Confidentiality**: does anyone else have access to the data
* **Integrity**: is the data not modified?
* **Availability**: can *you* access the data when needed?

Hospital example: Stolen and shared medical records.  Deleting critical "allergy" info in a patient's record.  Ransomeware attack so you can't provide any heath care.

---

### Types of security vulnerabilities

Most relevant terminology for us:

* Local privilege escalation
    * e.g. user --> root on cluster
* Remote code execution
    * e.g. network user can run something on web server computer
* 

---

### Supply chain attack

Wikipedia:

> A supply chain attack is a cyber-attack that seeks to damage an organization by targeting less secure elements in the supply chain.

Prototype for us **Hacked PyPI upload which gets pulled into our cluster**

* Quite common: we need many dependencies
* Quite risky: research has many diverse dependencies.
* Quite dangerous: poisoned dependencies can run anything.

---

## Example supply chain attacks

----

### Example supply chain attacks, 1/2

* `leftpad` - package removal (availability)
   * npm author got mad and removed the package
   * Many big sites failed building.
* Solar Winds - hack
    * US network monitoring company
    * Hackers got into SolarWinds
    * SolarWinds access allowed access to other US government sites
* ``libxz``
    * A unknown person became libxz maintainer (after a long period of actual contributions)
    * Poisoned `libxz` such that it only affects `openssh`
    * Allowed SSH access to compromised servers


----

### Example supply chain attacks, 2/2

* `node-ipc` - availability
    * Poisoned package detected if it was in RU/BY and overwrote every file on computer with :heart:
* Claude Code in Cursor extension marketplace
    * https://phorion.io/blog/offical-claude-code-vsx-compromise/
* `litellm` - PyPI
    * Ran arbitrary code even on *install*


----

### So what can SCA:s do?

* Anything: confidentility, integriety, availabity.
* Can affect at install time, run time, or in persistent services
* Research is not exempt from these problems


---

## Analysis

Various observations:

* Many types of SCA:s and it needs to be in our mind
* Most of our work is user-separated.
    * One user account has access to many projects
    * One poisoned dependency affects every project
* Dependencies can be pulled in recursively.
  * It's not just the dependencies you choose.
* Every new dependency is a risk.

----

### Analysis: my opinions

* I don't want to run any internet-installed stuff without protection
* Containerizaton (protecting most of the filesystem is enough)
  * There can be SCA combined with container escape
  * ... but still simple containerization reduces the blast radius a lot

I wish we could find one solution, but I don't see it likely.

----

### Usability of a solution

Our protection should be:

* *Easy to use* since it needs to be used *everywhere*.
* Low start-up barrier (not at the project level)
* Works on different platforms
* Works local and remote
* Easy to understand

---

## Existing solutions


### Existing solutions

* Containers: apptainer, etc.
* Virtual machines
* Separate user accounts for different projects
* Secure data enclaves

---

### venv-apptainer

* I (rkdarst) made this: https://github.com/rkdarst/venv-apptainer

```console
$ vea
... creating ./venva/ from requirements.txt
$ which python
./venva/bin/python       <-- wrapper script for container
$ python my_file.py
... runs isolated to $PWD automatically
```

* Designed for standalone projects
* Supports pip, conda, uv
* I've used it for every `pip install` since june-ish.

---

### moat

* Simo's thing for AI agents: https://github.com/AaltoRSE/moat
* More flexible, wider mission, more configuration
* Will be talked about later

---

## Demo: venv-apptainer

* I wrote venv-apptainer
* I am not saying others should use it
* This demo is to give some technical aspect to the seminar

----

### History

* From Radovan: apptainer-venv inspiration
* For years, I've had a `ve` shell function that
  * Makes ./venv if it doesn't exist
  * Activates ./venv if it exists

----

### Design basis

I wanted something with:

* $PWD in container, nothing else on the system.
* Suitable for mass use: new venv for every project
* No config files - make own container for advanced needs
* Safe by default. Add more mounted paths if needed.
* Wrapper scripts, no need to define commands to run
* 100% human written

----

### Filesystem layout

On host: everything contained in `./venva/`:

```console
./venva/          --> Base dir
./venva/exec      --> Execution wrapper (shell or command)
./venva/bin/*     --> Wrapper scripts for everything
                      in /venv-apptainer/bin/*
./venva/activate  --> activation wrapper
./venva/*.squashfs --> generated squashfs
./venva/tmp/       --> Temporary work spaces (home, tmp)
```

(pip/conda/uv/etc cache dirs are also mounted - but not the outside ones.)

Inside container: `/venv-apptainer/` is a normal venv.  Other paths mounted as on host.

----

### Usage (my automation)

What I do:

1. `vea` (auto-detect requirements, and make environment, and activates it)
2. Run whatever (`sphinx-build`), wrapper `./venva/bin/sphinx-build` runs in container with $PWD mounted.
3. do something else; come back later
4. `vea` (re-activates existing environment)

Auto-detection for `requirements.txt`, `environment.yml`, `pylock.toml`, etc.

----

### Usage: Manual control

Each of these passes the arguments to the respective installer in the container.

```console
$ vea --pip -r requirements.txt other_dependency
$ vea --conda -f env-test.yml
$ vea --uv pip install ARGS
$ vea --uv sync
```

Other options:
```console
## pip install -e of another project
$ vea --bind-ro ../other/ -r requirements.txt -e ../other/
$ vea --no-squash      # lets you `pip install` further`
```


----

### Advanced options

* Custom binds (`--bind`, `--bind-ro`)
* Custom images (`--img`)


---

## Discussion and/or demo

PRE-QUESTION: do people want longer theory of venv-apptainer?

* I can demo and discuss `venv-apptainer`
* Any other questions and comments?


---

## OLD BELOW HERE
## Outline
- Why is security important, CIA
- Types of security vulnerabilities (small glossary)
  - local privileged escalation, remote code execution, input validation, removal of dependencies
  - These matter even in research, in different ways

- Supply chain attacks - definition, what are the risks
  - RCE - can be anything. A lot of credential steeling. Crypto mining
  - Effects and risks
  - Types of supply chains

- History of some major ones
  - (to be selected from the list below)

- Possible solutions
  - install cooldowns (meme: when everyone has a cooldown, no one does)
  - pin dependencies
  - venv-apptainer, moat
  - Project vs user level isolation
  - Contributions from the audience


## List of famous supply chain attacks to consider
  - leftpad - not security but affects ability to re-deploy
  - Solar Winds
  - libxz
  - node-ipc (https://en.wikipedia.org/wiki/Peacenotwar)
  - common: litellm
  - Extensions in marketplaces (Claude Code in Cursor marketplace: https://phorion.io/blog/offical-claude-code-vsx-compromise/)
  - https://x.com/binance/status/2117626126214504729?a=E58iUWbiQFFRjbf3tEU&s=46
