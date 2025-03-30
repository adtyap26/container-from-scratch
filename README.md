# Container From Scratch

This repository contains a minimal container implementation written in Go, designed for educational purposes. The code demonstrates the fundamental building blocks of container technology similar to Docker, but in a simplified form.

> **Attribution**: This code is based on "Building a container from scratch in Go" by Liz Rice (Microscaling Systems), as presented in her YouTube talk. This implementation follows her educational approach to understanding container fundamentals.

## Overview

This project implements a basic container system using Linux namespaces and chroot to isolate processes. The program creates separate PID and UTS namespaces for the containerized process and uses a minimalist Alpine Linux rootfs for the container's filesystem.

## How It Works

The implementation consists of two main components:

1. **The parent process (`run` function)** - Sets up the container environment
2. **The child process (`child` function)** - Runs within the container context

### Detailed Explanation

#### Parent Process (Run Function)

```go
func run() {
    fmt.Printf("running %v as PID %d\n", os.Args[2:], os.Getpid())
    cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID,
    }
    must(cmd.Run())
}
```

In this function:

- The program first prints the command and current PID for debugging purposes
- It creates a new command that re-executes the current binary (`/proc/self/exe`) with "child" as the first argument, followed by the original command arguments
- It passes through standard input, output, and error streams
- Most importantly, it sets up Linux namespaces using `Cloneflags`:
  - `CLONE_NEWUTS`: Creates a new UTS (Unix Time-sharing System) namespace, which isolates hostname and domain name
  - `CLONE_NEWPID`: Creates a new PID namespace, allowing processes inside the container to have their own isolated process IDs, starting from PID 1

#### Child Process (Child Function)

```go
func child() {
    fmt.Printf("running %v as PID %d\n", os.Args[2:], os.Getpid())
    cmd := exec.Command(os.Args[2], os.Args[3:]...)
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    must(syscall.Chroot("/tmp/minirootfs/"))
    must(syscall.Chdir("/"))
    must(syscall.Mount("proc", "proc", "proc", 0, ""))
    must(cmd.Run())
}
```

In this function:

- The program again prints the command and current PID (which should now be different due to the new PID namespace)
- It prepares to execute the actual command specified by the user
- Before executing the command, it performs three critical containerization steps:
  1. `syscall.Chroot("/tmp/minirootfs/")`: Changes the root directory to a minimal Alpine Linux filesystem, isolating the file system view for the process
  2. `syscall.Chdir("/")`: Changes the current working directory to the new root
  3. `syscall.Mount("proc", "proc", "proc", 0, "")`: Mounts a new proc filesystem in the container, which is essential for many Linux tools and provides process information
- Finally, it executes the specified command within this containerized environment

#### Main Function

```go
func main() {
    switch os.Args[1] {
    case "run":
        run()
    case "child":
        child()
    default:
        panic("Add your arguments please")
    }
}
```

This function acts as a simple router:
- If the first argument is "run", it calls the parent process function
- If the first argument is "child", it calls the child process function (this is only called internally when the program re-executes itself)
- Otherwise, it shows an error message

#### Helper Function

```go
func must(err error) {
    if err != nil {
        panic(err)
    }
}
```

This is a simple error handling utility that panics if an error occurs, ensuring that the program fails immediately if any critical operation fails.

## How to Use

1. First, download and extract an Alpine Linux mini root filesystem:

```bash
# Create a directory for the container filesystem
mkdir -p /tmp/minirootfs

# Download Alpine minirootfs
# You can find the latest versions at:
# https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/
wget https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-minirootfs-3.18.4-x86_64.tar.gz

# Extract it to the target directory
tar -xzf alpine-minirootfs-3.18.4-x86_64.tar.gz -C /tmp/minirootfs
```

2. Build the container program:

```bash
go build -o container main.go
```

3. Run a command in the container (needs to be run as root):

```bash
sudo ./container run /bin/sh
```

This will launch a shell inside the container. You can also run other commands:

```bash
sudo ./container run /bin/ls -la
```

## Limitations

This is a minimal implementation for educational purposes and has several limitations compared to production container systems:

- No network namespace isolation
- No user namespace isolation
- No resource limitations (cgroups)
- No volume mounting
- No image management
- Requires root privileges to run
- Hardcoded filesystem path (/tmp/minirootfs)

## Technical Concepts Demonstrated

1. **Linux Namespaces**: The core isolation feature in Linux used by container technologies
   - PID Namespace: Isolates process IDs
   - UTS Namespace: Isolates hostname and domain name

2. **Chroot**: Changes the root directory for a process, limiting its view of the filesystem

3. **Filesystem Mounting**: Creates necessary virtual filesystems like /proc inside the container

4. **Process Execution**: Creates and manages child processes in isolated environments

## Security Considerations

This implementation is for educational purposes only and should not be used in production. It requires root privileges to run and doesn't implement proper security measures.

For a secure container implementation, consider:
- User namespace isolation
- Capability restrictions
- Seccomp filters
- SELinux/AppArmor profiles
- cgroup resource restrictions

## Resources for Learning More

- [Linux Namespaces Documentation](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Alpine Linux](https://alpinelinux.org/)
- [Understanding Docker Containers](https://docs.docker.com/get-started/overview/)
- [Building a container from scratch in Go - Liz Rice (YouTube)](https://www.youtube.com/watch?v=Utf-A4rODH8)
