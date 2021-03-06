### Introduction
#### Concept and Rationale
This is a beginner's tutorial to Snakemake.

Rather than get bogged down in bioinformatics workflows, this Snakemake tutorial builds on a theme of making sandwiches for children. Variations are provided that attempt to address common workflow approaches.

Like Make, the actors in Snakemake are *usually* files. In this example:
* The children are files
* The ingredients are files
* The sandwiches are files

#### Installation
Follow the installation instructions here (bioconda is strongly recommended):
https://bitbucket.org/snakemake/snakemake/wiki/Home

If you plan on using R within Snakemake take a look at [this](SETUP.md).

Clone this lesson:
```
git clone https://github.com/leipzig/SandwichesWithSnakemake
````
### Tutorial
#### Explicit targets (see [basic.snake](basic.snake))
This is my snakefile:
```
(snake-env)$ cat basic.snake
SANDWICHES = ['Jack.pbandj','Jill.pbandj']

rule all:
     input: SANDWICHES

rule clean:
     shell: "rm -f *.pbandj"

rule peanut_butter_and_jelly_sandwich_recipe:
     input: "kids/{name}", jelly="ingredients/jelly", pb="ingredients/peanut_butter"
     output: "{name}.pbandj"
     shell: "cat {input.pb} {input.jelly} > {output}"
```

`all` is a pseudo-rule - it only exists to establish the variable `SANDWICHES` as a target.
Since it is the first rule it will be run by default.

```
(snake-env)$ snakemake -s basic.snake
Provided cores: 1
Job counts:
    count	jobs
    2		peanut_butter_and_jelly_sandwich_recipe
    1		all
    3
rule peanut_butter_and_jelly_sandwich_recipe:
     input: kids/Jill, ingredients/peanut_butter, ingredients/jelly
     output: Jill.pbandj
1 of 3 steps (33%) done
rule peanut_butter_and_jelly_sandwich_recipe:
     input: kids/Jack, ingredients/peanut_butter, ingredients/jelly
     output: Jack.pbandj
2 of 3 steps (67%) done
rule all:
     input: Jack.pbandj, Jill.pbandj
3 of 3 steps (100%) done
```
We could have typed the targets directly:
```
(snake-env)$ snakemake -s basic.snake Jack.pbandj Jill.pbandj
Provided cores: 1
Job counts:
    count	jobs
    2		peanut_butter_and_jelly_sandwich_recipe
    2
rule peanut_butter_and_jelly_sandwich_recipe:
     input: kids/Jill, ingredients/jelly, ingredients/peanut_butter
     output: Jill.pbandj
1 of 2 steps (50%) done
rule peanut_butter_and_jelly_sandwich_recipe:
     input: kids/Jack, ingredients/jelly, ingredients/peanut_butter
     output: Jack.pbandj
2 of 2 steps (100%) done
```
But you cannot type variables as targets in Snakemake.
```
(snake-env)$ snakemake -s basic.snake snakemake -s SANDWICHES
MissingRuleException:
No rule to produce SANDWICHES.
```
**You can only type in files and explicit (i.e. non-wildcard) rules as targets.**

Why only explicit rules?

`peanut_butter_and_jelly_sandwich_recipe` is an implicit rule - it uses wildcards to define its inputs and outputs. Can we run it?
```
(snake-env)$ snakemake -s basic.snake peanut_butter_and_jelly_sandwich_recipe
RuleException in line 6 of Snakefile:
Could not resolve wildcards in rule peanut_butter_and_jelly_sandwich_recipe:
name
```
Why didn't this work? Because Snakemake has no idea what you are trying to make.

A wildcard rule is a recipe, not a target. You might say, "hey why didn't it just look in the `kids` folder and then make a sandwich for each of them?" (This approach is implemented in the `glob.snake` example below.)



The `kids` directory in this example was named just for neatness. Our rule could just as easily been:
```
rule peanut_butter_and_jelly_sandwich_recipe:
     input: "{name}", jelly="ingredients/jelly", pb="ingredients/peanut_butter"
     output: "{name}.pbandj"
     shell: "cat {input.pb} {input.jelly} > {output}"
```

So now any existing file in the filesystem is eligible for a sandwich! Exactly whose sandwich is it supposed to make if I run `peanut_butter_and_jelly_sandwich_recipe`?


In Snakemake, just as in Make:
*  Write re-usable wildcard rules based on transforming one type of file to another
*  Set variables to specify actual file targets
*  Write a pseudo-rule that uses the variable as input.

In Snakemake (but not Make) you have to write a pseudo-rule that uses the variable as input.

#### You will screw this up, so please review!
Trying to call an implicit rule is the most common beginner's error in Snakemake, but the error it returns is not so helpful.

Calling snakemake with the following targets:
* any old file *OK*
* a variable *NOT OK*
* a rule that has named files as input *OK*
* a rule that has as input variables that are lists of named files *OK*
* a rule that has wildcard placeholders in the output *NOT OK*

### How can we derive targets from existing source files?  (see [glob.snake](glob.snake))

Snakemake is Python, so we can simply use Python's `glob` function to read a directory contents and then transform the names into targets
```
KIDS = glob.glob('kids/*')
SANDWICHES = [os.path.basename(kid)+'.pbandj' for kid in KIDS]

...
```
You can try this with:
```
(snake-env)$ snakemake -s glob.snake
```
#### How can we access the targets or sources from a list? (see [listfile.snake](listfile.snake))
```
KIDS = [line.strip() for line in open("A.Kid.List.txt").readlines()]
SANDWICHES = [kid+'.pbandj' for kid in KIDS]

...
```
You can try this with:
```
(snake-env)$ snakemake -s listfile.snake
```
#### How do I tell Snakemake which list of kids to process as a command line argument?
There are (at least) two ways we can accomplish this:
##### Use a configuration parameter (see [config.snake](config.snake))
Snakemake autosets a global variable `config`, even if no configfile is loaded. This can be used to pass arguments to the snakefile.
```
#Usage: snakemake -s config.snake --config list=B
KIDS = [line.strip() for line in open(config.get("list")+".Kid.List.txt").readlines()]
SANDWICHES = [kid+'.pbandj' for kid in KIDS]

...
```

You can try this with:
```
(snake-env)$ snakemake -s config.snake --config list=A
```

##### Use a sentinel (see [sentinel.snake](sentinel.snake))
The sentinel strategy, popular in Make, creates an output file as a fake target or "sentinel" from our list file and for input we employ a vanilla Python function `get_sandwiches` that returns a list of sandwiches.

The wildcards argument sent to `get_sandwiches` is evaluated with all possible wildcards from all rules before the workflow is started, so it should be written in a manner that minimizes ambiguity. Writing `get_sandwiches` is easier if we maintain consistent naming conventions, i.e. we suffix kid lists with .List.txt. Our sentinel, and the target we ask Snakemake to produce, will be `A.Kid.List` in order to process a file called `A.Kid.List.txt`.
```
#Usage: snakemake -s sentinel.snake A.Kid.List
import os
import fnmatch

def get_sandwiches(wildcards):
    #these get evaluated with wildcards from every rule
    #so be careful to open only the list file
    for wildcard in wildcards:
        if os.path.exists(wildcard+'.txt') and fnmatch.fnmatch(wildcard+'.txt', '*.List.txt'):
            kids = [line.strip() for line in open(wildcard+".txt").readlines()]
            sandwiches = [kid+'.pbandj' for kid in kids]
            return(sandwiches)
    return("")

rule listfile:
     input: get_sandwiches
     output: "{listfile}"
     shell: "touch {output}"

...
```
You can try this with:
```
(snake-env)$ snakemake -s sentinel.snake A.Kid.List
```

