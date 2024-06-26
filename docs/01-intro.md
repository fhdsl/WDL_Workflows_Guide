

# Introduction to WDL

## Introduction

Welcome to building your first WDL workflow! This guide will help you strategically develop and scale up a WDL workflow that is iterative, reproducible, and efficient in terms of time and resource used.

To make sure that we are on the same page, this guide assumes that you are able to run a WDL on a computing engine of your choice, such as Cromwell, miniwdl, or a cloud computing environment such as Terra, AnVIL, or Dockstore. This guide also assumes that you have a beginner's understanding of the WDL syntax, and we will link out to additional resources to fill in the knowledge gap as needed! If you have never seen the WDL language in action, a great place to start is [OpenWDL docs](https://docs.openwdl.org/en/stable/) -- it teaches you the basic syntax and showcases WDL features via concrete examples.

## Review of basic WDL syntax

We will do some review of the WDL syntax. A WDL workflow consists of at least one task.

<!-- resources/basic_01.wdl -->



```         
version 1.0

task do_something {
    command <<<
        exit 0
    >>>
}

workflow my_workflow {
    call do_something
}
```

A workflow, and the tasks it calls, generally has inputs.

<!-- resources/basic_02.wdl -->

```         
version 1.0

task do_something {
    input {
        File fastq
    }
    command <<<
        exit 0
    >>>
}

workflow my_workflow {
    input {
        File fq
    }
    call do_something {
        input:
            fastq = fq
    }
}
```

The input `fq` is defined to be a File variable type. WDL supports various variable types, such as String, Integer, Float, and Boolean. For more information on types in WDL, we recommend [OpenWDL's documentation on variable types](https://docs.openwdl.org/en/stable/WDL/variable_types/).

To access a task-level input variable in a task's command section, it is usually referenced using \~{*input_name*} notation, such as (`~{fastq}`) in the example below. To access a workflow-level variable in a workflow, it is referenced just by its name without any special notation, such as `fq` in the example below. To access a workflow-level variable in a task, it must be passed into the task as an input.

<!-- resources/basic_03.wdl -->

```         
version 1.0

task do_something {
    input {
        File fastq
        String basename_of_fq
    }
    command <<<
        echo "First ten lines of ~{basename_of_fq}: "
        head ~{fastq}  ## example of referring to task-level input
    >>>
}

workflow my_workflow {
    input {
        File fq
    }
    
    String basename_of_fq = basename(fq)
    
    call do_something {
        input:
            fastq = fq,  ## example of referring to workflow-level input
            basename_of_fq = basename_of_fq
    }
}
```

Tasks and workflows also typically have outputs. The task-level outputs can be accessed by the workflow or any subsequent tasks. The workflow-level outputs represent the final output of the overall workflow.

<!-- resources/basic_04.wdl -->

```         
version 1.0

task do_something {
    input {
        File fastq
        String basename_of_fq
    }
    command <<<
        echo "First ten lines of ~{basename_of_fq}: " >> output.txt
        head ~{fastq} >> output.txt
    >>>
    output {
        File first_ten_lines = "output.txt"  ## output variable for task
    }
}

workflow my_workflow {
    input {
        File fq
    }
    
    String basename_of_fq = basename(fq)
    
    call do_something {
        input:
            fastq = fq,
            basename_of_fq = basename_of_fq
    }
    
    output {
        File ten_lines = do_something.first_ten_lines ## referring to task output here
    }
}
```

## Using JSONs to control workflow inputs

Running a WDL workflow generally requires two files: A `.wdl` file, which contains the actual workflow, and a `.json` file, which provides the inputs for the workflow.

In the example we showed earlier, the workflow takes in a file referred to by the input variable `fq`. This needs to be provided by the user. Typically, this is done with a JSON file. Here's what a JSON file for this workflow might look like:

<!-- resources/basic_04.json -->

```         
{
    "my_workflow.fq": "./data/example.fq"
}
```

JSON files consist of key-value pairs. In this case, the key is `"my_workflow.fq"` and the value is the path `"./data/example.fq"`. The first part of the key is the name of the workflow as written in the WDL file, in this case `my_workflow`. The variable being represented is referred to its name, in this case, `fq`. So, in the above example, the file located at the path `./data/example.fq` is being input as a variable called `fq` into the workflow named `my_workflow`.

Files aren't the only type of variable you can refer to when using JSONs. Here's an example JSON for every common WDL variable type.

<!-- resources/variables.json -->

```         
{
    "some_workflow.file": "./data/example.fq",
    "some_workflow.string": "Hello world!",
    "some_workflow.integer": 1965,
    "some_workflow.float": 3.1415,
    "some_workflow.boolean": true,
    "some_workflow.array_of_files": ["./data/example01.fq", "./data/example02.fq"]
}
```

::: {.notice data-latex="notice"}
Resources:

<ul>

<li>For more basic examples of WDL workflow with a single task, we recommend [OpenWDL's documentation on WDL's base structure](https://docs.openwdl.org/en/stable/WDL/Base_structure/).</li>

<li>For more information on types in WDL, we recommend [OpenWDL's documentation on variable types](https://docs.openwdl.org/en/stable/WDL/variable_types/).</li>

<li>If you are having difficulty writing valid JSON files, considering using <https://jsonlint.com/> to check your JSON for any errors.</li>

</ul>
:::


## Running WDL via a computing engine

In order to run a WDL workflow, we need a *computing engine* to execute it. The two most popular WDL executors are [miniwdl](https://miniwdl.readthedocs.io/en/latest/index.html) and [Cromwell](https://cromwell.readthedocs.io/en/stable/). Both computing engines can run WDLs on multiple configurations:  

- A local machine
- A High Performance Computing (HPC) Cluster
- A Cloud Computing backend (such as AWS/Terra/DNAnexus). 

If you are trying to run WDL at Fred Hutch Cancer Center's HPC system, you should use the PROOF executor. 

If you are computing on a HPC or the Cloud, you should find the best practice of running a WDL computing engine based on your institution's information technology system. 

If you are computing locally on your computer, below is a short guide on how to set that up. 


<details>

<summary><b>How to run simple workflows locally</b></summary>


Not every WDL workflow will run well on a laptop, but it can be helpful to have a basic setup for testing and catching simple syntax errors. Let's quickly set up a WDL executor to run our WDLs.

 In this course, we will be using miniwdl, but everything in this course will also be compatible with Cromwell unless explicitly stated otherwise. Additionally, almost all WDLs use Docker images, so you will also need to install Docker or a Docker-like alternative.

**Installing Docker and miniwdl is not required to use this course.** We don't want anybody to get stuck here! If you already have a method for submitting workflows, such as Terra, feel free to use that for this course instead of running workflows directly on your local machine. If you don't have any way of running workflows at the moment, that's also okay -- we have provided plenty of examples for following along.

### Installing Docker

**Note: Although Docker's own docs recommend installing Docker Desktop for Linux, [it has been reported](https://github.com/dockstore/dockstore/issues/5135) that some WDL executors work better on Linux when installing only Docker Engine (aka Docker CE).** To install Docker on your machine, follow the instructions specific to your operating system [on Docker's website](https://docs.docker.com/get-docker/). To specifically install only Docker Engine, [use these instructions instead](https://docs.docker.com/engine/install/).

If you are unable to install Docker on your machine, Dockstore (not affiliated with Docker) [provides some experimental alternatives](https://docs.dockstore.org/en/stable/advanced-topics/docker-alternatives.html). Dockstore also provides [a comprehensive introduction to Docker itself](https://docs.dockstore.org/en/stable/getting-started/getting-started-with-docker.html?highlight=engine#where-can-i-run-docker), including how to write a Dockerfile. Much of that information is outside the scope of this WDL-focused course, but it may be helpful for those looking to eventually create their own Docker images.

### Installing miniwdl

miniwdl is based on Python. If you do not already have Python 3.6 or higher installed, [you can install Python from here](https://www.python.org/downloads/).

Once Python is installed on your system, you can run `pip3 install miniwdl` from the command line to install miniwdl. For those who prefer to use conda, use `conda install -c conda-forge miniwdl` instead. Once miniwdl is installed, you can verify it works properly by running `miniwdl run_self_test`. This will run a built-in hello world workflow.

For more information, see [miniwdl's GitHub repository](https://github.com/chanzuckerberg/miniwdl).

### Launching a workflow locally with miniwdl

The generic method for running a WDL with miniwdl is the following:

```         
miniwdl run [path_to_wdl_file] -i [path_to_inputs_json]
```

If you have successfully installed miniwdl, create the following WDL file and name it greetings.wdl:

```         
version 1.0

task greet {
    input {
        String user
    }
    command <<<
        echo "Hello ~{user}!" > greets.txt
    >>>
    output {
        String greeting = read_string("greets.txt")
    }
}

workflow my_workflow {
    input {
        String username
    }
    call greet {
        input:
            user = username
    }
}
```

Next, use this JSON file (or create one of your own) to provide the string that the workflow expects, and call the JSON file greetings.json:

```         
{
    "my_workflow.username": "Ash"
}
```

On the command line, run the following:

```         
miniwdl run greetings.wdl -i greetings.json
```

Once the task completes, you should see something like this in your command line:

```         
[timestamp] wdl.w:my_workflow finish :: job: "call-greet"
[timestamp] wdl.w:my_workflow done
{
  "dir": "[working directory]/[timestamp]_my_workflow",
  "outputs": {
    "my_workflow.greet.greeting": "Hello Ash!"
  }
}
```

Where [timestamp] is the date and time that you are running the workflow, and [working directory] is the working directory that you are running the workflow from. For example:

```         
2023-12-27 13:54:12.209 wdl.w:my_workflow finish :: job: "call-greet"
2023-12-27 13:54:12.210 wdl.w:my_workflow done
{
  "dir": "/Users/ash/github/WDL_Workflows_Guide/resources/20231227_135400_my_workflow",
  "outputs": {
    "my_workflow.greet.greeting": "Hello Ash!"
  }
}
```

### Troubleshooting

#### DockerException

If you are seeing a verbose error message that begins with text like this:

```         
2023-12-27 13:43:37.525 wdl.w:my_workflow.t:call-greet task greet (greetings.wdl Ln 3 Col 1) failed :: dir: "/Users/sammy/github/WDL_Workflows_Guide/resources/20231227_134337_my_workflow/call-greet", error: "DockerException", message: "Error while fetching server API version: ('Connection aborted.', FileNotFoundError(2, 'No such file or directory'))", traceback: ["Traceback (most recent call last):", "  File \"/Library/Frameworks/Python.framework/Versions/3.11/lib/python3.11/site-packages/urllib3/connectionpool.py\", line 790, in urlopen", "    response = self._make_request(", "               ^^^^^^^^^^^^^^^^^^^", "  File \"/Library/Frameworks/Python.framework/Versions/3.11/lib/python3.11/site-packages/urllib3/connectionpool.py\",
```

This is likely caused by miniwdl being unable to connect to Docker Daemon, the underlying technology that runs Docker images. This is necessary with miniwdl even though our example WDL does not specify a Docker image. Make sure you have Docker installed correctly, and make sure Docker is actively running on your machine. If you installed Docker Desktop, simply opening the Docker Desktop app should start Docker Engine. If you installed Docker without Docker Desktop, running `dockerd` in your command-line should start it. Be aware that starting the Docker Daemon may take a few minutes.

#### Missing required inputs

If you forget to add `-i greetings.json` to your call, you will see something like this:

```         
my_workflow (greetings.wdl)
---------------------------

required inputs:
  String username

outputs:
  String greet.greeting

missing required inputs for my_workflow: username
```

You may also see this error if you remember to include a JSON file, but it is missing a required input.

#### Check JSON input

If you see an error message like this:

```         
check JSON input; unknown input/output: greetings.username
```

Double-check your input JSON. The first part of your JSON's keys refer to the name of the workflow in the WDL file, not the filename of the WDL itself. Even though our WDL is saved as `greetings.wdl`, within that file, the workflow is named `my_workflow`. This means that the input JSON must say `"my_workflow.username"`, not `"greetings.username"`.

Other common issues with JSON files are mistyping input variables (such as `"my_workflow.ussername"`) or forgetting to enclose strings in quotation marks. When in doubt, try using <https://jsonlint.com/> to check your input JSON, and double-check the name of your input variables.

</details>


<br>
<br>

<iframe src="https://docs.google.com/forms/d/e/1FAIpQLSeEKGWTJOowBhFlWftPUjFU8Rfj-d9iXIHENyd8_HGS8PM7kw/viewform?embedded=true" width="640" height="886" frameborder="0" marginheight="0" marginwidth="0">Loading…</iframe>
