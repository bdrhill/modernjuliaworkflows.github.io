+++
title = "Deploying your code"
+++

# Deploying your code

In this post, you will learn how to run Julia code on machines that you do not control.
Deployment can mean emailing a script to a colleague, shipping a container to a cluster, or keeping a web service alive for years.

{% <note> %}
This page is based on the JuliaCon 2026 talk [Lessons in deploying Julia to production services](https://www.youtube.com/watch?v=myZAp6vSgQA) by [Avik Sengupta](https://github.com/aviks), reused with permission.
It is a first draft: deployment practices change quickly, so contributions and corrections are welcome.
{% </note> %}

<!-- toc -->

{% <tldr> %}
Match the method to the target. Ship a pinned environment before reaching for containers or custom system images, build those artifacts ahead of time, and never precompile on the machines that serve users.
{% </tldr> %}

## What Deployment Means

To deploy is to run code on a machine that you do not own.
That includes a colleague's laptop, a supercomputer, a container in the cloud, or a customer's desktop.

Every program makes implicit assumptions about its environment: the operating system, the CPU instruction set, the available libraries, the versions of its dependencies, and its configuration files.
Deployment is the work of making those assumptions explicit, or removing them.

There is no single best method.
A batch job on an HPC cluster, a long-running web service, a desktop GUI, and a library called from Python have different requirements.
The rest of this page walks through the options from simplest to most involved.

## Start with the Trivial Method

The simplest way to deploy Julia code is to send someone a `.jl` file.

This works when the recipient already has Julia, the right packages, and a compatible environment.
For a small team that shares a development setup, that may be all you need.
Do not add a container or a system image just because production sounds complicated.

The next step up is to send the environment along with the code.

## Ship an Environment, Not Just Code

A Julia project has two TOML files:

- `Project.toml` lists the direct dependencies and compatibility bounds;
- `Manifest.toml` records the exact version of every direct and transitive dependency, as well as the Julia version used to resolve them.

For a **library**, commit only `Project.toml`: each user should resolve the manifest for their own Julia version and dependency set.
For an **application**, commit both `Project.toml` and `Manifest.toml`: you want every deployment to use exactly the versions that you tested.

A typical application layout looks like this:

    MyApp/
    ├── Project.toml
    ├── Manifest.toml
    └── src/
        └── main.jl

On the target machine, instantiate the environment and run the application:

```bash
julia --project=. -e 'using Pkg; Pkg.instantiate()'
julia --project=. src/main.jl
```

`Pkg.instantiate()` installs the versions recorded in `Manifest.toml`, but it does not update them.
Avoid calling `Pkg.update()` or `Pkg.resolve()` during deployment, since that can silently change the environment.

The manifest is tied to the Julia version that created it: if you resolved with Julia 1.13 and run with Julia 1.12, packages may fail to load or behave differently.
Install the same Julia version on the target machine, and make that version part of your build or deployment configuration.

### Private Registries

{% <tldr> %}If more than two people or more than two projects share internal packages, put those packages in a private registry.{% </tldr> %}

A Julia registry is a Git repository of small TOML files that describe package versions and their dependencies.
The General registry is the public one, but you can host your own.
This is useful for internal packages that should not be public, and it makes development and deployment more reproducible.

[LocalRegistry.jl](https://github.com/GunnarFarneback/LocalRegistry.jl) provides helpers for creating and updating a registry.
You can also manage it manually with pull requests, because the format is deliberately simple.
A private registry is worth the setup even during development, not only when you deploy.

## HPC Clusters

On a cluster, the compute nodes are often diskless or share a slow filesystem, and there may be thousands of them.
Two rules matter most:

1. put the Julia depot on a shared filesystem and precompile once, on a login or build node;
2. make it an error to start a job without a precompiled cache.

```bash
export JULIA_DEPOT_PATH=/shared/julia/depot

# Run once on the login/build node, not on every compute node
julia --project=. -e 'using Pkg; Pkg.instantiate(); Pkg.precompile()'
```

Then run jobs with:

```bash
julia --project=. --compiled-modules=strict --pkgimages=existing run.jl
```

`--compiled-modules=strict` fails if no precompile file is found instead of compiling on the fly.
`--pkgimages=existing` uses existing native code caches without creating new ones.
Without these flags, a misplaced environment can make every node recompile the same packages, wasting hundreds of CPU hours.

If the nodes have different CPUs, set a common target when you precompile:

```bash
export JULIA_CPU_TARGET=generic
```

The default target is already conservative and usually works across a homogeneous cluster.
For a heterogeneous cluster, choose a target that every node supports, or build separate depots for different node types.

MPI configuration is a topic of its own.
Many clusters require a system MPI, and [MPI.jl](https://github.com/JuliaParallel/MPI.jl) documents how to match it.

## Containers

Containers are the default way to deploy a service to the cloud.
Start from the [official Julia Docker images](https://hub.docker.com/_/julia) instead of installing Julia by hand.
Then move dependency installation and precompilation into the image build, where they are cached and paid for once.

```dockerfile
FROM julia:1.13

WORKDIR /app

# Precompile for a portable target, with a bounded number of tasks.
ENV JULIA_CPU_TARGET=generic
ENV JULIA_NUM_PRECOMPILE_TASKS=4

# Install dependencies in a separate layer: this is only rebuilt when the
# project or manifest changes.
COPY Project.toml Manifest.toml ./
RUN julia --project=. -e 'using Pkg; Pkg.instantiate(); Pkg.precompile()'

# Application code changes more often, so copy it last.
COPY src ./src

# Tell Julia about the resources the container is allowed to use.
ENV JULIA_NUM_THREADS=4
CMD ["julia", "--project=.", "--heap-size-hint=4G", "src/main.jl"]
```

A few details are easy to miss:

- **CPU target.** The machine that builds the image may differ from the machines that run it. Set `JULIA_CPU_TARGET` to a target that all runtime machines support.
- **Threads.** Julia may see the host's CPU count rather than the container's CPU limit. Set `JULIA_NUM_THREADS` explicitly. This also keeps libraries such as OpenBLAS from oversubscribing.
- **Memory.** Julia may see the host's memory rather than the container limit. Pass `--heap-size-hint` so the garbage collector starts working before the container is killed.
- **Precompile tasks.** Cap `JULIA_NUM_PRECOMPILE_TASKS` so a build does not start one task per host CPU.
- **Runtime precompilation.** Do not instantiate or precompile in the container entrypoint. Every second spent there is paid on every start.

If image builds are slow because your CI cannot cache layers, [this Discourse post](https://discourse.julialang.org/t/creating-a-docker-base-image-for-faster-deployments/121165) describes a base image that contains a custom sysimage and copies only the application source at deployment time.

## Precompile the Workload

Precompiling a package is not the same as precompiling the functions your application actually calls.
[PrecompileTools.jl](https://github.com/JuliaLang/PrecompileTools.jl) lets a package run a small workload during precompilation, so the methods on the hot path are compiled before anyone uses them.

```julia
using PrecompileTools: @compile_workload

@compile_workload begin
    # Exercise the code paths that users hit first.
    model = build_model()
    result = solve(model)
end
```

Keep the workload small but representative.
It runs every time the package is precompiled, so it should not dominate build time.
Use [SnoopCompile.jl](https://github.com/timholy/SnoopCompile.jl) to check what was compiled and to find invalidations.

For a web service, include a typical request in the workload.
Then add a warm-up request when the service starts, so the first real user does not pay for compilation that the workload missed.

## System Images

A system image is a serialized snapshot of a Julia session.
Julia itself ships with one containing `Base` and the standard libraries.
A custom system image adds your packages and their compiled code, which removes most of the load and compile latency.

[PackageCompiler.jl](https://github.com/JuliaLang/PackageCompiler.jl) creates custom system images:

```julia
using PackageCompiler

create_sysimage(
    ["MyPackage"];
    sysimage_path="MySysimage.so",
    cpu_target="generic",
)
```

The file extension depends on the operating system: `.so` on Linux, `.dylib` on macOS, and `.dll` on Windows.
Use the image with the `--sysimage` flag:

```bash
julia --sysimage=MySysimage.so --project=. src/main.jl
```

System images can make startup almost instantaneous, but they come with constraints:

- the build machine needs a C compiler;
- the image must be loaded by the same Julia version that created it;
- the image contains Julia code, not native artifacts, so package artifacts and shared libraries must be shipped separately;
- there is no cross-compilation: build one image per operating system;
- builds are slow and the resulting files can be gigabytes in size;
- every package in the image must be relocatable.

{% <advanced> %}
A sysimage is a serialized snapshot, so packages cannot assume that their source files sit at the same paths on the target machine.
Most packages in the ecosystem are relocatable, but package authors should test their code inside a sysimage.
{% </advanced> %}

Creating a system image also needs much more memory than running the application.
As a rule of thumb, budget three to four times the runtime memory: 16 GB for a build whose service needs 4 GB, and more for large package stacks.

## Applications and Libraries

A system image still requires Julia on the target machine.
PackageCompiler.jl can also bundle the sysimage, Julia runtime, and artifacts into an application that runs on a machine without Julia installed.

The entry point is a function `julia_main()::Cint` that returns `0` on success:

```julia
module MyApp

function julia_main()::Cint
    println("Hello from MyApp")
    return 0
end

end
```

Then build the application:

```julia
using PackageCompiler
create_app("MyApp", "MyAppCompiled")
```

Command-line arguments are available in the global `ARGS` array.
[ArgParse.jl](https://github.com/carlobaldassi/ArgParse.jl) makes them easier to handle.

PackageCompiler.jl can also produce a C-callable shared library with `create_library`.
Functions must be marked with `Base.@ccallable` and use C-compatible types such as `Cint`, `Cstring`, and `Cvoid`.
As with applications, relocatability is the hard part: see the [PackageCompiler.jl documentation](https://julialang.github.io/PackageCompiler.jl/stable/apps.html) before shipping binaries to other people.

[AppBundler.jl](https://github.com/PeaceFounder/AppBundler.jl) goes one step further and wraps an application in a native installer for Linux, macOS, or Windows.
It is a good fit if you want a double-clickable desktop application rather than a directory of files.

If your users live in Python, [PythonCall.jl](https://github.com/JuliaPy/PythonCall.jl) lets Python call Julia functions.
Distributing the result through PyPI or conda is possible, but still requires care around bundling the Julia runtime and any system image.

### JuliaC.jl

[JuliaC.jl](https://github.com/JuliaLang/JuliaC.jl) is a newer companion to PackageCompiler.jl.
It provides a `juliac` CLI and a library API for compiling and bundling Julia binaries, with optional trimming of unreachable code for smaller outputs.
It requires Julia 1.12 or later and a C compiler.
Install it as a Julia app, then run `juliac`:

```bash
juliac \
  --output-exe myapp \
  --bundle build \
  --trim=safe \
  --experimental \
  MyApp
```

The resulting executable still depends on `libjulia`, but the bundle can include the needed runtime libraries and artifacts.
See the JuliaC.jl README for the current flags and limitations.

## Long-Running Services

A web service built with [HTTP.jl](https://github.com/JuliaWeb/HTTP.jl), [Genie.jl](https://github.com/GenieFramework/Genie.jl), or [Oxygen.jl](https://github.com/ndortega/Oxygen.jl) has two latency problems: loading packages and compiling the first request.

The first is handled by precompilation and system images.
The second is handled by a precompile workload and a warm-up request.
After starting the service, send it a request that exercises the route your users will hit, and only then add it to the load balancer:

```bash
curl --fail --retry 10 --retry-connrefused http://localhost:8080/warmup
```

In Kubernetes, the same idea can be a startup probe.
The goal is to pay compilation once, before real traffic arrives.

Signal handling and graceful shutdown are still rough edges.
Test what happens when your orchestrator sends `SIGTERM`, and leave enough grace period for in-flight requests to finish.
In many setups, it is simpler to let the process exit and have the orchestrator retry than to implement graceful shutdown yourself.

Be careful about exposing a Julia service directly to the open internet.
Put it behind a mature reverse proxy, keep dependencies up to date, and pay attention to package vulnerabilities.
At the time of the talk, Avik recommended caution: fewer Julia services have been exposed to the open internet than services written in more established server languages, so the risks are less well understood.

## Cloud and Kubernetes

[AWS.jl](https://github.com/JuliaCloud/AWS.jl) and [Azure.jl](https://github.com/JuliaComputing/Azure.jl) provide Julia interfaces to their respective cloud platforms.
[Kuber.jl](https://github.com/JuliaComputing/Kuber.jl) wraps the Kubernetes API.

You do not always need them.
If your application starts, does some work, and shuts down, a Docker image and standard deployment tooling are enough.
Kuber.jl is useful when Julia code participates in scheduling or needs to inspect cluster state.
[JuliaHub](https://juliahub.com/) offers a managed way to run Julia workloads on Kubernetes.

Memory is usually a bigger cost than CPU on cloud services, and Julia processes tend to use more memory than comparable Go or C services.
Size containers for Julia's memory use, and keep precompilation out of the runtime path: precompiling uses much more memory than running, so a build machine may need 16 GB where the service needs 4 GB.

For parallel work, choose between multithreading inside a large machine and distributed processes across small machines.
[Distributed.jl](https://docs.julialang.org/en/v1/stdlib/Distributed/) is the standard library solution; [Dagger.jl](https://github.com/JuliaParallel/Dagger.jl) provides task-based distributed primitives.
The right split depends on the workload: some jobs are communication-heavy and prefer threads, while others scale better as many independent processes.

## Checklist

Before shipping a Julia application, check that:

- the Julia version is pinned in the build or deployment configuration;
- applications commit `Manifest.toml`, while libraries commit only `Project.toml`;
- internal packages live in a private registry;
- containers and system images are built ahead of time, not on startup;
- `JULIA_CPU_TARGET`, `JULIA_NUM_THREADS`, `JULIA_NUM_PRECOMPILE_TASKS`, and `--heap-size-hint` are set for the target environment;
- a PrecompileTools workload covers the hot path;
- a warm-up request runs before real traffic;
- graceful shutdown on `SIGTERM` has been tested;
- memory limits account for precompilation as well as runtime.

{% <note> %}
Deployment tooling moves quickly. The links below are the best starting points; check them for the current versions and flags.
{% </note> %}

## Further Resources

- [Lessons in deploying Julia to production services](https://www.youtube.com/watch?v=myZAp6vSgQA) by Avik Sengupta, JuliaCon 2026
- [PackageCompiler.jl documentation](https://julialang.github.io/PackageCompiler.jl/stable/)
- [JuliaC.jl](https://github.com/JuliaLang/JuliaC.jl)
- [PrecompileTools.jl](https://github.com/JuliaLang/PrecompileTools.jl)
- [Creating a Docker base image for faster deployments](https://discourse.julialang.org/t/creating-a-docker-base-image-for-faster-deployments/121165)
- [Julia manual: code loading](https://docs.julialang.org/en/v1/manual/code-loading/)
- [Julia manual: command-line options](https://docs.julialang.org/en/v1/manual/command-line-interface/)
- [Julia manual: environment variables](https://docs.julialang.org/en/v1/manual/environment-variables/)
