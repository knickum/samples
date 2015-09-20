# Simple Bash Logging

## Samples to automatically add in syslog logging to bash scripts

Preferred option in many cases would probably be option 4, except where making a temp file should be avoided for
whatever reason or in simple scripts where brevity would be nice

All information comes from: http://urbanautomaton.com/blog/2014/09/09/redirecting-bash-script-output-to-syslog/ with
some additional ideas from
http://stackoverflow.com/questions/11027679/store-capture-stdout-and-stderr-in-different-variables-bash

1. Logs stdout and stderr to syslog, echoes to stdout, but doesn't differentiate between stdin and stdout

    ```Bash
    #!/bin/bash
    
    # redirect stdin/stdout to syslog
    exec 1> >( logger -s -t $( basename $0 ) ) 2>&1
    
    echo "Writing to stdout"
    echo "Writing to stderr" >&2
    ```

2. Logs stdout and stderr to syslog, echoes to stdout and stderr, respectively, but doesn't retain ordering between
messages

    ```Bash
    #!/bin/bash
    
    # redirect stdin/stdout to syslog
    exec 1> >( logger -s -t $( basename $0 ) ) 2>&1
    # redirect stdin to syslog
    exec 2> >( logger -s -t $( basename $0 ) ) 2>&1
    
    echo "Writing to stdout"
    echo "Writing to stderr" >&2
    ```

3. Logs stdout and stderr to syslog, echoes to stdout and stderr, respectively, retains ordering, but may not handle
stdout/stderr in the same fashion from 'external' commands e.g. ls

    ```Bash
    #!/bin/bash
    
    readonly SCRIPT_NAME=$( basename $0 )
    
    log() {
        echo "$@"
        logger -p user.notice -t $SCRIPT_NAME "$@"
    }
    
    logerr(){
        echo "$@" >&2
        logger -p user.error -t $SCRIPT_NAME "$@"
    }
    
    log "Writing to stdout"
    logerr "Writing to stderr"
    
    out=$( ls $0 blah 2>&1 )
    echo "$out" 2> /dev/null
    echo "$out" > /dev/null
    ```

4. Logs stdout and stderr to syslog, echoes to stdout and stderr, respectively, retains ordering with temp file

    ```Bash
    #!/bin/bash
    
    readonly SCRIPT_NAME=$( basename $0 )
    
    log() {
        echo "$@"
        logger -p user.notice -t $SCRIPT_NAME "$@"
    }
    
    logerr(){
        echo "$@" >&2
        logger -p user.error -t $SCRIPT_NAME "$@"
    }
    
    log "Writing to stdout"
    logerr "Writing to stderr"
    
    echo "" 2>&1
    
    error_file=$( mktemp )
    
    # `ls $0 blah` should write to both stdout and stderr
    out=$( ls $0 blah 2> $error_file )
    err=$( < $error_file )
    
    rm $error_file
    
    log "Writing to stdout"
    log "${out}"
    
    logerr "Writing to stderr"
    logerr "${err}"
    ```

5. Logs stdout and stderr to syslog, echoes to stdout and stderr, respectively, attempts to retain ordering without
a temp file (doesn't seem to be working at last check, possible bash version problem?)

    ```Bash
    #!/bin/bash
    
    readonly SCRIPT_NAME=$( basename $0 )
    
    log() {
        echo "$@"
        logger -p user.notice -t $SCRIPT_NAME "$@"
    }
    
    logerr(){
        echo "$@" >&2
        logger -p user.error -t $SCRIPT_NAME "$@"
    }
    
    log "Writing to stdout"
    logerr "Writing to stderr"
    
    test_cmd() {
        echo "Writing to stdout"
        echo "Writing to stdout" >&2
    }
    
    unset t_std t_err t_ret
    eval "$( ( test_cmd; exit 2 ) 2> >(t_err=$(cat); typeset -p t_err) > >(t_std=$(cat); typeset -p t_std); t_ret=$?; typeset -p t_ret )"
    
    
    ```
