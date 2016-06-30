# Atomic initial clone

To encourage sidecar functionality, git-sync now provides the option to make your initial clone atomic. Many use cases require the target volume to be fully populated with the complete Git repository before another container begins accessing the content. 

The process involves cloning and pulling initially to a temporary directory and then renaming to the desired target name. We can ensure that the target directory will only appear to the in the volume filesystem once the pull is complete.

## Usage

In the configuration of the git-sync container, set the optional GIT_SYNC_ATOMIC environment variable to "true" (default is "false"). The GIT_SYNC_SUBDIR environment variable must also be set to a non-empty string (otherwise an error will result). This subdirectory will be created within the shared volume's root directory, specified by GIT_SYNC_DEST. GIT_SYNC_DEST should match the mountPath for git-sync's mounted emptyDir volume.

The subdirectory is necessary because the volume root directory path cannot be replaced by a rename. Any other containers using the volume's content should expect it to live in GIT_SYNC_SUBDIR within their own mountPath. For example, a webserver that uses the shared volume configured below might mount the volume at "/shared" in its own filesystem, and would look for the repo content at "/shared/data".
```
{
   ...
   containers: [
   {
       name: "git-sync",
       ...
       env: [{
               name: "GIT_SYNC_REPO",
               value: "https://github.com/kubernetes/kubernetes",
           },
           {
              name: "GIT_SYNC_DEST",
              value: "/git",
           },
           {
               name: "GIT_SYNC_SUBDIR",
               value: "data",
           },
           {
               name: "GIT_SYNC_ATOMIC",
               value: "true",
           },
       ],
       volumeMounts: [
           {
               name: "git-data",
               mountPath: "/git",
           },
       ],
   }],
   ...
   volumes: [
       {
           name: "git-data",
           emptyDir: {},
       },
   ],
   ...
}
```
 
## Pseudocode
 ```
 while
    pullTarget = $DEST/$SUBDIR  
    if initial pull
        if atomic
            mkdir $DEST/data-tmp
            pullTarget = $DEST/data-tmp
        endif
        git clone into pullTarget without checkout (only .git)
    endif    
    git pull into pullTarget    
    if atomic && initial pull
        rename pullTarget $DEST/$SUBDIR
    endif    
 endwhile
 ```
