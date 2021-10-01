---
layout: post
title: "Synchronizing local and remote directories"
categories: shell
---

Ever since deep learning models are on the rise, the compute capability required to train such models is exponentially increasing. Most ML developers often use remote systems to build, test and deploy these huge models. Building and debugging such models remotely can be time-consuming. There are alternatives such as [remote development using VSCode](https://code.visualstudio.com/docs/remote/ssh), or you can make local changes and deploy them erratically to test and run your models. When I was using PyCharm, it had this incredible feature where any changes in the local system would reflect on the remote server synchronously. It also had an option to set the default interpreter to a remote interpreter. So every time you run the model, it will run it in the remote system. This was comfortable for me. I loved this idea, I didn't have to run custom models on my system anymore, except for one problem. PyCharm is not a lightweight IDE. I switched to VSCode, and Sublime but none of them had this feature.

In this article, I talk about the a bash function that enables you to automatically deploy changes in your local directory to a remote server. And this doesn't need any fancy IDE's.


## Prerequisites

-    Bash shell
-    Remote server must have `rsync`
-    `inotify-tools` in local system.

#### 1. Set up your `~/.ssh/config` file

```ssh
Host aws_gpu
  User random
  IdentityFile /path/to/your/file/ec2_instance.pem
  LocalForward 8888 localhost:8888
  HostName ec1-23-456-789.eu-west-2.compute.amazonaws.com
```

#### 2. Create  `remote_sync.sh` file

> You can give any name to the file.

This file will have our bash function. 

```bash
# Make sure remote desktop has rsync 
syncRemote() {
    clear_vars() {
            unset SYNC_DESTINATION
            unset TEMP_VAR
            unset SYNC_SOURCE
    }
    usage(){
        echo 'usage: syncRemote --src /your/path --dst remote:/your/dest/path --ignore "*.pt"  --max-size 200m'
    }

    clear_vars
    sync_MAX_SIZE=200m

# ` symbol- command substitution. The `command` construct makes available the output of command for assignment to a variable. 
# This is also known as backquotes or backticks.
    TEMP_VAR=`getopt -o d:s:, --long dst:,src:,max-size:,ignore: -- "$@"`
    eval set -- "$TEMP_VAR"

    while true; do
        case "$1" in
            --src)
                case "$2" in
                    "--"* ) 
                        shift 2;;
                    "") 
                        shift 2;;
                    *)
                        SYNC_SOURCE=$2                    
                        shift 2;;
                esac ;;

            --dst)
                case "$2" in
                    "--"* ) 
                        echo "Destination cannot be empty ";
                        echo "Source 2:  ""$2"; 
                        clear_vars
                        return 1;; 
                    "")
                        echo "Destination cannot be empty ";
                        echo "Source 2:  ""$2"; 
                        clear_vars
                        return 1;; 
                    *) 
                        SYNC_DESTINATION=$2
                        shift 2;;
                esac ;;

            --max-size)
                case "$2" in
                    "--"* ) 
                        echo " Using default max-size: ""$sync_MAX_SIZE"
                        echo $TEMP_VAR;
                        shift 1;;
                    "") 
                        echo " Using default max-size: ""$sync_MAX_SIZE"
                        echo $TEMP_VAR;
                        shift 1;;
                    *)
                        sync_MAX_SIZE=$2                    
                        shift 2;;
                esac ;;

            --) shift ; break ;;
            *) 
                echo "Unknown Options: ""$@" ; 
                return 1;;
        esac
    done

    if [ -z "$SYNC_DESTINATION" ]
    then
        echo "Destination cannot be empty!!"
        clear_vars
        return 1
    fi

    if [ -z "$SYNC_SOURCE" ]
    then
        echo "Source is Empty, using ""$(pwd)"" as source."
        $SYNC_SOURCE=$(pwd)
    fi

    echo 'Watches established in folder -->' "$SYNC_SOURCE"
    echo "Maximum size : ""$sync_MAX_SIZE"

    while inotifywait -r -e modify,create,delete,move $SYNC_SOURCE; do
        rsync -avz -- --max-size="$sync_MAX_SIZE" "$SYNC_SOURCE""/"  "$SYNC_DESTINATION" 
    done

} 
```


#### 3. Source our file

```bash
source /path/to/our/file/remote_sync.sh

# or
echo "source /path/to/remote_sync.sh" >> ~/.bashrc
## Be extra careful with >> is append, > is write. 
```

##### For using this function without sourcing, you can just add 

### Demo

#### Sample Usage
```sh
syncRemote --src /your/source/directory --dst aws_gpu:/your/destination/directory 
# replace aws_gpu and source and destination directory
```

### Limitations

- There's no proper way to kill the process, so I use `Ctrl+C` to kill the foreground process. It is advisable not to use `nohup` with this command because it could be painful to kill the process later.
- Old files in remote system persist even if original file is renamed/deleted in local directory.

### Helpful resources

- [Stackoverflow - How to keep two folders automatically synchronized?](https://stackoverflow.com/questions/12460279/how-to-keep-two-folders-automatically-synchronized)
- [Tutorial on `getopt`](https://www.bahmanm.com/2015/01/command-line-options-parse-with-getopt.html)