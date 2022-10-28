

# Introduction to WDL
The Workflow Description Language (WDL) is a way to specify data processing workflows with a human-readable and -writeable syntax. 
WDL makes it straightforward to define analysis tasks, chain them together in workflows, and parallelize their execution. 
The language makes common patterns simple to express, while also admitting uncommon or complicated behavior; and strives to achieve portability not only across execution platforms, but also different types of users. 
Whether one is an analyst, a programmer, an operator of a production system, or any other sort of user, WDL should be accessible and understandable.

## openWDL

WDL was originally developed for genome analysis pipelines by the Broad Institute. 
As its community grew, both end users as well as other organizations using WDL for their own software, it became clear that there was a need to allow WDL to become a true community driven standard. 
The OpenWDL community has thus been formed to steward the WDL language specification and advocate its adoption.

There is ongoing work on WDL the specification, thus it has multiple versions.  Currently there are three versions to note:
- draft-2 - this version was the version that much of the Broad's public facing documentation and example workflows were written in. 
- 1.0 - this is a more recent version that is the most up to date version of the specification that Cromwell can interpret.  We use WDL 1.0 at the Hutch.
- 1.1 - this is an even more recent version that not all WDL engines support yet. 

We'll be using WDL 1.0 in this course but you can always check out the [openwdl repo](https://github.com/openwdl/wdl) for more information about tweaking these instructions for different versions of WDL.  

## WDL 1.0 Syntax


# Base structure
There are 5 basic components that form the core structure of a WDL script: 
* workflow
* task
* call
* command
* output

There are also some optional components you can use to specify **runtime** parameters (like environment conditions such as a Docker image), **meta** information like the task author and email, and **parameter_meta** descriptions of inputs and outputs -- but we're not going to worry about them right now.


Let's look at how the core components are structured in a minimal WDL script that describes a **workflow**called **myWorkflowName** and two tasks, **task_A** and **task_B** (the names can be anything you want and do not have to include the words 'task' or 'workflow'). 

```wdl
version 1.0
workflow myWorkflowName {
  input {
    File my_ref
    File my_input
    String name
  }
  call task_A {
    input: 
      ref = my_ref,
      in = my_input,
      id = name     
  }
  call task_B {
    input: 
      ref = my_ref,
      in = task_A.out
  }

task task_A {...}
task task_B {...}

```

To keep things  simple for now, we're assuming that any parameters, inputs and output filenames are hardcoded (meaning actual filenames and parameter values are written in the script itself), and there are no variables. We'll see in the next step how to add variables to this basic structure.

## Top-level components: workflow, call and task

At the top level, we define a `workflow` within which we make calls to a set of tasks. The tasks are actually defined outside of the workflow definition block while the `call` statements are placed inside of it.

The order in which the workflow block and task definitions are arranged in the script does not matter. Nor does the order of the call statements matter, as we'll see further on.

### call

The `call` component is used within the workflow definition to specify that a particular task should be executed. In its simplest form, a call just needs a task name.

Optionally, we can add a code block to specify input variables for the task. We can also modify the call statement to call the task under an alias, which allows the same task to be run multiple times with different parameters within the same workflow. This makes it very easy to reuse code. 

The order in which call statements are executed does not depend on the order in which they appear in the script; instead it is determined based on a graph of dependencies between task calls. This means that the program infers what order task calls should run in by evaluating which of their inputs are outputs of other task calls. 

#### Examples:
```wdl
# in it's simplest form
call my_task { }
```

```wdl
# with input variables
call my_task {
  input: 
    task_var1 = workflow_var1,
    task_var2 = workflow_var2, 
    ...
}
```

```wdl
# with an alias and input variables

call my_task as task_alias {
  input: 
    task_var1 = workflow_var1, 
    task_var2 = workflow_var2, 
    ...
}
```
### task

The task component is a top-level component of WDL scripts. It contains all the information necessary to "do something" centering around a command accompanied by definitions of input files and parameters, as well as the explicit identification of its output(s) in the output component. It can also be given additional (optional) properties using the runtime, meta and parameter_meta components.

Tasks are "called" from within the workflow definition, which is what causes them to be executed when we run the script. The same task can be run multiple times with different parameters within the same workflow, which makes it very easy to reuse code.

#### Example:
```wdl
task my_task {
  input { ... 
  }
  command <<< ... 
  >>>
  output { ... 
  }
}
```
### workflow

The `workflow` component is a required top-level component of a WDL script. It contains call statements that invoke task components, as well as workflow-level input definitions.

There are various options for chaining tasks together through call and other statements.

#### Example:
```wdl
workflow myWorkflowName {
  call my_task
}
```

## Core task-level components: command and output

If we look inside a task definition, we find its core components: the command that will be run, which can be any command line that you would run in a terminal shell, and an output definition that identifies explicitly which part of the command constitutes its output.

```wdl
task task_A {
  input {
    File ref
    File in
    String id
  }
  command <<<
    do_stuff -R ~{ref} -I ~{in} -O ~{id}.ext
  >>>
  runtime {
    docker: "ubuntu:latest"
  }
  output {
    File out = "~{id}.ext"
  }
}
```
### command

The command component is a required property of a task. The body of the command block specifies the literal command line to run (basically any command that you could otherwise run in a terminal shell) with placeholders (e.g. \~{input_file}) for the variable parts of the command line that need to be filled in. All variable placeholders MUST be defined in the task input definitions.

#### Example:
```wdl
command <<<
  java -jar myExecutable.jar \
  INPUT=~{input_file} \
  OUTPUT=~{output_basename}.txt
>>>
```
### output

The output component is a (mostly) required property of a task. It is used to explicitly identify the output(s) of the task command for the purpose of flow control. 

The outputs identified here will be used to build the workflow graph, so it is important to include all outputs that are used as inputs to other tasks in the workflow.

Technically, output is not required for tasks that don't produce an output that is used anywhere else, like in the canonical "Hello World" example. But this is very rare, as most of the time when you are writing a workflow that actually does something useful, each task command will produce some sort of output. Because otherwise, why would you run it, right?

All types of variables accepted by WDL can be included here. The output definitions MUST include an explicit type declaration.

#### Example:
```wdl
output {
  File out = "~{output_basename}.txt"
}
```
As you can see the basic structure of a WDL script is fairly straightforward. In the next section, we're going to make it more realistic by adding variables instead of assuming that input and output names and all parameters are hardcoded.

Go to the next section: [Add Variables](./add_variables.md)

## Workflow Structure

```
## Warning in readLines(dest_file): incomplete final line found on 'resources/
## other_chapters/workflow.md'
```


# Workflow

The `workflow` component is a required top-level component of a WDL script. It contains `call` statements that invoke `task` components, as well as workflow-level input definitions.

There are various options for chaining tasks together using `call` and other statements. For more information, see [Add plumbing](add_plumbing.md).

## Example:
```wdl
version 1.0
workflow myWorkflowName {
    call task_A
}
```

### Inputs

# Specify inputs
The simplest way to specify values for the input variables (such as file names and parameters) to the commands in your WDL scripts is to hard-code them, i.e. write them in the script itself. However, doing so forces you to make a new copy and edit the inputs every time you want to run your script on a new batch of data -- which undermines the advantages of setting up a pipeline script in the first place.

A much better way to proceed is to specify all the values for the input variables that you want to be able to customize from run to run in a JSON file (a structured text format a bit like XML but better -- certainly more readable). Then all you need to do is create a new file of inputs for each new batch of data that you want to run through your pipeline. The execution engine will use that JSON file to fill in the values of inputs to commands in your script where appropriate.

Still, on the face of it you might think that putting together the JSON file of inputs (specifically, structuring it correctly and not forgetting any command's inputs) would be a tedious and/or daunting task, especially in a command-line-only world with no point-and-click GUI.

But fear not! Help is at hand. WDL comes with a utility function (in the `wdltool package`) that will parse your WDL script and automatically generate a template JSON file containing the appropriate input file and parameter definitions. All you need to do then is to populate a copy of the file with the actual values that you want to use in a given run of the pipeline. When you are ready to run your script on your chosen execution engine, you'll simply provide the inputs file along with the script.

## Generating the template JSON

To generate the template of `inputs` for your WDL script, simply call the wdltool inputs function on your script:
```java
java -jar wdltool.jar inputs myWorkflow.wdl > myWorkflow_inputs.json
```

This will create a file called `myWorkflow_inputs.json` that lists all the inputs to all the tasks in your script following the pattern below:
```
{
    "<workflow name>.<task name>.<variable name>": "<variable type>"
}
```
This saves you from having to compile a list of all the tasks and their variables manually! Pretty nifty, right?

*If you omit the `> myWorkflow_inputs.json` part of the command, the template content will be output to the terminal instead of being written to file.*

## Customizing the inputs file for a particular run

Every time you want to run the script on some new data or with some different parameters, you simply open this file (or better, a copy) in a text editor and replace the part on the right of the colon with the value that you want.

In case you're wondering, the `"<variable type>"` bit in the original template is just there to remind you what type of variable the task will expect to see. In the same spirit, we do recommend giving your tasks and variables names that will be meaningful when you're going through your inputs file filling in filenames and parameter values. Otherwise you'll find yourself having to refer back to the pipeline script itself often, as it's not possible to add comments in a JSON file.

### Example

Let's say the `myWorkflow.wdl` script describes a workflow called`myWorkflowName`. This workflow includes a task called `stepA` that takes two inputs, a File called `input_file` and a String called `sample_name`. The input template generated by the command above would look like this:
```json
{
    "myWorkflowName.stepA.input_file": "File",
    "myWorkflowName.stepA.sample_name": "String"
}
```

So to run this script on a file called `input.bam` where the sample name of interest is `NA12878`, you would change it to:
```json
{
    "myWorkflowName.stepA.input_file": "~/path/to/input.bam",
    "myWorkflowName.stepA.sample_name": "NA12878"
}
```

Woohoo, we're done preparing the script and its inputs! All we need to do now is run the command to execute the script on our preferred execution engine.

Go to the next section: [Execute!](./execute.md)

### Calls

```
## Warning in readLines(dest_file): incomplete final line found on 'resources/
## other_chapters/call.md'
```


# Call

The `call` component is used within the workflow body to specify that a particular task should be executed. In its simplest form, a call just needs a task name.

Optionally, we can add a code block to specify input variables for the task. We can also modify the call statement to call the task under an alias, which allows the same task to be run multiple times with different parameters in the same workflow. This makes it very easy to reuse code. See [Task aliasing](task_aliasing.md) for more information.

Note that the order in which call statements are executed does not depend on the order in which they appear if the script; instead it is determined based on a graph of dependencies between task calls. This means that the program infers what order task calls should run in by evaluating which of their inputs are outputs of other task calls. 

## Examples:

```wdl
# in its simplest form
call task_A

# with input variables
call task_A {
    input: 
        ref = my_ref,
        in = my_input,
        id = name
}

# with an alias and input variables
call task_A as my_task {
    input:
        ref = my_ref,
        in = my_input,
        id = name
}
```

### Outputs
Outputs to the workflow itself.  



### Tasks

```
## Warning in readLines(dest_file): incomplete final line found on 'resources/
## other_chapters/task.md'
```


# Task

The `task` component is a top-level component of WDL scripts. It contains all the information necessary to "do something" by using a `command` along with definitions of input files and parameters, as well as the explicit identification of its output(s) in the `output` component. It can also be given additional (optional) properties using the `runtime`, `meta` and `parameter_meta` components.

Tasks are "called" from within the` workflow`, which is what causes them to be executed when we run the script. The same `task` can be run multiple times with different parameters in the same workflow, which makes it very easy to reuse code. See [Task aliasing](task_aliasing.md) for more information.

## Example:

```wdl
task task_A {
  input {
    File ref
    File in
    String id
  }
  command <<<
    do_stuff -R ~{ref} -I ~{in} -O ~{id}.ext
  >>>
  runtime {
    docker: "ubuntu:latest"
  }
  output {
    File out = "~{id}.ext"
  }
}
```

Task aliasing.

# Task aliasing
*When you need to call a task more than once in a workflow, use task aliasing.* 

It would be tedious to copy-paste a task's definition and change the name each time you needed to use it again in the workflow. This method, termed [copy and paste programming](https://en.wikipedia.org/wiki/Copy-and-paste_programming), is simple enough up front but difficult to maintain in the long run. Imagine you found a typo in one of your tasks--you'd need to fix that typo in every single pasted task! However, using WDL's built-in task aliasing feature, you can call the same task code and assign it an alias. Then, following the principle of hierarchical naming, in order to access the output of an aliased task we use the alias, rather than the original task name.

![Diagram depicting the same task being used twice for different input samples. The first time the task is called, it is called with an alias "as firstInput", whereas the second time it is called, it is called with the alias "as secondInput". The output from the task the first time is used in a process StepB whereas the output from the task the second time is used in a process StepC.](../Images/task_alias.png)

To use an alias, we use the syntax `call taskName as aliasName`, as demonstrated in the code snippet below:
```wdl
call stepA as firstSample { 
  input: 
    in = firstInput 
}
call stepA as secondSample { 
  input: 
    in = secondInput 
}
call stepB { 
  input: 
  in = firstSample.out 
}
call stepC { 
  input: 
    in = secondSample.out 
}
```
## Generic example script

The workflow and its tasks altogether in a WDL script would look as follows:

```
version 1.0
workflow taskAlias {
  input {
    File firstInput
    File secondInput
  }
  call stepA as firstSample {
    input: 
      in = firstInput 
  }
  call stepA as secondSample {
   input: 
    in = secondInput
  }
  call stepB { 
    input: 
      in = firstSample.out 
  }
  call stepC {
   input: 
     in = secondSample.out
  }
}

task stepA {
  input {
     File in
  }
  command <<<
    programA I = ~{in} O = outputA.ext 
  >>>
  output { 
    File out = "outputA.ext" 
  }
}

task stepB {
  input {
   File in  
  }
  command <<<
    programB I = ~{in} O = outputB.ext 
  >>>
  output { 
    File out = "outputB.ext" 
  }
}

task stepC {
  input {
    File in  
  }
  command <<<
    programC I = ~{in} O = outputC.ext 
  >>>
  output { 
    File out = "outputC.ext" 
  }
}
```
## Concrete example

Let's take a look at this task aliasing concept inside a real-world example; using the GATK, this workflow separates SNPs and Indels into distinct vcf files using the same task, `select`, but two distinct aliased calls, `selectSNPs` and `selectIndels`. The distinct outputs of these calls are then hard filtered by separate tasks designed specifically for them, `hardFilterSNP` and `hardFilterIndel`, respectively.

![Diagram depicting how a raw VCF is used as input to the same task, "select" that is used with two different aliases; selectSNPs and selectIndels. The two separate outputs are then passed to two different tools, hardFileterSNPs and hardFilterIndels, respectively.](../Images/gatk_alias.png)
For this toy example, we have defined three tasks:

* **select** takes in a `String type`, specifying "SNP" or "Indel", and a `File rawVCF`, outputting a `File rawSubset` that contains only variants of the specified type.
* **hardFilterSNP** takes in a `File rawSNPs` and outputs a `File filteredSNPs`.
* **hardFilterIndels**takes in a `File rawIndels`and outputs a `File filteredIndels`.

## Concrete example script

The above workflow description would look like this in its entirety:

```wdl
version 1.0
workflow taskAliasExample {
  input {
    File rawVCFSample
  }
  call select as selectSNPs { 
    input: 
      type = "SNP", 
      rawVCF = "rawVCFSample"
  }
  call select as selectIndels {
    input: 
      type = "INDEL", 
      rawVCF = "rawVCFSample" 
  }
  call hardFilterSNP {
    input:
      rawSNPs = selectSNPs.rawSubset 
  }
  call hardFilterIndel { 
    input:
      rawIndels = selectIndels.rawSubset 
  }
}

task select {
  input {
    String type
    File rawVCF
  }
  command <<<
    java -jar GenomeAnalysisTK.jar \
      -T SelectVariants \
      -R reference.fasta \
      -V ~{rawVCF} \
      -selectType ~{type} \
      -o raw.~{type}.vcf
  >>>
  output {
    File rawSubset = "raw.~{type}.vcf"
  }
}

task hardFilterSNP {
  input {
    File rawSNPs
  }

  command <<<
    java -jar GenomeAnalysisTK.jar \
        -T VariantFiltration \
        -R reference.fasta \
        -V ~{rawSNPs} \
        --filterExpression "FS > 60.0" \
        --filterName "snp_filter" \
        -o .filtered.snps.vcf
  >>>
  output {
    File filteredSNPs = ".filtered.snps.vcf"
  }
}

task hardFilterIndel {
  input {
     File rawIndels
  }
  command <<<
    java -jar GenomeAnalysisTK.jar \
        -T VariantFiltration \
        -R reference.fasta \
        -V ~{rawIndels} \
        --filterExpression "FS > 200.0" \
        --filterName "indel_filter" \
        -o filtered.indels.vcf
  >>>
  output {
    File filteredIndels = "filtered.indels.vcf"
  }
}
```

For simplicity, we omitted the handling of index files, which has to be done explicitly in WDL. 


## Wiring Tasks Together



```
## Warning in readLines(dest_file): incomplete final line found on 'resources/
## other_chapters/add_variables.md'
```


# Add variables

In this context, **variables** are placeholders we write into the script instead of actual filenames and parameter values. We can then specify the filenames and values we want to use at runtime (meaning when we run the script) without modifying the script at all, which is very convenient. We don't have to use variables for everything, mind you -- for some parameters, it makes sense to hardcode the values if they're never going to change from run to run.

So let's look at how to include variables in our WDL script (we'll talk later about how to specify variable values at runtime). There are two different levels at which we might want to include variables: within an individual `task`, or at the level of the whole workflow, so that the variables will be available to any of the tasks it calls. We'll start with task-level variables because it's the simplest case, then build on that to approach workflow-level variables, which come with a few (very reasonable and totally not hard) complications. Over the course of this explanation, we'll touch on how variable values can be inherited from one task to the next, which is a preview for the next topic, on connecting tasks together.

## Adding task-level variables

Let's look at what an example task, `task_A`, actually contains in its `command` and `output` component blocks. 

```wdl
task task_A {
  input {
    File ref
    File in
    String id
  }
  command <<<
    do_stuff -R ~{ref} -I ~{in} -O ~{id}.ext
  >>>
  runtime {
    docker: "ubuntu:latest"
  }
  output {
    File out = "~{id}.ext"
  }
}
```

We made up an imaginary program called `do_stuff` which presumably does something more interesting than printing "Hello World". This program requires two files to be provided with the arguments `-R` and `-I`, respectively, and produces an output file that must be named using the argument `-O`. 

If we were to hardcode the values, we could just write the command line as we would run it in the terminal:

```
do_stuff -R reference.fa -I input.bam -O variants.vcf
```

To replace these hardcoded values with variables, we first have to *declare* the variables, which is a fancy way of saying we write their name and what type of value they stand for in the `input` block of the task. In the example, `task_A`, there are three inputs declared by `File ref`, `File in`, and `String id`.

Then we can insert the variable name within the command, at the appropriate place, within curly braces and prefaced by a tilde, as in `~{ref}`, `~{in}`, and `~{id}`.

Here for the value of `-O`, we use the variable to specify only a base name. The script will automagically *concatenate* this base name with the `.ext` file extension that we hardcoded, producing the complete output file name from `~{id}.ext`.

Finally, we identify any arguments of the `command` that we want to track as program outputs (in this case, the `-O` argument) and *declare* them by copying their assigned contents to the `output` block, as shown in the example. Notice that here too, we specify the variable type explicitly.

## Adding workflow-level variables

You've seen how variables are declared in tasks, but what about workflows?

```wdl
version 1.0
workflow myWorkflowName {
  input {
    File my_ref
    File my_input
    String name
  }
  call task_A {
    input: 
      ref = my_ref,
      in = my_input,
      id = name     
  }
  call task_B {
    input: 
      ref = my_ref,
      in = task_A.out
  }

task task_A {...}
task task_B {...}
```

Moving one level out to the body of the workflow, you see that we've *declared* a set of variables at the top. These declarations follow essentially the same rules as those inside a task. All we need to do now is connect these two levels so that arguments passed to the workflow can be used as inputs to the task.

To do so, we simply add those variables to the `call` function. This block simply contains an `input:` line, followed by additional lines that enumerate which workflow-level variables connect to which task-level variables.

We do something very similar for the second task we're calling in this workflow, `task_B`, with a key difference. We are using the output of `task_A` as the input to `task_B`. Conveniently, we can simply refer to that using a `task_name.output_variable syntax`. In this case, we use `task_A.out`.

Finally, we still need to know how to pass values to the workflow to populate all of those variables, don't we? Yes. Yes, we do.

You could hardcode the values when you declare the variables, of course, and for some parameters that you know will always be the same, it makes sense to do it that way. But there are certainly some variables you need to keep, well, variable, so you don't have to edit your script from run to run.

How that is done in practice is mostly up to the execution engine, not WDL. Pipelining systems typically use one of two main strategies to fulfill this need: either provide the values as part of the command line that launches the workflow execution, or provide a configuration file that lists all the desired values. In the Cromwell execution engine, we prefer to use a file of input definitions, as detailed in a later section, [Specify inputs](./specify_inputs.md).

---

Well, that wasn't so bad. In the next section, we'll look at how we can connect tasks through their inputs and outputs, plus some additional functions, to produce full-featured code without unnecessary complexity.

Go to the next section: [Add plumbing](./add_plumbing.md).

Or, learn more about [what variable types are available in WDL](./variable_types.md).



```
## Warning in readLines(dest_file): incomplete final line found on 'resources/
## other_chapters/add_plumbing.md'
```


# Add plumbing

In this section, you'll learn about the different plumbing options supported by WDL and how you can implement plumbing in your own workflows.

## What is plumbing?

By plumbing, we mean chaining tasks together to form sophisticated pipelines. So how do we do that in WDL?

With effortless grace, that's how.

### Simple connections

At this point, you know how to include multiple tasks in your workflow script. If you were paying attention to the section about variables, you even know how to connect the output of one task to the input of the next. That enables you to build **linear** or simple **branching and merging** workflows of any length, with single or **multiple inputs and outputs** connected between tasks without any other code than what we've already covered.

### Switching and iterating logic

In addition to those basic connection capabilities, you'll sometimes need to be able to switch between alternate pathways and iterate over sets of data, either in series or in parallel. For that, we're going to need some more code, but don't worry... you don't have to write any of it! Instead, we're going to take advantage of some handy features built into WDL syntax that make it easy to add this logic without a lot of effort. 

One of the features of this type in WDL is **scatter-gather** parallelism, which is a great way to speed up execution when you need to apply the same command to subsets of data that can be treated as independent units. 

You can also use **conditionals** in WDL, which allow you to choose under which conditions you want a block of code to be executed. 

### Efficiency through code re-use

Finally, you'll find you often need to do similar things in different contexts. Rather than copying the relevant code into several places -- which produces more code overall to maintain and update -- you can re-use the code smartly, either through [task aliasing](task_aliasing.md) if your problem involves running the same tool in different ways, or by importing workflows wholesale.

With all these options for assembling calls to tasks into workflows, we can build some fairly sophisticated data processing and analysis pipelines. Admittedly this can seem a bit overwhelming at first, but don't worry -- we're here to help! Once you're done going through the WDL basics, we'll point you to a series of tutorials that show how to apply each option to some classic use cases that we encounter in genomics. But first, we'll show you how to check that the syntax of a workflow is valid.

You can read about different plumbing options below, or continue to the next section, [Validate Syntax](validate_syntax.md).

## Plumbing options

WDL supports several different plumbing options and you can read more about them below.

### Linear Chaining

The simplest way to chain tasks together in a workflow is a **linear chain**, where we feed the output of one task to the input of the next, like so:

![Diagram depicting the flow of data in a workflow that uses linear chaining. The output from step A is used as the input for step B, and the output from step B is used as the input for step C.](../Images/linear-chaining.png)

This is easy to do because WDL allows us to refer to the output of any task (declared appropriately in the task's `output` block) within the `call` statement of another task (and indeed, anywhere else in the `workflow` block), using the syntax `task_name.output_variable`. So here, we simply specify in the call to `stepB` that we want it to use `stepA.out` as the value of the input variable `in`, and it's the same rule for `stepC`.

```
call stepB {
  input:
    in = stepA.out
}
call stepC {
  input:
    in = stepB.out
}
```

This relies on a principle called *hierarchical naming* that allows us to identify components by their parentage.

### Multi-input / Multi-output

The ability to connect outputs to inputs described in [Linear Chaining](Linear_chaining.md), which relies on *hierarchical naming*, allows you to chain together tools that produce multiple outputs and accept multiple inputs, and specify exactly which output feeds into which input.

![Diagram depicting the flow of data in a workflow with tasks producing multiple outputs that are used as inputs to a downstream task. The output from step A is used as the input for step B, and the outputs from step B, out1 and out2, are used as the inputs for step C, in1 and in2.](../Images/multi-input-multi-output.png)

Since the outputs for `stepB` are named differently, we can specify which output maps to which input for `stepC`.

```
call stepC {
  input:
    in1 = stepB.out1,
    in2 = stepB.out2
}
```

### Branch and Merge

The ability to connect outputs to inputs described in [Linear Chaining](Linear_chaining.md) and [Multi-input/Multi-output](MultiInput_MultiOutput.md), which relies on *hierarchical naming*, can be further extended to direct a task's outputs to separate paths, do something with them, then merge the branching paths back together.

![Diagram depicting the flow of data in a workflow with branching paths that merge back together. The output from step A is used as the input for steps B and C, and the outputs from steps B and C are used as the inputs for step D.](../Images/branch-and-merge.png)

Here you can see that the output of `stepA` feeds into both `stepB` and `stepC` to produce different outputs, which we then feed together into `stepD`.

```
call stepB {
  input:
    in = stepA.out
}
call stepC {
  input:
    in = stepA.out
}
call stepD {
  input:
    in1 = stepC.out,
    in2 = stepB.out
}
```

### Conditionals (if/else)

Sometimes when pipelining, there are steps you won't want to run all the time. This could mean switching between two paths (run a tool in modeA vs. run a tool in modeB) or skipping a step entirely (run a tool vs. not running a tool). In cases such as these, we will use a **conditional** statement.

![Diagram depicting the flow of data in a workflow with conditional statements. The output from step A is used to determine whether step B will be executed. If the conditional statement is true, the output from step A is used as the input for step B, and the output from step B is used as the input for step C. If the conditional statement is false, the output from step A is used as the input for step C, and step B is skipped.](../Images/conditionals.png)

To use a conditional statement in WDL, you write a standard `if` statement:

```
if (shouldICallStepB) {
  call stepB {
    input:
      in = stepA.out
  }
}
```

The `if` statement can be controlled explicitly, as we have done in the above example by using a Boolean variable. It can also be controlled implicitly by testing the value of some other variable with its own purpose besides being a switch mechanism.

```
if (myVar>0) {
  call stepB
}
```

WDL doesn't support `else` statements in this context, but we can get around that by writing paired `if` statements using the `!` modifier to get the opposite value of the original variable, like so:

```
if (!shouldICallStepB) {
  call stepC {
    input:
      in = stepA.out
  }
}
```

### Scatter-Gather Parallelism

Parallelism is a way to make a program finish faster by performing several operations in parallel, rather than sequentially (i.e. waiting for each operation to finish before starting the next one). For a more detailed introduction on parallelism, you can read about it in-depth in the GATK article, [Parallelism - Multithreading - Scatter Gather](https://gatk.broadinstitute.org/hc/en-us/articles/360035532012).

![Diagram depicting the flow of data in a workflow using scatter-gather parallelism. Several input files are fed into step A in parallel, and the output from each execution of step A is combined and used as the input for step B.](../Images/scatter-gather-parallelism.png)

To do this, we use the `scatter` function from the [WDL standard library](https://github.com/openwdl/wdl/blob/main/versions/1.0/SPEC.md#standard-library), which will produce parallelizable jobs running the same task on each input in an array, and output the results as an array as well.

```
input {
  Array[File] inputFiles
}
scatter (oneFile in inputFiles) {
  call stepA {
    input:
      in = oneFile
}
call stepB {
  input:
    files = stepA.out
}
```

The magic here is that the array of outputs is produced and passed along to the next task without you ever making an explicit declaration about it being an array. Even though the output of `stepA` looks like a single file based on its declaration, just referring to `stepA.out` in any other `call` statement is sufficient for WDL to know that you mean the array grouping the outputs of all the parallelized `stepA` jobs.

In other words, the **scatter** part of the process is *explicit* while the **gather** part is *implicit*.

### Task Aliasing

When you need to call a task more than once in a workflow, you can use **task aliasing**. It would be tedious to copy-paste a task's definition and change the name each time you needed to use it again in the workflow. This method, termed [copy and paste programming](https://en.wikipedia.org/wiki/Copy-and-paste_programming), is simple enough up front but difficult to maintain in the long run. Imagine you found a typo in one of your tasks--you'd need to fix that typo in every single pasted task! However, using WDL's built-in task aliasing feature, you can call the same task code and assign it an alias. Then, following the principle of hierarchical naming, in order to access the output of an aliased task we use the alias, rather than the original task name.

![Diagram depicting the flow of data in a workflow using task aliasing. Step A is first called using the task alias, firstInput, and the output of this task is used as the input for step B. Step A is called a second time using the task alias, secondInput, and the output of this task is used as the input for step C.](../Images/task-aliasing.png)

To use an alias, we use the syntax `call taskName as aliasName`.

```
call stepA as firstInput {
  input:
    in = in1
}
call stepA as secondInput {
  input:
    in = in2
}
call stepB {
  input:
    in = firstInput.out
}
call stepC {
  input:
    in = secondInput.out
}
```

Continue to the next section, [Validate Syntax](./validate_syntax.md).


# Linear chaining
The simplest way to chain tasks together in a workflow is a **linear chain**, where we feed the output of one task to the input of the next, as shown in the diagram below.

![diagram of linear chaining. Input goes through stepA and generates an output which is then used as input in stepB, which generates output that is used as input in StepC](../Images/linear_chaining.png)

This is easy to do because WDL allows us to refer to the output of any task (declared appropriately in the task's `output` block) within the `call` statement of another task (and indeed, anywhere else in the `workflow` block), using the syntax `task_name.output_variable`. 

Using the diagram as an example, we can specify in the call to `stepB` that we want it to use `stepA.out` as the value of the input variable, which we'll define as `in`.  It's the same rule for `stepC`.

To write this in a WDL script, we'd use the following code:

```wdl
  call stepB { 
    input: 
      in = stepA.out
    }
  call stepC {
    input: 
      in = stepB.out
    }
```

This relies on a principle called "hierarchical naming" that allows us to identify components by their parentage.

## Generic example script

To put this in context, here is what the code for the workflow illustrated above would look like in full:

```wdl
version 1.0
workflow LinearChain {
  input {
    File firstInput
  }
  call stepA { 
    input: 
      in = firstInput
  }
  call stepB {
   input:
     in = stepA.out
  }
  call stepC { 
    input: 
      in = stepB.out
  }
}

task stepA {
  input {
    File in
  }
  command <<<
    programA I=~{in} O=outputA.ext
  >>>
  output { 
    File out = "outputA.ext"
  }
}

task stepB {
  input {
    File in
  }
  command <<<
    programB I=~{in} O=outputB.ext
  >>>
  output { 
    File out = "outputB.ext"
  }
}

task stepC {
  input {
    File in
  }
  command <<<
   programC I=~{in} O=outputC.ext
  >>>
  output { 
    File out = "outputC.ext" 
  }
}
```

## Concrete example

Let’s look at a concrete example of linear chaining in a workflow designed to pre-process some DNA sequencing data (MarkDuplicates), perform an analysis on the pre-processed data (HaplotypeCaller), then subset the results of the analysis (SelectVariants).

![an image depicting how a BAM file is used as input to the MarkDuplicate software tool, which produces a metrics file and a deduplicated BAM, which is then used as input to HaplotypeCaller. This produces a raw VCF which is used as an input along with a variable "type" to the SelectVariants tool, producing a subsetted VCF.](../Images/GATK_linear_chaining.png)

The workflow involves three tasks:

* MarkDuplicates takes in a `File bamFile` and produces a `File metrics` and a `File dedupBAM`.
* HaplotypeCaller takes in a `File bamFile` and produces a `File rawVCF`.
* SelectVariants takes in a `File VCF` and a `String type` to specify whether to select INDELs or SNPs. It produces a `File subsetVCF`, containing only variants of the specified type.

### Concrete example script

This is what the code for the workflow illustrated above would look like:
```wdl
version 1.0
workflow LinearChainExample {
  input {
    File originalBAM
  }
  call MarkDuplicates { 
    input: 
      bamFile = originalBAM 
  }
  call HaplotypeCaller { 
    input: 
      bamFile = MarkDuplicates.dedupBam 
  }
  call SelectVariants { 
    input: 
      VCF = HaplotypeCaller.rawVCF, 
      type="INDEL" 
  }
}

task MarkDuplicates {
  input {
    File bamFile
  }
  command <<<
    java -jar picard.jar MarkDuplicates \
    I=~{bamFile} O=dedupped.bam M= dedupped.metrics
  >>>
  output {
    File dedupBam = "dedupped.bam"
    File metrics = "dedupped.metrics"
  }
}

task HaplotypeCaller {
  input {
    File bamFile
  }
  command <<<
    java -jar GenomeAnalysisTK.jar \
      -T HaplotypeCaller \
      -R reference.fasta \
      -I ~{bamFile} \
      -o rawVariants.vcf
  >>>
  output {
    File rawVCF = "rawVariants.vcf"
  }
}

task SelectVariants {
  input {
    File VCF
    String type
  }
  command <<<
    java -jar GenomeAnalysisTK.jar \
      -T SelectVariants \
      -R reference.fasta \
      -V ~{VCF} \
      --variantType ~{type} \
      -o rawIndels.vcf
  >>>
  output {
    File subsetVCF = "rawIndels.vcf"
  }
}
```

*Note that here for simplicity we omitted the handling of index files, which has to be done explicitly in WDL.* 



```
## Warning in readLines(dest_file): incomplete final line found on 'resources/
## other_chapters/ScatterGather_parallelism.md'
```


# Scatter-gather parallelism
Parallelism is a way to make a program finish faster by performing several operations in parallel, rather than sequentially (i.e. waiting for each operation to finish before starting the next one). For a more detailed introduction on parallelism, you can read about it in-depth [here](https://gatk.broadinstitute.org/hc/en-us/articles/360035532012).

![Diagram depicting parallelism. Three separate inputs are individually used as input to the same workflow running in parallel. The workflow runs the input through a process StepA. Each independent workflow produces one output which is are then gathered and all used together as input to a new process StepB, which produces a single output.](../Images/ScatterGather.png)

To do this, we use the `scatter` function described in the [WDL 1.0 spec](https://github.com/openwdl/wdl/blob/main/versions/1.0/SPEC.md#scatter--gather), which will produce parallelizable jobs running the same task on each input in an array, and output the results as an array as well. 

The code below shows an example of the scatter function:

```wdl
version 1.0
workflow wf {
  Array[File] inputFiles
  scatter (oneFile in inputFiles) {
    call stepA {
     input: 
       in = oneFile 
    }
  }
  call stepB { 
    input: 
      files = stepA.out
  }
}
```

The magic here is that the array of outputs is produced and passed along to the next task without you ever making an explicit declaration about it being an array. 

Even though the output of stepA looks like a single file based on its declaration, just referring to `stepA.out` in any other `call` statement is sufficient for WDL to know that you mean the array grouping the outputs of all the parallelized stepA jobs.

In other words, the `scatter` part of the process is *explicit* while the `gather` part is *implicit*.

## Generic example script

To put this in context, here is what the code for the workflow illustrated above would look like in full:

```wdl
version 1.0
workflow ScatterGather {
  input {
    Array[File] inputFiles
  }

  scatter (oneFile in inputFiles) {
    call stepA { 
      input: 
        in = oneFile 
    }
  }
  call stepB { 
    input: 
      files = stepA.out 
  }
}

task stepA {
  input {
    File in
  }
  command <<<
   programA I = !{in} O = outputA.ext 
  >>>
  output { 
    File out = "outputA.ext" 
  }
}

task stepB {
  input {
    Array[File] files
  }
  command <<<
   programB I = ~{files} O = outputB.ext 
  >>>
  output { 
    File out = "outputB.ext" 
  }
}
```
## Concrete example

Let’s look at a concrete example of scatter-gather parallelism in a workflow designed to call variants individually per-sample (HaplotypeCaller), then to perform joint genotyping on all the per-sample GVCFs together (GenotypeGVCFs).

![Diagram depicting how three individual sample BAM files are used as input to the HaplotypeCallerERC tools in parallel. Each individual run of the tool produces a GVCF. The GVCFs are gathered and the array is used as an input to the GenotypeGVCFs tool, which produces a raw VCF file as final output.](../Images/scatter_concrete.png)

The workflow involves two tasks:

* `HaplotypeCallerERC` takes in a `File bamFile` and produces a `File GVCF`.
* `GenotypeGVCFs` takes in an `Array[File] GVCF` and produces a `File rawVCF`.


### Concrete example script

This is what the code for the workflow illustrated above would look like:

```wdl
version 1.0
workflow ScatterGatherExample {
  input {
    Array[File] sampleBAMs
  }

  scatter (sample in sampleBAMs) {
    call HaplotypeCallerERC { 
      input: 
        bamFile=sample 
    }
  }
  call GenotypeGVCF {
   input: 
     GVCFs = HaplotypeCallerERC.GVCF 
  }
}

task HaplotypeCallerERC {
  input {
    File bamFile
  }
  command <<<
    java -jar GenomeAnalysisTK.jar \
        -T HaplotypeCaller \
        -ERC GVCF \
        -R reference.fasta \
        -I ~{bamFile} \
        -o rawLikelihoods.gvcf
  >>>
  output {
    File GVCF = "rawLikelihoods.gvcf"
  }
}

task GenotypeGVCF {
  input {
    Array[File] GVCFs
  }
  command <<<
    java -jar GenomeAnalysisTK.jar \
        -T GenotypeGVCFs \
        -R reference.fasta \
        -V ~{GVCFs} \
        -o rawVariants.vcf
  >>>
  output {
    File rawVCF = "rawVariants.vcf"
  }
}
```

For simplicity, we omitted the handling of index files in the code above, which has to be done explicitly in WDL. 

Additionally, please note that we did not expressly handle delimiters for Array data types in this example. 

To learn explicitly about how to specify the `~{GVCFs}`input, please see [this tutorial](./jointcall_with_scatter.md).

#### Scatter limitations

Note that the max allowable scatter width per scatter in Terra is 35,000.

### Managing Software Environments


## Tools for Editing WDLs
- VSCode has an extension
- ways to plot a DAG? 


