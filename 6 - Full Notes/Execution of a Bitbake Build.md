[[BITBAKE]]

Yeah, so how do we start getting shit done with this? Let's see - 

The corporate ass answer is as follows - 
The primary purpose for running BitBake is to produce some kind of output such as a single installable package, a kernel, a software development kit, or even a full, board-specific bootable Linux image, complete with bootloader, kernel, and root filesystem. Of course, you can execute the bitbake command with options that cause it to execute single tasks, compile single recipe files, capture or clear data, or simply return information about the execution environment.


What does this mean? Use the command `bitbake` to do whatever the fuck do you want when it comes to making a Linux image. Get a distro, or a simple file, whatever. It all depends on how have you set the bastard up for execution. 

Here is a flow of what it looks for and does when it comes to executing the command - 

## 1.Parsing the base configuration Metadata

Yeah, no shit Sherlock, of course it will look through the metadata once it starts doing ANYTHING. In all seriousness, the base config metadata has the project's `bblayers.conf` file to determine what layers does bitbake need to look through, all the necessary `layers.conf` files - one from each layer that you have defined and the `bitbake.conf` file. The data itself can be divided into 3 major types - 
- Recipes - Details about pieces
- Class Data - Abstraction of a common build
- Configuration Data - Machine-specific stuff

`Layers.conf` files are used to create key variables such as `BBPATH` and `BBFILES`  `BBPATH` is used to search for configuration and class files under the conf and classes directories, respectively. `BBFILES` is used to locate both recipe and recipe append files (`.bb` and `.bbappend`). If there is no `bblayers.conf `file, it is assumed the user has set the `BBPATH` and `BBFILES` directly in the environment.

Then the `bitbake.conf` file is located with the `BBPATH` variable that was defined earlier. This may also hold other configuration files that are defined elsewhere, include or require directives.

Before all that - BitBake looks for certain variables like - 
- BB_ENV_PASSTHROUGH
- BB_ENV_PASSTHROUGH_ADDITIONS
- BB_PRESERVE_ENV
- BB_ORIGENV
- BITBAKE_UI


The first 4 are related to the shell behaviour of BitBake. It cleans all `env` variables in the shell before running, but through the first 4, we can set some variables that we need untouched. 

The base configuration metadata is global and therefore affects all recipes and tasks that are executed.

BitBake first looks in the PWD for a `conf/bitbake.conf` file, that has a `BBVARIABLE` which holds a space-delimited list of `layer` directories. 

For each layer, there is a `conf/layer.conf` file parsed with the LAYERDIR variable being set to the directory where the layer was found. These files are supposed to set up the BBPATH and other variables correctly for any build dir. 

Then there should be the `conf/bitbake.conf` under the `BBPATH`, which has directives to pull data from a given path. Only variable definitions and incluse directives are allowed in a `bitbake.conf` file. 

After the parsing is done, BitBake has an inheritance mechanism through the class files, which is used to inherit some standard classes. A class is parsed through only when an inheritance directive is encountered. 

The `base.bbclass` is always included. Other classes specified using the `INHERIT` var are also included. There is a `classes` sub directory under the `BBPATH` that BitBake looks for. 

To find out the configuration files and the class files - `$ bitbake -e > mybb.log`

## 2. Locating and Parsing the Recipes

During the config, bitbake has a set of `BBFILES`. Now, it is used to start parsing the entire thing, along with the append files that are to be applies. Each recipe and append file is parsed and stored into a datastore. A common way of doing this is to set variables that hold some of the metadata. For example - a recipe called `something_1.2.3.bb` would set `PN` to `something` and `PV` to `1.2.3`.

By the time parsing is complete for a recipe, BitBake has a list of tasks that the recipe defines and a set of data consisting of keys and values as well as dependency information about the tasks.

However, it only needs a small subset of the information to make decisions about the recipe. Consequently, BitBake caches the values in which it is interested and does not store the rest of the information. Experience has shown it is faster to re-parse the metadata than to try and write it out to the disk and then reload it. Where possible, subsequent BitBake commands reuse this cache of recipe information.

## 3. Providers

Assuming everything has been given and parsed by BitBake, now it starts trying to figure out a way to start building the target that it has been told to.  

This is started by looking through the `PROVIDES` list for each of the recipes. This is nothing but a list of names by which the recipe can be known. It is created implicitly via the `PN` variable and explicitly via the `PROVIDES` variable. 

## 4. Preference

Should there be multiple providers for a target, we use the `PREFERENCE` variables to define the version of the provider that we wish to use. By default, the preference values for files are set to 0. Setting `DEFAULT_PREFERENCE` to “-1” makes the recipe unlikely to be used unless it is explicitly referenced. Setting `DEFAULT_PREFERENCE` to “1” makes it likely the recipe is used. 

`PREFERRED_VERSION` overrides any `DEFAULT_PREFERENCE`setting. `DEFAULT_PREFERENCE` is often used to mark newer and more experimental recipe versions until they have undergone sufficient testing to be considered stable.

When there are multiple “versions” of a given recipe, BitBake defaults to selecting the most recent version, unless otherwise specified. If the recipe in question has a `DEFAULT_PREFERENCE` set lower than the other recipes (default is 0), then it will not be selected. 

This allows the person or persons maintaining the repository of recipe files to specify their preference for the default selected version. Additionally, the user can specify their preferred version.

## 5. Dependencies

Each task, such as fetch, unpack, compile, configure, and patch are considered as independent with their own set of dependencies. 

## 6. Task List

Now that it has everything, BitBake can handle the execution. First, it starts forking off threads till the set number of threads, as defined by a variable. As a task completes, there is a `STAMP` variable that is written to, which keeps track of what has been done and what is left to be done. On the next run, bitbake shall look at the `/tmp/stamp` directory for the `build` dir within it, and if a stamp is found, the particular task shall not be re-run. 

## 7. Execution of Tasks

A task can be either a python or a shell task. 

For shell tasks - BitBake writes a shell script to `${T}/run.do_taskname.pid` and then executes the script. The generated shell script contains all the exported variables, and the shell functions with all variables expanded. Output from the shell script goes to the file `${T}/log.do_taskname.pid`.

Looking at the expanded shell functions in the run file and the output in the log files is a useful debugging technique.

For Python tasks, BitBake executes the task internally and logs information to the controlling terminal.

