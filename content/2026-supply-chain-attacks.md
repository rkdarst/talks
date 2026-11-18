# Supply chain attacks (SCAs) in scientific computing

RSE seminar series
Richard Darst, 2026-11-16


Audience: Research Engineers

Purpose: Understand the problem, see possible solutions, discuss what
to recommend to our users.  This is not a security-focused talk.

:::{revealjs-notes}
* Intro, discussion, not lecture
* I'm not a security expert, this isn't a security talk.
* I have a tool I use, but I'm not saying it is the solution.
:::

...

This is made to be a presentation, and you can {revealjs-link}`see the
presentition mode with ?revealjs`.  The content should be the same,
but there is extra material in the reading view (after an ellipsis,
you find extra audience notes).  I'm testing this sphinx + revealjs
system, so give feedback.

This is not designed to only be a lecture - it should be a discussion.
I am not a security expert, thought I have been around enough to give
this talk.




## Cybersecurity

Surely people know why security is important - it's the protection
from harm from the **"CIA" triple**:

* **Confidentiality loss**: does anyone else have access to the data
* **Integrity loss**: is the data not modified?
* **Availability loss**: can *you* access the data when needed?

Hospital example: Stolen and shared medical records.  Deleting
critical "allergy" info in a patient's record.  Ransomeware attack so
you can't provide any heath care.

:::{revealjs-notes}
* We often think only of C, because of research
* I and A are less important, but still there.
:::




### Types of security vulnerabilities

Most relevant terminology for us (*not everything*):

* Local privilege escalation
    * e.g. user --> root on cluster
* Remote code execution
    * e.g. network user can run something on web server computer
* Denial of Service
  * Something stops working
* Container escape
  * Escape security boundaries like VMs or containers.

Worth reading [wp:Computer
Security](https://en.wikipedia.org/wiki/Computer_security] for more.

:::{revealjs-notes}
* I'm not pretending this is an introduction to CompSec
* But these are terms used for the latest vulnerabilities that come up
  (unrelated to SCAs)
:::




### Supply chain attack

Wikipedia:

> A supply chain attack is a cyber-attack that seeks to damage an
>  organization by targeting less secure elements in the supply chain.

Representative example: **Hacked PyPI account uploads package which
gets pulled onto our cluster**

* Common risk: we need many dependencies
* High risk: research has many diverse dependencies.
* Dangerous risk: poisoned dependencies can run arbitrary code.




### What are SCA entry points?

* Maintainer account compromise.
* Maintainer decision to poison package.
* Release infrastructure compromise.
* An earlier supply chain attack.
* Typosquatting(?)

:::{revealjs-notes}
* We mostly think about code execution,
* ... but it can be much more.
* Is Denial of Service a SCA?
:::




## Example supply chain attacks




### Example supply chain attacks, 1/3

* `leftpad` - package removal ([ref](https://en.wikipedia.org/wiki/Npm_left-pad_incident))
   * npm author got mad and removed the package
   * Many big sites failed building (denial of service).
* Solar Winds - hack ([ref](https://en.wikipedia.org/wiki/SolarWinds#2019%E2%80%932020_supply_chain_attacks))
    * US network monitoring company
    * Hackers got into SolarWinds
    * SolarWinds access allowed access to other US government sites
* ``libxz`` ([ref](https://en.wikipedia.org/wiki/XZ_Utils_backdoor))
    * A new maintainer took over.
    * Poisoned `libxz` such that it allowed `openssh` root login to a
      certain key.
    * Allowed SSH access to compromised servers.




### Example supply chain attacks, 2/3

* `node-ipc` - added `peacenotwar` as a dependency ([ref](https://en.wikipedia.org/wiki/Peacenotwar))
  * `peacenotwar` detected if it was in RU/BY and overwrote every
    file on computer with heart emoji
* Claude Code in *Cursor* extension marketplace
  ([ref](https://phorion.io/blog/offical-claude-code-vsx-compromise/))
* `litellm` - PyPI uploads... twice in one year.  ([ref](https://en.wikipedia.org/wiki/LiteLLM#Security))
  * Ran arbitrary code even on *install*




### Honorable mentions, 3/3

* Samsung fridge update bricking them ([ref](https://arstechnica.com/gadgets/2026/09/owners-mourn-spoiled-food-after-firmware-update-bricks-samsung-smart-fridges/))
  * Accidental update... not an attack
* Cloud Strike update in 2026 ([ref](https://en.wikipedia.org/wiki/CrowdStrike#2024_IT_outages))
  * An auto-update to security software broke other services.

:::{revealjs-notes}
* How deep does it go, to physical effects?:
  * Freezers turned to heat?
  * Outlook outage and Chlorine poisoning
:::



### So what can SCA:s do?

* Anything: confidentiality, integrity, availability.
* Can affect at install time, run time, or in persistent services.
* Research is not exempt from these problems.




## Analysis

Various observations:

* Many types of SCA:s and we need to consider them.
* Dependencies are what makes ecosystems useful
  * e.g. extension marketplaces.
* Most of our work is user-isolated, not project-isolated.
    * One user account has access to many projects.
    * One compromised dependency can affect *every* project's date.
* Dependencies can be pulled in recursively.
  * It's not just the dependencies you choose.
  * This Sphinx project has 5 direct dependencies leading to *91*
    installed packages!
* Every new dependency is a risk.




### Analysis: my opinions

* I don't want to run any internet-installed stuff without protection.
* Containerizaton (protecting most of the filesystem is enough).
  * There can be SCA combined with container escape.
  * ... but still simple containerization reduces the blast radius a lot.

A universal solution seems unlikely.




### Usability considerations

Our protection should be:

* *Easy to use* since it needs to be used *everywhere*.
* Low start-up barrier (easy to use for every throw-away test)
* Works on different platforms
* Works local and remote
* Easy to understand




## Possible solutions




### Possible solutions: broad categories

* Minimize dependencies
* Review every dependency
* Lock dependencies, update infrequently
* "Cooldowns"
* Containers: Apptainer, etc.
* Separate user accounts for different projects
* Virtual machines for each project
* Secure data enclaves (CSC SD-services, Aalto SecData)


:::{revealjs-notes}
* Many options
* Cooldowns and locking good idea for end-user projects.
* My solution: Minimize.  Everything in container.  Everything.
* Many are not usable or agile enough.
* *not* including filling out forms here.
  * Software bill of materials (list dependencies)
  * Legislate liabilities
  * Ask about economic impact of regulation
:::


### Minimizing or reviewing dependencies

* I think minimizing is a good idea, but not always practical.
* IT Services think we get everything approved, but we know that's not
  happening.
* **Analysis: not enough**



### Cooldowns

* Concept: a package must be on the archive for X days before it is
  eligible for installation
* For example
  * `uv pip install --exclude-newer '3 days' foo`
  * `pip install --uploaded-prior-to P3D foo`
  * See more: https://cooldowns.dev/
* Can delay security updates.
* **Analysis: good practice, usable, not enough for my tastes.**


:::{revealjs-notes}
* If everyone uses, then ???
* But at least it gives automated scanners time
:::



### Containerization

* Reasonable but containers aren't perfect for security.
* Still much better than nothing.
* **Analysis: not perfect but good enough.  Needs usability for common
  workflows**



### VMs, separate accounts, enclaves, etc

* **Analysis: too heavyweight to be used to isolate everything we work
  on.**



### Hierarchy of controls

This is a concept from industrial safety:

```{image} https://upload.wikimedia.org/wikipedia/commons/3/36/NIOSH%E2%80%99s_%E2%80%9CHierarchy_of_Controls_infographic%E2%80%9D_as_SVG.svg
```

Where does each solution fit?

[image: [NIOSH via Wikimedia Commons](https://commons.wikimedia.org/wiki/File:NIOSH_Hierarchy_of_Controls.png)]



## Demo: venv-apptainer

* I wrote venv-apptainer to isolate *every* Python project.
* I am not saying others should use it (but I do think it's good).
  * Should have more docs and testing before wide usage.
* This demo is to give some technical aspect to the seminar.


### Executive summary

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



### Background and history

* From Radovan: apptainer-venv had `ve` alias for NixOS.
* For years, I've had a `ve` shell function that:
  * makes `./venv/` if it doesn't exist
  * activates `./venv/` if it exists
  * automatically installs `requirements.txt`
* I wanted to upgrade this for containers.



### Design basis

I wanted something with:

* `$PWD` mounted in the container (and nothing else).
* Suitable for mass use: easy `venv` for every project.
* No config files - simple use case only.
* Safe by default. Add more mounted paths if needed.
* Wrapper scripts for binaries in the environment (no need to define
  commands to run)
* *Not* building a new container for every project.
* 100% human written



### Filesystem layout of `./venva/`

On host: everything contained in `./venva/`:

```console
./venva/          --> Base dir
./venva/*.squashfs --> generated squashfs of env
./venva/exec      --> Execution wrapper (shell or command)
./venva/bin/*     --> Wrapper scripts for everything
                      in /venv-apptainer/bin/*
./venva/activate  --> activation wrapper
```

(pip/conda/uv/etc cache dirs are also mounted)

Inside container layout: `/venv-apptainer/` is the venv.  Other paths
mounted as on host.



### Usage (my automation)

What I do:

1. `vea` (auto-detect requirements, and make environment, and activates it)
2. Run whatever (`sphinx-build`), wrapper `./venva/bin/sphinx-build` runs in container with $PWD mounted.
3. (do something else; come back later)
4. `vea` (re-activates existing environment)

Auto-detection for `requirements.txt`, `environment.yml`, `pylock.toml`, etc.



### Usage: Manual control

Various way to make an environment, passing through options to
pip/uv/miniconda to run inside:

```console
$ vea --pip -r requirements.txt other_dependency
$ vea --conda -f env-test.yml
$ vea --uv sync
$ vea --uv pip install ARGS
```

Advanced control:
```console
## pip install -e of another project
$ vea --bind-ro ../other/ -r requirements.txt -e ../other/
$ vea --no-squash      # lets you `pip install` further`
```

...

Want to try it yourself?

* **You have requirements.txt and use pip**: ``vea``
* **You have environment.yml and use miniconda**: ``vea``
* **You have pyproject.toml and use uv**: ``vea --uv sync``



### Advanced options

I've added the advanced options I need:

* Custom binds (`--bind`, `--bind-ro`)
* Custom images (`--img`)

Limitations:

* Not designed for system library integration (GPU, Slurm) but could
  probably be added.




## Discussion and/or demo

* I would be happy if anyone else demoed solutions
* Any other questions and comments?


:::{revealjs-notes}
* There is a lot to do, as usual biggest problem is usability.
* There is plenty more legal and administrative stuff
  * Cyber Resilience Act
    * Effect on open source
	* Paper security != real security (CloudStrike) ("is inserting one
      software into every critical path really security?")
:::
