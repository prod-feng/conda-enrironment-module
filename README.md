# conda-environment-module


# UPDATE: this "$PATH_modshare counter" issue only happens to the environment-modules versions from 4.0 to 4.8. The older version of 3, and most recent version of 5, do not have this problem.



The "module" environment package is very popular in High Performance Computing filed, and the anacaonda is also becoming popular in HPC too. People usually use them both at the same time to mamage runtime environment.

Sometime, when I use anaconda in my environment, it seems it has conflicts with the Linux module enrironment pakcage for managing the enrironment virables.

For exapmple, using conda, I want to activate an env named "alpha"

>> module add basic  

(this command will triger module command to initialize "$PATH_modshare",see follwing for explanation)
>> conda activate alpha

Then, I use module commane to set my GCC9 environment:

>>module add gcc9

It will report the following warning messages:

```text
WARNING: $PATH does not agree with $PATH_modshare counter. The following directories' 
usage counters were adjusted to match. Note that this may mean that module unloading 
may not work correctly
...
```
This is because that both conda and module are used to manage the environment here, and the two have different ways to do so.

conda activate command will change the default $PATH of "/apps/anaconda3/bin" to the new conda env of "alpha" here to "/apps/anaconda3/envs/alphafold/bin".

While at the same time, module command will use it's own variable named "$PATH_modshare" to keep tracking it's original "$PATH".

So here what happens is: conda avtivate changed $PATH, but not "$PATH_modshare"; then when you try to use "module" command to load/rm a module environment, the error occurs.

Most of the time, this warning won't cause issue. While in some cases, it may cause you to use unexpected/wrong lib/excutables, since the "module" command will try to correct "$PATH" using it's "$PATH_modshare".
When you see this warning, you can double check the $PATH variable to verify if it is the one you expect, otherwise you may need to manually corret it.

To avoid this issue, you need to use "module" to load all of the runtime environment modules at first, and then, you can run "conda activate XXX".

Remember every time you run "module" load or rm, it will update the $PATH, and $PATH_modshare.

And before you run "module rm YYY", you need run "conda deactivate" command first to reverse back to original state of $PATH, if you activate a conda env before.


There is another quick way to fix this issue. It is to modify it's source file of "/apps/anaconda3/lib/python3.7/site-packages/conda/activate.py".

In this file, in function "_build_activate_stack", near the end of the function, add one line to let conda to manage the PATH_modshare every time it changes the $PATH:

```text
        #add the fllowing line here to manage PATH_modshare
        export_vars['PATH_modshare'] = new_path
        self._build_activate_shell_custom(export_vars)

        return {
            'unset_vars': unset_vars,
            'set_vars': set_vars,
...
        }
```

and in function "build_deactivate", also near the end of this function:

```text
        #add the following line here to manage PATH_modshare
        export_vars['PATH_modshare'] = new_path
        return {
            'unset_vars': unset_vars,
            'set_vars': set_vars,
            'export_vars': export_vars,
...
        }
```

And also the function of "build_reactivate', which get called after installation,upgradation, and removal of packages.

This fix looks like can fix the issue, at least temporarily.

The issue can be avoided partially via "--stack" option of "conda activate". This option will prepend the new path to the $PATH variable, other than modify the original anaconda bin path to be the new env. For example:

```text
>module load base

>conda activate --stack alpha 

>module load gcc9
...
>module rm gcc9
```

At the point, everythin is fine. 

After that, you deactivate the "alpha" env:
```text
>conda deactivate
...
>module load cmake

```

Now the same warning message pops out.

So the solution is: 1. Use module command to load all needed environment modules, then activate your conda env(e.g., "alpha"). After job is done, deactivate your env, then can module rm. Or exit the session;2. Modify the activate.py source file, update $PATH_modshare same as update $PATH each time, like the above.

