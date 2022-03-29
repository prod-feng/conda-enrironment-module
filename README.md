# conda-enrironment-module

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

While at the same time, module command will use it's own varaible named "$PATH_modshare" to track it's original "$PATH".

So here what happens is: conda avtivate changed $PATH, but not "$PATH_modshare"; then when you try to use "module" command to load/rm a module environment, the error occurs.

Most of the time, this warning won't cause issue. While in some cases, it may cause you to use unexpected/wrong lib/excutables, since the "module" command will try to correct "$PATH" using it's "$PATH_modshare".
When you see this warning, you can double the $PATH virable to verify if it is the one you expect, otherwise you may need to manually corret it.

To avoid this issue, you need to use "module" to load all of the runtime environment modules, at the end, you can then run "conda activate XXX".

And before you run "module rm YYY", you need run "conda deactivate" command first to reverse back to original state, if you activate and conda env before.


There is another quick way to fix this issue. It is to modify it's source file of "/apps/anaconda3/lib/python3.7/site-packages/conda/activate.py".

In this file, in function "_build_activate_stack" near the end of the function, add one line to let conda to manage the PATH_modshare too:

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

This fix looks like can fix the issue, at least temporarily.



