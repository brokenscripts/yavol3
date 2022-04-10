# yavol3  

Yet Another Volatility 3 Docker setup  

This is another Volatility 3 build centered around the official Python container running on top of Alpine Linux.  This work is a mashup of both [sk4la](https://github.com/sk4la/) and the official Volatility 3 Dockerfile under their github docker branch.  

The containers are built using [dumb-init](https://github.com/Yelp/dumb-init) that acts as the `init` system, and an unprivileged user (_Note: Check the basic usage section for information about dumping files out / docker user permission sections_). Refer to the Image Tag section for additional information about container contents.  

---

## Image Tags  

This build has 3 versions that I'm maintaining for now shown below all that rely on a pinned version of Python 3 and that I will keep centered around the latest official Python 3 release, assuming it doesn't break Volatility 3.  


### latest  
This is the latest and recommended image.  
* Latest Python 3 non-rc release  
* Yara Python - Master branch  
* Volatility 3 - **stable** branch  
* pdbparse
* pycryptodome  
* capstone (via pip)  

Size: ~89 MB  


### edge  
This runs the develop branch of Volatility 3 to ensure that it is the absolute latest version.  
* Latest Python 3 non-rc release  
* Yara Python - Master branch  
* Volatility 3 - **develop** branch  
* pdbparse  
* pycryptodome  
* capstone (via pip)  

Size: ~90 MB  


### slim  
Slimming down as much as possible, and I do not recommend this image, but it strips functionality and packages (Does not include yara, pdbparse, pycryptodome, or capstone).  The goal with this one is to shrink it as much as possible.  
* Latest Python 3 non-rc release  
* ~~Yara Python - Master branch~~  
    * The following plugins will not work without yara and potentially any other imported plugin that needs yara  
        * `volatility3.plugins.windows.callbacks`  
        * `volatility3.plugins.windows.svcscan`  
        * `volatility3.plugins.windows.vadyarascan`  
        * `volatility3.plugins.yarascan`  
* Volatility 3 - **stable** branch  
* ~~pdbparse~~  
* ~~pycryptodome~~  
    * The following plugins will not work without pycryptodome and potentially any other imported plugin that needs crypto  
        * `volatility3.plugins.windows.cachedump`  
        * `volatility3.plugins.windows.hashdump`  
        * `volatility3.plugins.windows.lsadump`  
* ~~capstone (via pip)~~  
  * Capstone disassembles code and performs malware analysis  
  * Capstone improves accuracy of Windows 8+ memory samples  

Size: ~63 MB  

---

## Symbols  
Volatility 3 will automagically download symbols, if connected to the internet, in order to process memory.  In order to keep these symbols there is a `VOLUME` mount specified in the docker image for use.  
```docker
VOLUME [ "/symbols" ]
```

This volume expects the `windows`, `linux`, and `mac` sub-folders to be created under it.  In all examples below, I will create a `symbols` directory in my home folder to support this bind mount.  It is **NOT** needed, but every run will require re-downloading the symbols from Microsoft.  

> ⚠️ Ensure the folder on the host is created as your host user, rather than letting docker create it, to prevent permission problems.  

```bash
mkdir -p "${HOME}"/symbols/{windows,linux,mac}
```

This can then be used in every instance below with the following example:  
```bash
-v "${HOME}/symbols":/symbols
```

---

## Basic usage  
Pull an image, in this case the `latest` tag:  
```bash
docker pull brokenscripts/yavol3:latest
```

To mount in the current working directory, display the default help command, and then destroy the container after completion, without mounting a symbol directory:  
```bash
docker run \
--rm -i \
-v "$PWD":/workdir \
brokenscripts/yavol3
```


### A real usage example with keeping symbols  
First, download the symbols, and display the information: 
```bash
docker run \
--rm -i \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
brokenscripts/yavol3 \
-f memory.mem windows.info.Info 2>/dev/null
```

Next, since the symbol now exists, STDERR does not have to be redirected, so run a normal command
```bash
docker run \
--rm -i \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
brokenscripts/yavol3 \
-f memory.mem windows.pslist.PsList
```

In both of these examples, it is mounting in the current directory `"$PWD"` that gets mapped to `/workdir` inside of the container.  If you want a more complex usage supporting mapping in something other than the `"$PWD"` by hand, go to the bash wrapper below.  

---

## Docker permissions (Inside of the container)  
The Dockerfile used sets up the `unprivileged` user (`UID`: `1000`) with the group of `ci` (`GID`: `101`).  This can cause problems on systems when writing out files via `-D` to dump files.  This can be mitigated by adding the following run as user docker syntax to get the current running users IDs and map them in to the unprivileged user:  

```docker
--user $(id -u):$(id -g)  
```

An updated working example:  

```bash
docker run \
--rm -i \
--user $(id -u):$(id -g) \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
brokenscripts/yavol3 \
-f memory.mem windows.info.Info
```

---

## Alternative Entrypoints  
By default the containers  use the following entrypoint and command:  
```docker
ENTRYPOINT [ "/usr/bin/dumb-init", "--", "volatility3" ]
CMD [ "--help" ]
```  

Whenever input is put on the command line it overwrites the `CMD` input, sending user input to the `ENTRYPOINT` or in this case volatility 3.    


### volshell  
`volshell` will require the use of interactive (`-i`) tty (`-t`) flags in docker in order to keep the terminal open.   
```bash
docker run \
--rm -it \
-v "$PWD":/workdir \
--entrypoint volshell \
brokenscripts/yavol3
```


### pdbconv or pdbconv.py  
`pdbconv` is used to deal with pdb files.  Ensure you map the directory that contains the PDB file to `/workdir`.  In the case below, I am assuming that `"$PWD"` houses the PDB file to work on.  

> Windows symbol tables can be manually constructed from an appropriate PDB file.  The primary tool for doing this is built into Volatility 3, called `pdbconv.py`

**Symlinks** are created for ease of use that allow the ability to specify `pdbconv` or `pdbconv.py`:  
```bash
docker run \
--rm -it \
--user $(id -u):$(id -g) \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
--entrypoint pdbconv \
brokenscripts/yavol3 \
-f ntkrnlmp.pdb  
```

```bash
docker run \
--rm -it \
--user $(id -u):$(id -g) \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
--entrypoint pdbconv.py \
brokenscripts/yavol3 \
-f ntkrnlmp.pdb  
```

Once this command is used, ensure you move the built ISF file to its appropriate symbol directory (such as `symbols/windows`).  


### Shell (/bin/ash or /bin/sh)  
`/bin/ash` or `/bin/sh` will require the use of interactive (`-i`) tty (`-t`) flags in docker in order to keep the terminal open.  

```bash
docker run \
--rm -it \
--user $(id -u):$(id -g) \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
--entrypoint /bin/ash \
brokenscripts/yavol3  
```

```bash
docker run \
--rm -it \
--user $(id -u):$(id -g) \
-v "$PWD":/workdir \
-v "${HOME}/symbols":/symbols \
--entrypoint /bin/sh \
brokenscripts/yavol3  
```

This can be useful if you want to keep one container alive for running everything, or just to troubleshoot overall.  

---

## **Bash** wrapper  
In order to make this more seamless, you can add this function / bash wrapper to your environment.  It is not POSIX compliant, but it works great on bash.  

This parses the entire command line and hunts for `-f`.  Once found it pulls out the location to the specified file and mounts in the directory that contains the file being pointed at to `/workdir`. If there is an issue with the file or it does not recognize what is being pointed to for some reason, it will mount `$PWD` in and pass all arguments as the user typed.  

Why do this?  
This allows you to use this docker container seamlessly since you can point at a memory image like `-f ../../../../memory.mem` and this function will trim that to pass only `memory.mem` into the docker `CMD` and bind mount in the actual path that contains this file.  

An an example, the example relative path above `../../../../memory.mem` might translate to `/home/brokenscripts/memory_dumps/memory.mem`.  This wrapper will make `/home/brokenscripts/memory_dumps` be bind mounted to `/workdir`. This is the same as this line in docker run:
```bash
-v /home/brokenscripts/memory_dumps:/workdir
```
The file argument that Volatility expects `-f` would not have the same pathing inside the container to recognize `-f ../../../../memory.mem`. To fix that, the bash wrapper trims off any leading pathing before the actual file itself and passes, to Volatilty, just the filename:
`-f ../../../../memory.mem` becomes `-f memory.mem` in `/workdir` inside the container.  

Optionally, and shown in this script, is the `--user` parameter to make it play nicer.   

In order to use this, add this into a file such as `.dockerfunc` in your home directory and then inside a file such as `.bashrc` make it source this file. You can also create additional aliases, as shown in the `.dockerfunc` snippet for ease of us such as: `vol3`, `volatility3`, etc.  

```bash
# .bashrc (Add this snippet near the bottom)

if [[ -f "${HOME}/.dockerfunc" ]]; then
	source "${HOME}/.dockerfunc"
fi
```

Next is the magic in a wrapper / function. A different function can be created for `volshell`, `pdbconv`, etc. I will provide a wrapper for all of the entrypoints in a separate file.   
```bash
# .dockerfunc

volatility3(){
  # Store original arguments for re-use
  local INPUTARGS="$@"
  # Create a modified argument variable for passing into docker
  local MODIFIEDARGS=()

  # Find the file argument to dynamically bind that directory in Vol3 as the /workdir
  # Then modify file argument to be based on being inside docker instead of the host
  for i in "$@"; do
    case $i in
      '-f') # Explicitly look for -f
        INPUTFILE="$2"
        MODIFIEDARGS+="$i "
        MODIFIEDARGS+=$(basename "${INPUTFILE} ")
        shift # past argument
        shift # past value
        ;;
      *)  # Any argument not specified above
        MODIFIEDARGS+="$1 "
        shift # past value
        ;;
    esac
  done

  # If the file is valid, then set up the bind mount and run with the modified file arg
  if [[ -e "${INPUTFILE}" ]]
  then
    MYPATH=$(realpath "${INPUTFILE}")
    MYDIR=$(dirname "${MYPATH}")
    MYFILE=$(basename "${INPUTFILE}")
    ARGS=${MODIFIEDARGS}
  else
    # Either the file specified wasn't found, or the bash test failed.  
    # Bind the current shell's ${PWD} to /workdir as the fallback
    # Run the docker container with the arguments specified as is
    echo "[!] Warning: File option (-f) not found, bind mounting PWD!"
    MYDIR=${PWD}
    ARGS=${INPUTARGS}
  fi

  # Do not include -t (TTY) when executing single use commands
  docker run \
  -i --rm \
  --user $(id -u):$(id -g) \
  -v "${MYDIR}":/workdir \
  -v "${HOME}/symbols":/symbols \
  brokenscripts/yavol3:latest ${ARGS}
}

alias vol3="volatility3"
```

---  

## FAQ  
* Why do some run examples have `-t` and others do not?  
  * When using the `-t` (TTY) docker flag it will append a CRLF. This can cause a ridiculous amount of screen scrolling, typically when it involves the very first time symbols are pulled for the memory image being worked. In some cases you will need to use `-t`, but if not leave it off, or **recommened** redirect `STDERR`: `2>/dev/null`. I recommend using `windows.info.Info` and `2>/dev/null` for the first instance (see Basic Usage above) to get the appropriate symbols and then the rest of the commands can be ran without redirection.  
  > Reference: https://github.com/moby/moby/issues/8513  

* Why am I using `"${PWD}"` or `"$PWD"` instead of `"$(pwd)"`?  
  * `${PWD}` refers to the value of the variable named `PWD`.  This is the absolute path including symbolic links.  
  * `$(pwd)` refers to the output of the command `pwd`, which is ran in a sub shell.  

* What is the deal with the `RUN apk add --virtual stage && apk del stage` in the Dockerfile?  
  * This enables the installation of packages for use in compilation, in this case, or other uses and then immediate removal.  When used in the same Dockerfile `RUN` command, the overall image size is not affected.  

* Why use `--rm` and destroy the container after an individual command?  
  * I want this to be treated more like a binary rather than a container. So spin up a container long enough to do what I need, then destroy the environment. If you want something that lasts long term just remove the `--rm` docker parameter and add a `--name` docker parameter to be more friendly.  

* Dumping files is not working correctly  
  * Refer to section above: Docker permissions (Inside of the container)  

## Errors  

### Automagic / Symbols error: 
```
Volatility 3 Framework 2.0.1
WARNING  volatility3.framework.plugins: Automagic exception occurred: FileNotFoundError: [Errno 2] No such file or directory: '/usr/local/lib/python3.10/site-packages/volatility3/symbols/windows/ntkrpamp.pdb'
```

This is due to having incorrect permissions on the symbols folder (the user/group created in the container is `1000:101`)

As mentioned previously, any files dumped via Volatility will be written with this user / group.  


### sys.meta_path is None, Python is likely shutting down  
```
Exception ignored in: <function FileLayer.__del__ at 0x7f8203d87880>
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/volatility3/framework/layers/physical.py", line 193, in __del__
  File "/usr/local/lib/python3.10/site-packages/volatility3/framework/layers/physical.py", line 190, in destroy
  File "/usr/local/lib/python3.10/site-packages/volatility3/framework/layers/physical.py", line 109, in _file
  File "/usr/local/lib/python3.10/site-packages/volatility3/framework/layers/resources.py", line 101, in open
  File "/usr/local/lib/python3.10/urllib/request.py", line 601, in build_opener
  File "/usr/local/lib/python3.10/urllib/request.py", line 1400, in __init__
ImportError: sys.meta_path is None, Python is likely shutting down
```

This appears to be due to Python, specifically the `urllib` library, shutting down too fast (docker) and it doesn't fully close/close properly.  `2>/dev/null` can be used to ignore this.  
**Note**: This only seems to appear when downloading symbols for the very first time.  If the symbol is already downloaded this doesn't seem to happen.  A workaround can be running `windows.info.Info` with `2>/dev/null` to have the symbol downloaded and placed on the bind mount.  


### Screen spam
Refer to the `-t` above, but here is a more direct link:  
> Reference: https://github.com/moby/moby/issues/8513#issuecomment-216191236  
> TTY by default translates newlines to CRLF.  The default line-endings used by TTY is CRLF.
> ```
> $ docker run -t --rm debian sh -c "echo -n '\n'" | od -c
> 0000000   \r  \n
> 0000002
> ```
> disabling "translate newline to carriage return-newline" with stty -onlcr correctly gives:
> ```
> $ docker run -t --rm debian sh -c "stty -onlcr && echo -n '\n'" | od -c
> 0000000   \n
> 0000001
> ```

I can't tell if this is the cause or if it is something in Volatility, but this is what I'm talking about:  

```bash
# Hundreds of lines before this
...
...
Progress:   99.86               Reading Symbol layer
Progress:   99.87               Reading Symbol layer
Progress:   99.87               Reading Symbol layer
Progress:   99.88               Reading Symbol layer
Progress:   99.89               Reading Symbol layer
Progress:   99.90               Reading Symbol layer
Progress:   99.93               Reading Symbol layer
Progress:   99.93               Reading Symbol layer
Progress:   99.94               Reading Symbol layer
Progress:   99.95               Reading Symbol layer
Progress:   99.95               Reading Symbol layer
Progress:   99.96               Reading Symbol layer
Progress:   99.97               Reading Symbol layer
Progress:   99.98               Reading Symbol layer
Progress:   99.98               Reading Symbol layer
Progress:   99.99               Reading Symbol layer
Progress:  100.00               PDB scanning finished
```

`2>/dev/null` can prevent this from showing. This only seems to happen if no symbol exists and it has to download it on the first time.  