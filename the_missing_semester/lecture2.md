1. !! 表示上一次输入的命令  
    Example:  
    >$ cat mcd.sh  

    output:
    ```
    foo() {
        mkdir -p "$1"
        cd "$1"
    }
    ```
    >$ !!  

    output:
    ```
    cat mcd.sh  (上一次输入的命令)
    foo() {
        mkdir -p "$1"
        cd "$1"
    }
    ``` 
2. $? 表示上一次命令执行的结果，0表示正常，非0表示异常
    Example:  
    >$ echo $?

    output:
    ```
    0
    ```
3. $(some command) 可在字符串，或者赋值给变量
    Example:
