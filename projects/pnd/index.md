---
release: 0.2.1
date: 2018-10-11
authors:
 - L. Bianchi, me@ludob.com
---

# Changelog
- 0.2.1: Minor edits
- 0.2: Create Markdown version
- 0.1: Initial draft

# Preface

This is a draft version of a document describing the principles and design choices of `pnd`, the PandaRoot companion utility to manage and streamline common operations.
The focus of the document is on the concept rather than the implementation, and in the identified issues more than their solution. Code snippets are included mostly to show the proposed API, and occasionally some pseudocode-like description of an algorithm.
Finally, this document is built from the experience, opinion and point of view of its author(s), which are inevitably partial and subjective.
Any lack of knowledge and misrepresentation of the PandaRoot codebase is to be considered due to the author(s) of this document, rather than the authors of the code itself.
The document is structured in sections, each of which focused on showing the motivation, desired functionality, and proposed API. 

# Introduction
As with any other project of similar size and scope, using the PandaRoot framework is a complex endeavor.
Even simple tasks, such as running simulations using the standard detector setup, require a detailed knowlegde of many components.
Documentation and tutorials are available, and especially for the core operations (installation and analysis), of good quality and coverage. However, the sheer volume of information makes it easy, for experts and novices alike, to make mistakes during the installation and setup process.  
In addition, to be useful, the documentation needs to be kept up to date, which is an infamously hard challenge in a framework that is subject to constant evolution and improvements such as PandaRoot.  
Finally, there is a large number of tasks across different applications that lie in an unconfortable grey area: not basic enough to be covered in the tutorials, but still necessary, to the point that most users end up re-implementing the functionality by themselves. Some scripts are available for some of these tasks, but a) the users need to be aware of them, and b) they are subjected to the same maintenance penalty as the documentation.

To address some of these issues, at the core of this proposal is the idea of creating a single, high-level entry point for users to group most, if not all, the available functionality of the PandaRoot framework.

More in detail:

- Identify the most common tasks, across a typical (but general) set of use cases
- Develop an official implementation for each of these tasks, grouped together as a modular tool
- Distribute the modular tool as an integral part of the code

The proposed solution for the tool is a standalone program, invoked as `pnd` on the command line, and each tasks available as Git-style subcommands, possibly further grouped in namespaces.

# Principles

### Provide APIs, not tutorials
Writing and mantaining up-to-date documentation is hard, and in most cases not the most efficient way to arrive to the final goal.
Instead of writing a list of steps to accomplish `task_x`, it's better to provide a way to do `task_x` automatically.
Performing the task programmatically offers in addition a series of advantages that can be exploited to streamline and further automate the whole simulation pipeline.

### There should be one, and possibly only one way to do it
If a task is identified as common, then the functionality is included in `pnd`, and users instructed to perform the task through `pnd`.
Adding new functionality to `pnd` also happens in a centralized way (either via core developers implementing it, or via pull requests), and a new version of the tool is released.
In this way, all users are notified of the updates (and the availability of new functionality, or improvements of existing workflows) through the release notes.

### Separation of concerns between domain and implementation
It's generally possible to categorize an option or task as belonging either to the domain (in this case, simulations with PandaRoot) or the implementation.
Domain options should be exposed to the user, in the most consistent way across separate functionalities, while implementation options should be given sane defaults so that they can be safely ignored, but overrideable by experienced users.

### Do not break backwards compatibility with existing workflows
At least during the initial phase, the adoption of the tool should be recommended, not unavoidable.
The introduction of the new tool not prevent expert users to continue using their preferred workflows.

# Examples

## Installation

### Current status
The installation process is described as a detailed tutorial. Users, or their system administrators, are supposed to install each release by themselves.
### Issues
- The installation process is performed manually (error-prone)
- The directory structure needs to be specified manually by the user
    + Advanced decision, unoptimal for beginners
    + If directory structure is irregular, no assumptions can be made by orchestration tools, to e.g. select a specific version as a simulation parameter 

### Solutions
The installation process is managed by the `pnd install` command:

```sh
# with no arguments, fetches metadata about available versions from the server
$ pnd install

# -v/--version installs the specified version
$ pnd install -v oct18

# lists installed versions
$ pnd install list-installed

# or the ones available on the download server
$ pnd install list-available

# manually check if dependencies are OK
$ pnd install check
```

If an installation from a local source is needed, this is also available as 
```sh
$ pnd install --source=/path/to/source -v 'custom_version_tag'
```

The installed directory structure is kept uniform, so that `pnd` is at all time aware of the mapping between the version and its path on the filesystem.
Then the following tasks are trivial:
```sh
# cd's to the source (build) directory
$ cd $(pnd dir -v oct18 -s/-b)

# build a specific version, regardless of $PWD
pnd build -v oct18

# or all versions with modifications
pnd build -a
```

## Running simulations

### Current status
ROOT macros (interpreted C++) are used to steer the simulation process, most commonly by calling and initializing Tasks (compiled C++).
The ROOT interpreter is required to launch the macros, and PandaRoot functionality will be available if the `config.sh` script is launched beforehand from the directory where the desired PandaRoot version is installed:

```sh
$ source $PATH_TO_PR_VERSION_INSTALL/config.sh
$ root -l macro.C
# macro processes, interactive prompt not available
```

### Issues
- Launching environment is overwritten by the simulation environment
    + In other words, the simulation environment leaks into the launching environment. As system environment is global state, this can lead to hard-to-catch bugs.
    + Conceptually, the simulation and steering environment should be kept separate. This issue manifests itself in a particularly frustrating way when one wants to compare simulation results across different versions of the framework, a very common task to compare the effects of e.g. changes in the geometry of the detector. Not only the required steps are cumbersome, they are highly dependent on the order with which they were performed. Launching  the jobs in parallel, depending on the strategy, would introduce another non-deterministic layer of confusion.

### Proposed solutions

The central entry point to run a simulation is `pnd run`:
```sh
$ pnd run

$ pnd run -m macro.C
```

`pnd run` launches the ROOT interpreter as a subprocess, so the necessary environment settings will not affect the launching process.
An interesting functionality is offered by enabling `pnd` with a bare-bones batch submission system, so that simulation jobs can be monitored and managed programmatically.

Comparing simulations using different versions of the framework becomes then:
```sh
$ pnd run -m macro.C -v jan16

$ pnd run -m macro.C -v feb17
```

Another essential benefit for this is allowing a unified interface for all tasks matching the high-level concept of "running a simulation", in a way which is decoupled from the actual infrastructure and methods used to perform the simulation.
For a concrete case, let's consider running the same simulation with different number of events. As a rule of thumb, no more than 1000-2000 events should be run as a single job.
From a conceptual point of view, there is no difference from running a simulation with 10 events (finishing almost instantaneously, for checking that the environment works as expected), and later running the exact same task on a higher number of events, to check the results with less statistical fluctuations.
In practice, this means that the user needs to start worrying about managing jobs, merging output files...: a completely different workflow, even if the intended task is very similar.

`pnd` can perform this switch automatically, and in principle completely transparently for the user:
```sh
$ pnd run -m macro.C -n 10
# > Starting 1 job with 10 events each...

$ pnd run -m macro.C -n 10000 # or 10K, or 1e5
# > starting 10 jobs with 1000 events each...

# or even:
$ pnd run -m macro.C -n 10000 --test
# > before launching 10 jobs with 1000 events each, will launch 1 job with 10 events each to test
```

The same principle can be applied to running the simulation jobs locally, vs. on a computing cluster:
```sh
$ pnd run -m macro.C -n 1e4  # can run locally in a reasonable time

$ pnd run -m macro.C -n 1e6 --cluster
```
In this case `pnd` will take care of all the complicated but non-interesting steps needed to run the simulation on the cluster, without the user needing to change anything.

Crucially, continuing along this line of thought, it wouldn't really matter whether there is any version of the framework installed locally, as long as the `pnd` command is available.
Even though this functionality is not yet available, the transition to a cloud-based model, with instances of simulation machines running remotely, would be almost completely seamless for the user, thanks to the high-level semantics provided by `pnd`.

The commonly identified simulation phases are: `sim`, `digi`, `reco`, `pid` and `ana`. `pnd` is aware of this, and can perform simple heuristics based on e.g. the filename of the requested macro.
```sh
# will run the macro from the current directory with "ana" in the filename (prompting for ambiguities)
$ pnd run -p/--phase ana

$ pnd run fullsim # will run macros from the current directory sequentially, according to the specified order
```

## Simulation parameters

### Current status
Values of simulation parameters are hard-coded in the macros or Tasks used to steer the simulation.

### Issues
- Macros are intended for interactive use, so this is somehow matching their intended purpose. However, running a macro with different values of one or more parameters is a very common task. If the value is written (hard-coded) in the macro, it must be changed manually at every interaction, as there is no way of doing so automatically
- Values set in Tasks are even more problematic, since changing them requires their re-compilation across every version of the framework.

### Proposed solutions

- Parameters are kept separate from macros/Tasks. Macros/Tasks should describe what operations are performed and how to use the values, while the values themselves should be stored separately, and loaded in the macro with a dedicated object.

In file `macro.C`:
```c++
using std::string;
auto sett = Settings("settings.yaml");
FairRunAna *fRun = new FairRunAna();

fRun->SetOptions(
    sett.get<float>("evtgen_beam_momentum"),
    sett.get<string>("file_evtgen_dec"),
    sett.get<int>("n_events"),
);
```

In file `settings.yaml`:
```yaml
evtgen_beam_momentum: 6.998
file_evtgen_dec: process1.evt
n_events: 10000
```

Launching the macro:
```sh
$ pnd run -m macro.C --settings=settings.yaml
```

The upside of this is that parameters can be manipulated and stored programmatically, allowing a wide range of useful possibilities:
- Specify ranges programmatically, i.e. `evtget_beam_momentum: {range: [5.3, 6.1, 6.9]}` would generate three simulations with those values for `evtgen_beam_momentum`
- Interpreting parameters as metadata associated with the simulation output

In addition, it would be desirable to be able to edit parameter at the command line so that they overwrite parameters in the file with the same key (with semantics similar to `dict.update()` in Python):
```sh
$ pnd run -m macro.C --settings=settings.yaml --evtgen_beam_momentum=4.334  # simulation runs with 4.334 as beam momentum
```
To allow this in an unambiguous way, `pnd` generates a second file with the merged parameter map.
The file is not supposed to be edited directly by the user, and is loaded in the macro by not specifying arguments to the constructor:
```c++
Settings sett;
```

## Debugging
In any user-facing software project without a dedicated team for user management, developers will inevitably find themselves dividing their time between working on the code and assisting the users.
`pnd` can help in this regard, by offering an automated way of collecting information about the user's environment without the user needing to know what exactly, and how, to collect it.

In the simplest cases, by telling the user to run
```sh
$ pnd info --troubleshoot/--detailed/--system
```
and paste the output on the technical support forum, the interaction between supporting developer and user is much more efficient.
This is a perfect example of the advantages provided according to the "API, not tutorials" principle.

The same principle can be used to share the current environment (macro, framework version, parameter files) with another user, to perform cross-checks or report potential bugs:
```sh
$ pnd env collect --name=date # creates a zip file with the date, containing all necessary files (overridable by the user)
```
A modification of the same functionality is used for programmatic persistence of the simulation results, i.e. storing the output data together with all files and information needed to maximize chances of reproducibility.

## Generator-level studies
Even if a full simulation of the detector is needed in most cases, a common use case is to study processes at the generator level, i.e. studying the raw properties of the events as they appear before being transported and modified by the response of the detector. This is useful to perform quick studies of e.g. the kinematical distributions of the particles involved in the process, since the simulation run time at the generator level is much faster than with the full detector simulation.

### Issues
From the point of view of the user, these two tasks are two sides of the same coin, especially considering that the simulation parameters needed for a generator-level study are a subset of those needed for a full simulation. However, practically speaking, even though the same generator (EvtGen) is used for generator-level studies and in the initial phase of full simulations, the concrete workflow is very different. The user-facing part of the generator is written independently from the macro/Task system, and is invoked as a separate executable, `EvtGenStandalone`.

### Proposed solutions
An ideal solution would be to act at a deeper level, by exposing a standalone EvtGen workflow from within a macro or Task similarly to the full simulation. However, through `pnd`, the need for this more radical refactoring can be obviated by offering a common API on top of an adaptation layer:

```sh
# for the full simulation
$ pnd run fullsim --settings=settings.yml -d/--decfile=process-foo.dec -p/--pdtfile=particles1

# for the generator-level only: wraps the concrete path to the EvtGenStandalone executable, transparently to the user
$ pnd run gen --settings=settings.yml -d/--decfile=process-foo.dec -p/--pdtfile=particles1
```
A post-processing hook of the `pnd run gen` command can ensure that the output is in the same format as the `fullsim` output.
Again, once the standalone EvtGen refactoring is performed, the implementation of `pnd run gen` can be modified without changes to its user-facing API.

# Implementation

## Language

The programming language chosen for running `pnd` is Python.
This choice is strongly motivated by the almost perfect overlap between the implementation requirements and Python's strong suits.
In particular, the large ecosystem of available libraries makes the development very convenient since most of the lower-level functionality is already available, and developers can concentrate on integrating the various elements and implementing the domain logic.
The readability and the increasing popularity among scientists also are relevant, since the core idea of `pnd` is to be developed continuously to develop new functionality.
In any case, the choice of the language remains an implementation detail, and as long as the API is kept stable, users would not be affected by a change in this regard.

## Libraries

One of the most important choice for the development of `pnd` is the module managing the CLI functionality. At the moment, three choices are being considered:

- Invoke
- argh
- click

The final choice will concern mostly developers, but there are non-trivial variations in user-facing functionality that should not be neglected: e.g. in Invoke the subcommand namespaces are divided by `.`.
