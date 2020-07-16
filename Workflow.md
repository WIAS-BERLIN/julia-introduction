Julia Workflow Tipps
====================

## Workflow and code structure

A part of these workflow tipps are focused on editing the code in a favorite editor and runing it from the command line. In particular, with Visual Studio Code or Atom, a different workflow is possible.


### 1. Develop everything in functions

Many available Julia examples and  the mindset influenced by Matlab or Python suggest  that one just  starts to  perform everything  in the global context of a script (e.g. `MyScript.jl`)

````
using Package1
using Package2

# action here
````

However, for Julia this is a bad idea for at  least two reasons

- Type-stable action of the JIT is possible only for functions, so this code does not optimize well
- One encounters precompilation time hiatus when running after each modified  version
````
$ julia MyScript.jl
````

The suggestion is from start to develop any code in functions. `MyScript.jl` then would look like:

````
using Package1
using Package2

function main(; kwarg1=1, kwarg2=2)
 # action here 
end
````

and would be invoked as

````
$ julia
julia> include("MyScript.jl")
julia> main(kwarg1=5)
````

In this   case you encounter the Read-Eval-Print-Loop (REPL) of Julia.You don't need to leave the session for restarting modified code (except in the case when you re-define a constant or a struct). Just reload the code by repeating the `include` statement.


### 2. Always use modules 

The previous example can be enhanced by wrapping the code of the script into a module:
````
Module MyScript

using Package1
using PackageN

function main(; kwarg1=1, kwarg2=2)
 # action here 
end

end
````

and would be invoked as

````
$ julia
julia> include("MyScript.jl")
julia> MyScript.main(kwarg1=5)
````

This has the advantage that you can run different scripts in the same session without
interfering namespaces.

### 3. Use Revise.jl

In the previous examples, re-loading the code requierd to re-run the include statement. The package `Revise.jl`  adds a command `includet` which triggers automatic recompile of modified code if the source code of the script file and of packages  used therein changes.

In order to set this up, place the following into the Julia startup file `.julia/config/startup.jl` in your home directory:

````
if isinteractive()
    try
        @eval using Revise
        Revise.async_steal_repl_backend()
    catch err
        @warn "Could not load Revise."
    end
end
```` 

You would then run:
````
$ julia -i
julia> includet("MyScript.jl")
julia> MyScript.main(kwarg1=5)
````
After changing `MyScript.jl`, just another  invocation of `MyScript.main`  would see the changes.




## Packages and Environments

### Using the Package manager

- Documentation on https://julialang.github.io/Pkg.jl

- Packages can be added by name or by git url

- In the REPL it is convenient to use the package manager mode which is invoked by `]`.

- Packages can be  added via name, url  and other possibilities
````
$ julia
julia>]
pkg> add Package1
pkg> add https://github.com/User2/Package2.jl
````

- If added by name, the Julia default registry is queried for the package.
This registry is the working copy of the git repository
`https://github.com/JuliaRegistries/General.git`

- In the moment a package is added, a local working copy is cloned  into `.julia/environments/v1.x`

- Their dependencies are resolved and all other needed packages are cloned as well

- It is possible to add another registry

````
pkg> registry add https://github.com/User3/Registry
````

### Project development and environments

When working with several projects, the approach above is not convenient as the global enviromnent
is cluttered with added packages from many projects.  It is therefore better to use environments.

Here, we assume that a _project_ is Julia code residing in a given directory `project_dir`, uses one or several other Julia packages and is not intended to be invoked from other Julia codes.

An environment is described by a `Project.toml` file in the project directory . One can set up an environment
in the following way:

````
$ cd project_dir
$ julia
$ pkg> activate .
$ pkg> add Package1
$ pkg> add PackageN
$ exit()
````
After setting up the environment like this, you can  perform

````
$ cd project_dir
$ julia --project=@.
````
and work in the environment. All packages added in this case affect only this enviroment. All packages added to the global environment still will be visible.

When it comes to versioning, the `Project.toml` file should be checked in along with the source code, so another project collaborator can easily establish a similar environment via

````
$ cd project_dir
$ julia --project=@.
$ pkg> instantiate
````
When instantiated, a `Manifest.toml` file appears which holds the information about the exact versions of all Julia packages used by the project.  Putting this under version control would allow to establish the exact versions of Julia packages necessary to establish a given result.  For shared code development this appears to be rather rigid. However, when it comes to publishing results, this option significantly improves the possibility to verify results.



## Package development
Here we assume that a _package_ is Julia code which resides in a given `package_dir` and is supposed to be used by other projects or packages. 
When developing Julia packages it is advisable to set the environment variable `JULIA_PKG_DEVDIR` to reasonable path. All Julia packages under development should  reside there.

### Starting a package

Create empty repo `NewPackageName.jl` on server with `git_url`
````
$ cd JULIA_PKG_DEVDIR
$ julia
pkg> generate NewPackageName
julia> exit()
cd NewPackageName
git init
git add remote origin git_url
git add .
````


Julia packages have a fixed structure which includes
- `Project.toml`: description of dependencies. This also provides the packages with its own environment during development.
  Furthermore, it contains the uuid used to identify the package and its name, author and version number.
- `src`: subdirectory containing code
   - `src/NewPackageName.jl`: Julia source file which must contain a module `NewPackageName`
- `test`: subdirectory containing test code
   - `test/runtests.jl`: script for running tests
- `docs`: subdirectory containing documentation 
   - `docs/make.jl`: script for builidng documentation based on Documenter.jl

### Using  a package under development

While the package is unregistered you can add it to the
environment of a project:
````
$ cd project_dir
$ julia --project=@.
pkg> add git_url
````
In this case, the version in the git repository on the server is used via a shallow clone (clone without versioning information) created in `$HOME/.julia/packages`. If you use
````
pkg> dev git_url
````
instead, a full clone is created in `$JULIA_PKG_DEVDIR` and used. If the corresponding subdirectory
in `JULIA_PKG_DEVDIR` does not exist, it will be checked out from `git_url`. The same would
happen with registered packages added by name.


## Registration of a new package or a new version

We assume to speak about `WIASJuliaRegistry` as an example, but generally this is true for all registries. 

### Without write access to this repository

Send an  email to  one of  the admins  containing the  git url  of the corresponding repository singnaling the advance of a new version.  The `Project.toml` file already should contain the new version number.

### With write access to this repository

Add the [LocalRegistry.jl](https://github.com/GunnarFarneback/LocalRegistry.jl) package to your environment
````
pkg>  add LocalRegistry
````


For a new package `X.jl` or a new version thereof, first obtain a clone, either by checking it out via its url, or by adding it to the environment, getting into develop mode and updating it:
````
pkg> add X
pkg> develop X
pkg> update X # not clear if really necessary
````

Then, you will be able to register a new version based on the updated version number in `Project.toml`:

````
julia> using X
julia> using LocalRegistry
julia> LocalRegistry.register(X,"WIASJuliaRegistry",push=true)
````
This assumes that the remote origin of the local clone in `.julia/registries/WIASJuliaRegistry' has been modified to `git@github.com:WIAS-BERLIN/WIASJuliaRegistry`.
The `push=true` can be omitted, in this case, `LocalRegistry.Register` results in a commit to the local copy which you can push later.

After this, do not forget to create a tag in your repository marking the version `x.y.z` of the package:
````
$ git tag vx.y.z
````

## Using the General registry
Registring packages in the general registry assumes full compliance to a number of rules:

- Package is deployed with automatic testing on github (at least this is the current practice) with [travis, deploydocs etc. enabled](https://juliadocs.github.io/Documenter.jl/stable/man/hosting/index.html)

- Don't forget compat info for  julia and all packages  in Project.toml
- Proper versioning sequence
- Must wait 3 days for merge for  new package
- Must wait $\approx$ 1 hour for new version

All of this is automatically handeled by the [JuliaRegistrator](https://github.com/JuliaRegistries/Registrator.jl)

<img src="https://raw.githubusercontent.com/JuliaRegistries/Registrator.jl/master/graphics/logo.png" width=200>

It needs to be installed as a github app in for the package repository.