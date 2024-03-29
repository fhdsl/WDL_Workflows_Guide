

# Appendix: Backends and Executors
Generally speaking, WDL workflows are quite portable thanks to their usage of Docker images to maintain software dependencies. However, the executor used to run WDLs and what backend they are being run upon can lead to specific scenarios where minor tweaks to your WDL are necessary to ensure portability.

## Commonly used runtime attributes
Runtime attributes do not behave the same on all platforms. Here are how some of the most commonly used runtime attributes work on some of the most common WDL setups.

| Attribute      | Fred Hutch HPC                                                                               | Local Cromwell                    | Local miniwdl                     | Terra                                                                                       |
|----------------|----------------------------------------------------------------------------------------------|-----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------|
| bootDiskSizeGB | n/a                                                                                          | n/a                               | n/a                               | Request a disk of this size to boot the Docker image (useful for very large Docker images)  |
| cpu            | Reserve at most this many cores                                                              | n/a                               | Reserve at most this many cores   | Request a minimum of this many cores (scales with memory)                                   |
| disks          | n/a                                                                                          | n/a                               | n/a                               | Request this much disk size - soft requirement, if not specified, will request 10 GB        |
| docker         | Run the task in this Docker image                                                            | Run the task in this Docker image | Run the task in this Docker image | Run the task in this Docker image                                                           |
| memory         | Maximum amount of memory to use                                                              | n/a                               | Maximum amount of memory to use   | Minimum amount of memory to use (scales with CPU)                                           |
| preemptible    | n/a                                                                                          | n/a                               | n/a                               | Attempt running on a preemptible instance this many times, then switch to a non-preemptible |
| walltime       | How much  [walltime](https://en.wikipedia.org/wiki/Elapsed_real_time)  to request for a task | n/a                               | n/a                               | n/a                                                                                         |

## General advice
* Consider using more runtime attributes, not fewer. miniwdl and Cromwell will ignore runtime attributes as necessary, so including a runtime attribute that only applies to a particular backend will not harm portability on other backends.
* The Dockstore CLI, as of v1.15, uses Cromwell to run WDLs. Advice about Cromwell will therefore also apply to the Dockstore CLI.
* If running a workflow with a scattered task on a local compute, consider using miniwdl instead of Cromwell.
* Whether you are using miniwdl or Cromwell locally, make sure Docker has enough resources to be able to download and run the Docker images specified in your WDL tasks.
* Workflow systems with a UI like Terra may become unresponsive if you run a task scattered more than 1000x. Outputs may need to be interacted with using an API specific to that backend.

## Executor-specific notes

### Cromwell
Cromwell running on a local machine has a tendency to heavily use system resources, especially when running scattered tasks. Sometimes, it will use too many resources at once. When this happens, your workflow's tasks will tend to fail with exit code 137, or you will see hints about running out of memory in the logs of your tasks. You may also observe Docker becoming unresponsive. You can fix Docker by restarting Docker or your machine, but you will likely want to prevent this issue rather than keep having to restart Docker. To prevent this, modify `concurrent-job-limit` for your backend in the Cromwell configuration file.

Other notes:

* Cromwell runs as a jar file, so it is very portable and does not need to be "installed" provided you have a modern Java runtime environment.

* If you just want to check workflows are valid, you can run `womtool` as a jar file (available on Cromwell's GitHub page). This will check not only the WDL file you pass it, but also any WDLs it imports.

* Cromwell does not use call caching on most backends, but it is the default on Terra. For non-Terra backends, it can be enabled in the Cromwell configuration file.

* Cromwell supports the `gpu` and `disks` runtime attributes on certain backends. If using the `gpu` runtime attribute, make sure your task is set up correctly to properly use this resource.

* Generally speaking, Cromwell will be able to directly modify input files by default without causing permission errors.

### miniwdl
The authors of this course have informally observed that miniwdl appears to be "safer" in terms of using up too many resources than Cromwell. However, if you are sharing a compute with other users, it is still worth looking at the miniwdl configuration file and limiting how many Docker images you spin up at the same time.

Other notes:

* miniwdl is a Python program, and it must be installed via `pip` or `pip3` to use.

* miniwdl's equivalent to `womtool` is `miniwdl check`, but it also includes `shellcheck` to check the command section of your tasks.

* miniwdl supports call caching, but it is turned off by default.

* miniwdl does not support the `gpu` or `disks` runtime attributes and will ignore them if present in a task's runtime section.

* By default, miniwdl does not duplicate input files. If your workflow only needs to read input files, this helps save disk space, but if your workflow directly modifies input files, this can result in permission errors. A simple fix is to run miniwdl with the `--copy-input-files` flag.

## Backend-specific notes

### GCP
#### Disk space
Running Cromwell on GCP is one of the few times that the `disks` runtime attribute is a soft requirement. Due to how Cromwell works, it must request a certain amount of disk space from GCP before running the task's command section. In other words, you must know roughly how much disk space your task will use before it runs. If not specified in the runtime attribute of a task, Cromwell will request 10 GB of SSD space.

A helpful way to handle disk space in WDLs you anticipate will be run on GCP/Terra is to make disk size a function of the size of your task inputs. The task inputs and any private variable declarations made outside of the command section are all calculated before Cromwell requests a disk from GCP, so you can use this section to define a disk size function. For example:

<!-- TODO: make seperate file in resources and miniwdl check it --->
```
version 1.0

task do_something {
    input {
        Array[File] some_array_of_files
        File one_file
    }
    
    Int predicted_disk_space = ceil(size(some_array_of_files), "GB") + ceil(size(one_file), "GB") + 1
    # size(x, "GB") returns a float representing the size of x in gigabytes
    # ceil() is a WDL built-in function that rounds a float up into an integer
    
    command <<<
    pass
    >>>
    
    runtime {
        docker: "ubuntu-latest"
        disks: "local-disk " + predicted_disk_space " HDD"
    }
}
```

Common pitfalls:

* Not wrapping `size()` in `ceil()` -- the `disks:` runtime attribute requires an integer, not a float

* Not using `"GB"` when calling `size()` -- if you don't specify units for `size()` it will return bytes!

* Forgetting the space between "local-disk" and the integer (or the integer and "HDD"/"SSD")

#### Preemptibles
Preemptible machines are an excellent way to save money when running workflows. A preemptible machine is a Google Cloud machine that is significantly cheaper (often less than half the price) than a standard one, at the cost of potentially being stopped suddenly. When running a task on a preemptible machine using Cromwell, if the preemptible is preempted (stopped suddenly), Cromwell will automatically retry the task. This does mean that in a worst case scenario, such a task could take about twice as long to run as normal and end up slightly more expensive, so you will want to weigh the costs and benefits. As a general rule of thumb, if you expect a task to take less than 2 hours, it is usually worth trying to use preemptible machines.

### HPCs
It is difficult to provide specific advice on HPCs, as they can vary greatly. Some general notes:

* Some HPCs do not support the use of Docker due to security concerns

  * HPCs that do not allow the use of Docker may be able to run WDLs using alternative container technologies such as podman or rootless Docker

* Some HPCs will use `disks` to determine which disk to run on, which can be useful for managing disk space

#### Fred Hutch HPC
* The Fred Hutch HPC supports the use of Docker, so the `docker` runtime attribute works as expected

* The `memory` and `gpu` runtime attributes are ignored, but `cpu` works as expected

* The Fred Hutch HPC supports the use of multiple JSON files going into the same workflow
