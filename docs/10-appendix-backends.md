

# Appendix: Backends and Executors

Generally speaking, WDL workflows are quite portable thanks to their usage of Docker images to maintain software dependencies. However, the executor used to run WDLs and what backend they are being run upon can lead to specific scenarios where minor tweaks to your WDL are necessary to ensure portability.

## Commonly used runtime attributes

Runtime attributes do not behave the same on all platforms. Here are how some of the most commonly used runtime attributes work on some of the most common WDL setups.

| Attribute                                       | Fred Hutch HPC                               | Local Cromwell                    | Local miniwdl                     | Terra                                                                                       |
|---------------|---------------|---------------|---------------|---------------|
| bootDiskSizeGB                                  | n/a                                          | n/a                               | n/a                               | Request a disk of this size to boot the Docker image (useful for very large Docker images)  |
| cpu                                             | Reserve at most this many cores              | n/a                               | Reserve at most this many cores   | Request a minimum of this many cores (scales with memory)                                   |
| disks                                           | n/a                                          | n/a                               | n/a                               | Request this much disk size - soft requirement, if not specified, will request 10 GB        |
| docker                                          | Run the task in this Docker image            | Run the task in this Docker image | Run the task in this Docker image | Run the task in this Docker image                                                           |
| memory                                          | n/a                                          | n/a                               | Maximum amount of memory to use   | Minimum amount of memory to use (scales with CPU)                                           |
| preemptible                                     | n/a                                          | n/a                               | n/a                               | Attempt running on a preemptible instance this many times, then switch to a non-preemptible |
| walltime, partition, modules, dockerSL, account | See back-end notes for Fred Hutch HPC below. | n/a                               | n/a                               | n/a                                                                                         |

## General advice

-   Consider using more runtime attributes, not fewer. miniwdl and Cromwell will ignore runtime attributes as necessary, so including a runtime attribute that only applies to a particular backend will not harm portability on other backends.
-   The Dockstore CLI, as of v1.15, uses Cromwell to run WDLs. Advice about Cromwell will therefore also apply to the Dockstore CLI.
-   If running a workflow with a scattered task on a local compute, consider using miniwdl instead of Cromwell.
-   Whether you are using miniwdl or Cromwell locally, make sure Docker has enough resources to be able to download and run the Docker images specified in your WDL tasks.
-   Workflow systems with a UI like Terra may become unresponsive if you run a task scattered more than 1000x. Outputs may need to be interacted with using an API specific to that backend.

## Executor-specific notes

### Cromwell

Cromwell running on a local machine has a tendency to heavily use system resources, especially when running scattered tasks. Sometimes, it will use too many resources at once. When this happens, your workflow's tasks will tend to fail with exit code 137, or you will see hints about running out of memory in the logs of your tasks. You may also observe Docker becoming unresponsive. You can fix Docker by restarting Docker or your machine, but you will likely want to prevent this issue rather than keep having to restart Docker. To prevent this, modify `concurrent-job-limit` for your backend in the Cromwell configuration file.

The default Cromwell configuration file,, [can be found on Cromwell's GitHub repo](https://github.com/broadinstitute/cromwell/blob/86/cromwell.example.backends/cromwell.examples.conf). To point Cromwell to a specific configuration file, run Cromwell with the flag `-Dconfig.file=` followed by the address of the config file to use.

If you are using the Dockstore CLI, [follow these instructions](https://docs.dockstore.org/en/stable/advanced-topics/dockstore-cli/local-cromwell-config.html?highlight=cromwell) to use a custom configuration file for Dockstore's version of Cromwell. Because the Docker lockup issue mentioned previously isn't uncommon, it is recommended to use a custom configuration file that limits `concurrent-job-limit` process on a local backend.

Other notes:

-   Cromwell runs as a jar file, so it is very portable and does not need to be "installed" provided you have a modern Java runtime environment.

-   If you just want to check workflows are valid, you can run `womtool` as a jar file (available on Cromwell's GitHub page). This will check not only the WDL file you pass it, but also any WDLs it imports.

-   Cromwell does not use call caching on most backends, but it is the default on Terra. For non-Terra backends, it can be enabled in the Cromwell configuration file.

-   Cromwell supports the `gpu` and `disks` runtime attributes on certain backends. If using the `gpu` runtime attribute, make sure your task is set up correctly to properly use this resource.

-   Generally speaking, Cromwell will be able to directly modify input files by default without causing permission errors, unlike miniwdl.

-   Cromwell uses a system known as [hog factors](https://cromwell.readthedocs.io/en/stable/cromwell_features/HogFactors/) to adjust how task order is prioritized.

### miniwdl

The authors of this course have informally observed that miniwdl appears to be "safer" in terms of using up too many resources than Cromwell. However, if you are sharing a compute with other users, it is still worth looking at the miniwdl configuration file and limiting how many Docker images you spin up at the same time.

The default miniwdl configuration file, which you can use as a template for setting up your own configuration file for miniwdl, [can be found on miniwdl's GitHub repo](https://github.com/chanzuckerberg/miniwdl/blob/v1.11.1/WDL/runtime/config_templates/default.cfg). To point miniwdl to a specific configuration file, run `miniwdl run` with the `--cfg` flag.

Other notes:

-   miniwdl is a Python program, and it must be installed via `pip` or `pip3` to use.

-   miniwdl's equivalent to `womtool` is `miniwdl check`, but it also includes `shellcheck` to check the command section of your tasks.

-   miniwdl supports call caching, but it is turned off by default.

-   miniwdl does not support the `gpu` or `disks` runtime attributes and will ignore them if present in a task's runtime section.

-   By default, miniwdl does not duplicate input files. If your workflow only needs to read input files, this helps save disk space, but if your workflow directly modifies input files, this can result in permission errors. A simple fix is to run miniwdl with the `--copy-input-files` flag.

## Backend-specific notes

### HPCs

It is difficult to provide specific advice on HPCs, as they can vary greatly. Some general notes:

-   Some HPCs do not support the use of Docker due to security concerns

    -   HPCs that do not allow the use of Docker may be able to run WDLs using alternative container technologies such as podman or rootless Docker

-   Some HPCs will use `disks` to determine which disk to run on, which can be useful for managing disk space

#### Fred Hutch HPC

-   [PROOF](https://sciwiki.fredhutch.org/datascience/proof/) is a Shiny frontend that launches Cromwell Server and lets you manage your jobs from a GUI. User guide can be found [here](https://sciwiki.fredhutch.org/dasldemos/proof-how-to/).

-   The Fred Hutch HPC supports the use of multiple JSON files going into the same workflow

-   The Fred Hutch HPC supports the use of Docker, so the `docker` runtime attribute works as expected

-   `memory` and `gpu` runtime attributes are ignored, but `cpu` works as expected. Here is [some information](https://sciwiki.fredhutch.org/scicomputing/compute_jobs/#memory) on approximating memory use.

-   `walltime` attribute is a string ("HH:MM:SS") that specifies how much time you want to request for the task. Can also specify \>1 day, e.g. "1-12:00:00" is 1 day+12 hours. The amount of walltime you can request depends on which cluster partition you want to use. More info [here](https://sciwiki.fredhutch.org/scicomputing/compute_jobs/#wall-time).

-   `partition` attribute specifies the [cluster partition](https://sciwiki.fredhutch.org/compdemos/gizmo_partition_index/)to use. The default is "campus-new".

-   `modules` attribute is a a space-separated list of the [environment modules](https://sciwiki.fredhutch.org/scicomputing/compute_environments/) you'd like to load (in that order) prior to running the task. For example, "modules:"GATK/4.2.6.1-GCCcore-11.2.0 SAMtools/1.16.1-GCC-11.2.0", the GATK module will be loaded first, followed by the SAMtools module.

-   `account` attribute allows users connected to multiple PI accounts to specify which account to use for each task, to manage cluster allocations. An example of the value for this variable is "paguirigan_a", following the PI's "lastname_firstNameInitital" pattern.

### GCP/Terra

GCP powers Terra, which is a Cromwell-based platform that is commonly used for running WDL workflows, but it can also be used on its own. Some special considerations need to be taken into account when designing workflows for compatibility on this backend.

#### Preemptibles

Preemptible machines are an excellent way to save money when running workflows. A preemptible machine is a Google Cloud machine that is significantly cheaper (often less than half the price) than a standard one, at the cost of potentially being stopped suddenly. When running a task on a preemptible machine using Cromwell, if the preemptible is preempted (stopped suddenly), Cromwell will automatically retry the task. This does mean that in a worst case scenario, such a task could take about twice as long to run as normal and end up slightly more expensive, so you will want to weigh the costs and benefits. As a general rule of thumb, if you expect a task to take less than 2 hours, it is usually worth trying to use preemptible machines.

#### Disk space

Running Cromwell on GCP is one of the few times that the `disks` runtime attribute is a soft requirement. Due to how Cromwell works, it must request a certain amount of disk space from GCP before running the task's command section. In other words, you must know roughly how much disk space your task will use before it runs. If not specified in the runtime attribute of a task, Cromwell will request 10 GB of SSD space.

A helpful way to handle disk space in WDLs you anticipate will be run on GCP/Terra is to make disk size a function of the size of your task inputs. The task inputs and any private variable declarations made outside of the command section are all calculated before Cromwell requests a disk from GCP, so you can use this section to define a disk size function. For example:

```         
version 1.0

task do_something {
    input {
        Array[File] some_array_of_files
        File one_file
    }
    
    Int predicted_disk_space = ceil(size(some_array_of_files, "GB")) + ceil(size(one_file, "GB")) + 1
    # size(x, "GB") returns a float representing the size of x in gigabytes
    # ceil() is a WDL built-in function that rounds a float up into an integer
    
    command <<<
    pass
    >>>
    
    runtime {
        docker: "ubuntu-latest"
        disks: "local-disk " + predicted_disk_space + " HDD"
    }
}
```

Common pitfalls:

-   Not wrapping `size()` in `ceil()` -- the `disks:` runtime attribute requires an integer, not a float

-   Not using `"GB"` when calling `size()` -- if you don't specify units for `size()` it will return bytes!

-   Forgetting the space between "local-disk" and the integer (or the integer and "HDD"/"SSD")

As a more in-depth example of how to use disk space, see the following WDL:

```         
version 1.0

task autoscale_disk_size_for_basically_anything {
    input {
        File ref_genome       # required file
        File? ref_alt         # optional file
        Array[File] fastqs    # required array of files
        File tarball_bams     # uncompressed tarball of bam files
        File compressed_vcfs  # compressed (tar.gz) tarball of VCFs
        Int? addl_disk_gb     # user-defined integer 
    }

    # ref_genome always exists, so we can size() on it safely. size() returns a float,
    # but some backends require disk size is an integer, so we use ceil() to turn
    # that float into an integer. ceil() always rounds up.
    # You should ALWAYS explictly defining which unit you are using with size(). I 
    # recommend sticking with GB. If you do not define a unit, it is easy to accidentally
    # request ludricious amounts of disk space, eg:
    # 1 GB --> 1073741824 bytes --> 1073741824 GB --> 1.07 exabytes
    Int disk_ref = ceil(size(ref_genome, "GB"))

    # ref_alt does not always exist, and size() will break if you try to use it on an
    # undefined file. To prevent this, we use select_first to fall back on the always-
    # defined ref_genome file if ref_alt is undefined.
    Int disk_alt = ceil(size(select_first([ref_alt, ref_genome]), "GB"))

    # Of course, this means that if ref_alt is undefined, then we don't really need 
    # disk_alt, because the size of ref_genome is already covered by disk_ref.
    Int disk_both_refs = if defined(ref_alt) then disk_alt + disk_ref else disk_ref

    # You might be wondering why we don't just do this:
    #     Int disk_alt = if defined(ref_alt) then ceil(size(ref_alt, "GB")) else 0
    # This is synatically valid, but generally speaking it is best to avoid running
    # WDL built-in functions (except select_all, select_first, and defined) on optional
    # variables as much as possible. 

    # When dealing with arrays, size() will return the sum of all Files in the array.
    Int disk_fastqs = ceil(size(fastqs, "GB"))

    # If you are passing in a tarball, you will likely be expanding it. If so, consider
    # doubling the disk size to account for it in both states.
    Int disk_tarball = 2*ceil(size(tarball_bams, "GB"))

    # A compressed tar.gz is harder to predict. As a very rough rule of thumb, 10x should
    # be enough. If you are passing in a tar.gz made from an upstream task, you may want
    # to decrease this number based on the compression ratio you used.
    Int disk_compressed = 10*ceil(size(compressed_vcfs))

    # Because addl_disk_gb is an optional integer with no fallback, you cannot add it to 
    # another integer, eg, this will error as "Non-numeric operand to + operator":
    #    Int final_disk_size = disk_both_refs + disk_fastqs + disk_tarball + disk_compressed + addl_disk_gb
    # We have to use select_first() to coerce it into a not-optional integer first.
    Int disk_addl = select_first([addl_disk_gb, 0])

    # Finally, let's add everything up. (Recall that disk_both_refs accounts for ref_genome and
    # also ref_alt.)
    Int final_disk_size = disk_both_refs + disk_fastqs + disk_tarball + disk_compressed + disk_addl

    command <<<
    echo disk_ref: ~{disk_ref}
    echo disk_alt: ~{disk_alt}
    echo disk_both_refs: ~{disk_both_refs}
    echo disk_fastqs: ~{disk_fastqs}
    echo disk_tarball: ~{disk_tarball}
    echo disk_compressed: ~{disk_compressed}
    echo addl_disk_gb: ~{addl_disk_gb}
    echo final_disk_size: ~{final_disk_size}
    >>>

    runtime {
        # On GCP!Cromwell, if you do not define the units in the runtime attribute, this
        # will be interpreted as the size in GB. This is why it's so easy to accidentally
        # request the wrong amount of disk space if you do not define the units for size();
        # In one place WDL defaults to bytes but in the other it defaults to GB.
        disks: "local-disk " + final_disk_size + " HDD"
    }
}
```

<iframe src="https://docs.google.com/forms/d/e/1FAIpQLSeEKGWTJOowBhFlWftPUjFU8Rfj-d9iXIHENyd8_HGS8PM7kw/viewform?embedded=true" width="640" height="886" frameborder="0" marginheight="0" marginwidth="0">Loading…</iframe>

