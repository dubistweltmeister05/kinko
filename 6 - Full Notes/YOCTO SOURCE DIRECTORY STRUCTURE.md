[[3 - Tags/YOCTO]]

The top level components of the source directory are as follows - 
-  1. `bitbake/`.  This matches the current copy of the bitbake stable repo. BitBake is a Metadata interpreter, that reads the config files that have been defined in the repository (working) and does tasks that have been defined by this.  
   When we source the env setup script, the scripts and the bitbake/bin driectories are setup in the shell's PATH variable.
   
- 2.` build/. `This is the directory that contains the user conf files and the output of the openembedded build system in it's standard configuration , where the source tree is combines with the system output.
  We may set the output and config files in separate directories, by changine the sourced env-setup script.
   
- 3. `documentation/.` Houses the source of the yocto documentation and the templates/tools to generate our own docs in HTML/PDF formats. 
  
- 4. `meta/.` This holds the most minimal underlying openembedded metaddata. 
  
- 5.` meta-poky/.` This adds enough metadata to setup the poky reference distribution.
  
- 6.` meta-yocto-bsp/. `Contains the yocto project reference hardware's BSP.
  
- 7. `meta-selftest/.` Adds recipes and append files used by openembedded selftests to verify if everything is working as expected or nah. Running the self-tests is controlled by the addition or removal of this layer from the bblayers.conf file. 
  
- 8. `meta-skeleton/.` This is the template recipe for BSP and kernel development. 
  
- 9. `scripts/. `This has the various integration scripts that are implemented for extending the functionality of the yocto project environment.
  
- 10 `oe-init-build-env. `This is the script that initializes the output build environment. Sets up the `build` filder under the `poky` directory, and sources the scripts and the  bitbake/bin variables to the shell's path. The default output of the oe-init-build-env script is from the conf-summary.txt and conf-notes.txt files, which are found in the meta-poky directory within the Source Directory. 