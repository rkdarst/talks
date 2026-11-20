# Supply chain attacks (SCAs) in scientific computing

RSE seminar series
Richard Darst, 2026-11-20


Audience: Research Engineers

Purpose: Understand the problem, see possible solutions, discuss what
to recommend to our users.  This is not a theoretical security-focused
talk.

:::{revealjs-notes}
* Intro, discussion, not lecture
* I'm not a security expert, this isn't a security talk.
* I have a tool I use, but I'm not saying it is the solution.
:::

...

This is made to be a presentation, and you can {revealjs-link}`see the
presentation mode with ?revealjs`.  The content should be the same,
but there is extra material in the reading view (after an ellipsis,
you find extra audience notes).  I'm testing this sphinx + revealjs
system, so give feedback.

This is not designed to only be a lecture - it should be a discussion.
I am not a security expert, though I have been around enough to give
this talk.




## Cybersecurity

Surely people know why security is important - it's the protection
from harm from the **"CIA" triad**:

* **Confidentiality loss**: does anyone else have access to the data?
* **Integrity loss**: is the data not modified?
* **Availability loss**: can *you* access the data when needed?

Hospital example:
* Stolen and shared medical records.
* Deleting critical "allergy" info in a patient's record.
* Ransomware attack so you can't provide any health care.

:::{revealjs-notes}
* We often think only of C, because of research
* I and A are less important, but still relevant.
:::




### Background: some common security terminology

* **Local privilege escalation**
  * e.g. user --> root on cluster
* **Remote code execution**
  * e.g. network user can run something on web server computer
* **Denial of Service**
  * Something stops working
* **Container escape**
  * Escape security boundaries like VMs or containers.

Worth reading [wp:Computer
Security](https://en.wikipedia.org/wiki/Computer_security) for more.

:::{revealjs-notes}
* These terms in the news a lot - **not only SCA:s**.
* I'm not pretending this is an introduction to CompSec
* But these are terms used for the latest vulnerabilities that come up
  (unrelated to SCAs)
:::

...

I mention these just because you have probably been hearing a lot
about HPC cluster security lately, and these are the most important
terms.  Reading the linked Wikipedia article is probably a good
summary for everyone.



### Supply chain attack (SCA)

Wikipedia:

> A supply chain attack is a cyber-attack that seeks to damage an
> organization by targeting less secure elements in the supply chain.

Representative example: **Hacked PyPI account uploads package which
gets pulled onto our cluster, stealing the user's SSH keys and other
data.**

...

* Common risk: we need many dependencies.
* High risk: research has many diverse dependencies.
* Dangerous risk: poisoned dependencies can run arbitrary code.




### How do bad packages get into the supply chain?

Ways to get a bad package in the supply chain:

* Maintainer account compromise.
* Maintainer decision to poison package.
* Release infrastructure compromise.
* An earlier supply chain attack.
* ... any other security vulnerability.

:::{revealjs-notes}
* We mostly think about code execution,
* ... but it can be much more.
* Is Denial of Service a SCA?
:::



### What can SCA:s do, once on your computer?

Payload can run at install time, run time, or persist.

* Anything: confidentiality, integrity, availability.
* Most run code and extract information from your computer
  * Tokens, SSH keys, files, data, ... (next slide)
* Add files to your computer
  * e.g. new SSH authorized keys
* Research is not exempt from these risks.

:::{revealjs-notes}
* As an example, at `pip install` time
* Python `.pth` file at interpreter startup
* Python `import`-time
:::



### What info can an attack get from your computer?

* Every file on your computer (e.g. secret files in `~/`)
* Every environment variable
* Environment variables from other processes (`/proc/*/environ`) (same
  user)
* Command line from other processes (`/proc/*/cmdline`) (every user)

[Read more at the microsoft-agent analysis](https://www.stepsecurity.io/blog/microsoft-agent-framework-pypi-package-compromised#payload-capabilities)


:::{revealjs-notes}
Discussion
* What happens if it runs on our cluster?
* All users' data.

END SECTION
:::


## Example supply chain attacks

:::{revealjs-notes}
* Fun section to write - so many things to choose from.
* This is only a small selection.
:::



### Example supply chain attacks, 1/3

* `left-pad` - package removal ([ref](https://en.wikipedia.org/wiki/Npm_left-pad_incident))
   * npm author got mad and removed the package
   * Many sites failed building (denial of service).
* SolarWinds hack ([ref](https://en.wikipedia.org/wiki/SolarWinds#2019%E2%80%932020_supply_chain_attacks))
    * Network monitoring company.
    * Hackers got into SolarWinds.
    * SolarWinds access allowed access to other US government systems.
* ``xz-utils`` ([ref](https://en.wikipedia.org/wiki/XZ_Utils_backdoor))
    * A new maintainer took over.
    * When `liblzma` used in OpenSSH, it allowed root login to a
      certain key.




### Example supply chain attacks, 2/3

* `node-ipc` - added `peacenotwar` as a dependency ([ref](https://en.wikipedia.org/wiki/Peacenotwar))
  * `peacenotwar` detected if it was in RU/BY and overwrote every
    file on computer with heart emoji
* Claude Code in *Cursor* extension marketplace
  ([ref](https://phorion.io/blog/offical-claude-code-vsx-compromise/))
  * Extension marketplaces often overlooked as a vector.
* `litellm` ([ref](https://en.wikipedia.org/wiki/LiteLLM#Security))
  * Attacked twice in one year.
  * Ran arbitrary code even on *install*.




### Honorable mentions, 3/3 (1-2 aren't attacks)

* Samsung fridge update bricking them ([ref](https://arstechnica.com/gadgets/2026/09/owners-mourn-spoiled-food-after-firmware-update-bricks-samsung-smart-fridges/))
  * Accidental update... not an attack.
* CrowdStrike update in 2024 ([ref](https://en.wikipedia.org/wiki/CrowdStrike#2024_IT_outages))
  * An auto-update to security software broke other services.
* **Yesterday**: `agent-framework` on PyPI
  ([ref](https://www.stepsecurity.io/blog/microsoft-agent-framework-pypi-package-compromised))
  * Execute at Python startup via `.pth` file (no import needed)
  * Vector: malicious maintainer account added to project.
  * Credential stealer (conf files, env vars, /proc/*/environ)

:::{revealjs-notes}
* Look at agent-framework for an example of how bad it can be.
* How deep does it go, to physical effects?:
  * Freezers turned to heat?
  * Outlook outage and chlorine poisoning
* Typosquatting?

END SECTION
:::

...

Some of these are just caused by accidents or incompetence, but it's
worth thinking about the bigger supply chain risk at the same time.



## Analysis

Various observations:

* Many types of SCA:s.
* Dependencies
  * Make ecosystems useful (e.g. extension marketplaces).
  * But make more vectors.
* Dependencies can be pulled in recursively.
  * It's not just the dependencies you choose.
  * This Sphinx project has 5 direct dependencies leading to *91*
    installed packages!
* Every new dependency is a risk.



### Analysis: level of protection

* Our primary protection seems to be "don't run anything un-approved".
* Most of our work is user-isolated, not project-isolated.
  * One user account has access to many projects.
  * One compromised dependency can affect *every* project's data.
* We are quite un-prepared.



### Analysis: my opinions

* I don't want to run any custom environments without protection.
* Containerization seems appropriate.
  * There can be SCA combined with container escape.
  * ... but still simple containerization reduces the blast radius a lot.
* Can easily turn into "dump more stuff on our users"

A universal solution seems unlikely.



### Usability considerations

Our protection should:

* *Easy to use* since it needs to be used *everywhere*.
* Low start-up barrier (easy to use for every throw-away test)
* Works on different platforms
* Works local and remote
* Easy to understand

:::{revealjs-notes}
END OF SECTION
:::

...

We can think of many solutions, but what is most usable?  Our audience
is researchers, not Linux experts.  On top of that, consider how much
harder it is to manage on other OSs.



## Possible solutions



### Minimizing or reviewing dependencies

* I think minimizing is a good idea, but not always practical.
* IT Services thinks we get everything approved, but we know that's not
  happening.
* **Analysis: not enough**

:::{revealjs-notes}
* Reviewing everything is impossible.
* Always good to use less stuff.
* Science still needs lots of weird dependencies.
* Joke: Can AI rewrite everything for your own code so you don't need
  dependencies anymore?
:::



### Cooldowns

* Concept: a package must be on the archive for X days before it is
  eligible for installation
* For example
  * `uv pip install --exclude-newer '3 days' foo`
  * `pip install --uploaded-prior-to P3D foo`
  * See more: <https://cooldowns.dev/>
* Side-effect: delays security updates.
* **Analysis: good practice, usable, not enough for my tastes.**


:::{revealjs-notes}
* Buffer time between upload and first install
* If everyone uses, then ???
* But at least it gives automated scanners time
:::


### Locking dependencies

e.g. `pylock.toml`, `uv.lock`

* Create lockfile with exact versions for your environment.
* Fewer updates --> smaller attack window.
* Good practice for research.
* Maybe not flexible enough for developers

:::{revealjs-notes}
* This is a good idea, but for me...
  * I want to rotate dependencies in my dev environment frequently
  * and find breaking updates quickly.
:::


### Containerization and Linux namespaces

e.g. run in Apptainer.

* Reasonable (but "containers aren't designed for security").
* Still much better than nothing.
* e.g. Apptainer, bubblewrap
* **Analysis: not perfect but good.  Needs usability to be widely
  adopted.**

:::{revealjs-notes}
* I like this
* You know Apptainer
* Bubblewrap more directly uses namespaces
* I made a way to use containers without building a new one
:::



### VMs, separate accounts, enclaves, etc

* Request separate VMs for everything?
* Isolate projects by separate user accounts?
* **Analysis: too heavyweight to be widely used at the scale we need.**

:::{revealjs-notes}
* I guess we know what virtual machines are
* Also separate user account ideas
* Too heavyweight
:::




### Hierarchy of controls

This is a concept from industrial safety:

```{image} https://upload.wikimedia.org/wikipedia/commons/3/36/NIOSH%E2%80%99s_%E2%80%9CHierarchy_of_Controls_infographic%E2%80%9D_as_SVG.svg
```

Where does each solution fit?

[image: [NIOSH via Wikimedia Commons](https://commons.wikimedia.org/wiki/File:NIOSH_Hierarchy_of_Controls.png)]

:::{revealjs-notes}
* You know I need something about accident investigations, right?
:::

...

We shouldn't just dump more load onto our users.  Is there any
possible way to re-design things to make things naturally easier?



### Possible solutions: a list

* Minimize dependencies
* Review every dependency
* Lock dependencies, update infrequently
* "Cooldowns" (delayed install)
* Containers: Apptainer, etc.
* Automated scanning?
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

What is practical?  --> Discussion later

END OF SECTION
:::



## Demo: venv-apptainer

* I wrote venv-apptainer to isolate *every* Python project.
* I am not saying others should use it (but I do think it's good).
  * Should have more docs and testing before wide usage.
* This demo is to give some technical aspect to the seminar.

:::{revealjs-notes}
* Wanted to say something technical
* Not saying this is the best (but it works well for me)
* Go/no-go for the section?
:::

...

<https://github.com/rkdarst/venv-apptainer>



### Executive summary

* I (rkdarst) made this: <https://github.com/rkdarst/venv-apptainer>

```console
$ vea
... creating ./venva/ from requirements.txt
$ which python
./venva/bin/python       <-- wrapper script for container
$ python my_file.py
... runs isolated to $PWD automatically
```

* Designed for standalone project dir isolation
* Supports pip, conda, uv
* I've used it for every `pip install` since June-ish.



### Background and history

* From Radovan: told me of a `ve` alias for making envs.
* For years, I've had a `ve` shell function that:
  * makes `./venv/` if it doesn't exist
    * automatically installs `requirements.txt`
  * activates `./venv/` if it exists
* I wanted a new version using containers.



### Design basis

I wanted something with:

* `$PWD` mounted in the container (and nothing else).
* Suitable for mass use: easy `venv` for every project.
* No config files - simple use case only.
* Safe by default.
* Wrapper scripts for binaries in the environment (easy execution).
* *Not* building a new container for every project.
* 100% human written.



### Filesystem layout of `./venva/`

On host: everything contained in `./venva/`:

```
./venva/          --> Base dir
./venva/*.squashfs --> generated squashfs of env
./venva/exec      --> Execution wrapper (shell or command)
./venva/bin/*     --> Wrapper scripts for everything
                      in /venv-apptainer/bin/*
./venva/activate  --> activation wrapper
```

(pip/conda/uv/etc cache dirs are also mounted)

Inside container layout:

* `/venv-apptainer/` is the venv.
* Other paths mounted as on the host (`$PWD`, etc.).



### Usage (my automation)

Has my preferred auto-detection for `requirements.txt`,
`environment.yml`, `pylock.toml`, etc.

What I do:

1. `vea` (auto-detect requirements, and make environment and activate it)
2. Run whatever (`sphinx-build`), wrapper `./venva/bin/sphinx-build`
   runs in container with `$PWD` mounted.

(do something else; come back later):

3. `vea` (re-activates existing environment)



### Usage: Manual control of install

Can flexibly control the install options:

```console
$ vea --pip -r requirements.txt other_dependency
$ vea --conda -f env-test.yml
$ vea --uv sync
$ vea --uv pip install ARGS
## `pip install -e` of another project to here
$ vea --bind-ro ../other/ -r requirements.txt -e ../other/
```
...

Want to try it yourself?

* **You have requirements.txt and use pip**: `vea`
* **You have environment.yml and use miniconda**: `vea`
* **You have pyproject.toml and use uv**: `vea --uv sync`



### Advanced options

I've added the advanced options I need:

* Custom binds (`--bind`, `--bind-ro`)
* Custom images (`--img`)
* No squashfs (env can be updated): `--no-squash`

Limitations:

* Not designed for system library integration (GPU, Slurm) but could
  probably be added.

:::{revealjs-notes}
END OF SECTION
:::



## Wrap up and discussion

* I would be happy if anyone else demoed solutions.
* Most **usable** solutions for researchers?
* How can we search for bad packages?
  * Can hide in environments, containers, ...
* Any other questions and comments?


:::{revealjs-notes}
* There is a lot to do, as usual biggest problem is usability.
* There is plenty more legal and administrative stuff
  * Cyber Resilience Act
    * Effect on open source
    * Paper security != real security (CrowdStrike) ("is inserting one
      software into every critical path really security?")
* On my side I learned I need --pid in venv-apptainer.
* Trust the user vs block the user (e.g. locked-down OS)
:::

...

In conclusion, I think that supply chain attacks are very relevant to
scientists who install and use lots of different software.  There are
many solutions, but as usual the problem is making it usable enough
for mass adoption.  Us HPC admins have been thinking about this and
occasionally looking for vulnerable packages, but I think we need to
go beyond reactive and start figuring out how to proactively protect
our users.
