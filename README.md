# Flock

Automated deployment of your Swift project to servers. Inspired by [Capistrano](https://github.com/capistrano/capistrano).

## Installation
### Homebrew
```bash
brew install jakeheis/repo/flock
```
### Manual
```bash
git clone https://github.com/jakeheis/FlockCLI
cd FlockCLI
swift build -c release
ln -s .build/release/FlockCLI /usr/bin/local/flock
```

## Usage
### Set up
To start using Flock, run:
```bash
flock --init
```

Flock will create a number of files:

#### Flockfile
The Flockfile specifies which tasks and configurations you want Flock to use. In order to use some other tasks, just import the task library and tell Flock to use them:
```swift
import Flock
import VaporFlock

Flock.use(Flock.Deploy) // Located in Flock
Flock.use(Flock.Tools) // Located in Flock
Flock.use(Flock.Vapor) // Located in VaporFlock

...
```
If the tasks are in a separate library (as `Flock.Vapor` is above), you'll also need to add `VaporFlock` as a dependency as described in the next section.

By default, `Flock.Deploy` and `Flock.Tools` will be used. Using `Flock.Deploy` allows you to run `flock deploy` which deploys your project onto your servers. Using `Flock.Tools` allows you run `flock tools` which will install the necessary tools for your swift project to run on the server.

If you want to add additional configuration environments (beyond "staging" and "production), you can do that here in the `Flockfile`. First run `flock --add-env Testing` and then modify the `Flockfile`:
```swift
...

Flock.configure(.always, with: Always()) // Located at deploy/Always.swift
Flock.configure(.env("production"), with: Production()) // Located at deploy/Production.swift
Flock.configure(.env("staging"), with: Staging()) // Located at deploy/Staging.swift
Flock.configure(.env("test"), with: Testing()) // Located at deploy/Testing.swift

...
```

#### config/deploy/FlockDependencies.json
This file contains your Flock dependencies. To start this only contains `Flock` itself, but if you want to use third party tasks you can add their repositories here. You specify the repository's URL and version (there are three ways to do this):
```json
{
   "dependencies" : [
       {
           "name" : "https://github.com/jakeheis/Flock",
           "version": "0.0.1"
       },
       {
           "name" : "https://github.com/jakeheis/VaporFlock",
           "major": 0
       },
       {
           "name" : "https://github.com/someone/something",
           "major": 2,
           "minor": 1
       }
   ]
}
```

#### config/deploy/Always.swift
This file contains configuration which will always be used. This is where configuration info which is needed regardless of environment. Some fields you'll need to update before using Flock:
```
Config.projectName = "ProjectName"
Config.executableName = "ExecutableName"
Config.repoURL = "URL"
```

#### config/deploy/Production.swift and config/deploy/Staging.swift
These files contain configuration specific to the production and staging environments, respectively. They will only be run when Flock is executed in their environment (using `flock task -e staging`). Generally this is where you'll want to specify your production and staging servers. There are two ways to specify a server:
```swift
func configure() {
      Config.SSHKey = "/Location/of/my/key"
      Servers.add(ip: "9.9.9.9", user: "user", roles: [.app, .db, .web])

      // Or, if you've added your server to your `.ssh/config` file, you can use this shorthand:
      Servers.add(SSHHost: "NamedServer", roles: [.app, .db, .web])
}
```

#### .flock
You can (in general) ignore all the files in this directory.

### Running tasks

You can see the available tasks by just running `flock` with no arguments. Running a task is as easy as:
```bash
flock deploy # Run the deploy task
flock deploy:build # Run the build task located under the deploy namespace
flock tools -e staging # Run the tools task using the staging configuration
flock vapor:start -n # Do a dry-run of the start task located under the Vapor namespace - print the commands that would be executed without actually executing anything
```

### Writing your own tasks
Start by running:
```bash
flock --create db:migrate # Or whatever you want to call your new task
```

This will create file at config/deploy/DbMigrateTask.swift with the following contents:
```swift
import Flock

extension Flock {
   public static let <NameThisGroup>: [Task] = [
       MigrateTask()
   ]
}

// Delete if no custom Config properties are needed
extension Config {
   // public static var myVar = ""
}

class MigrateTask: Task {
   let name = "migrate"
   let namespace = "db"

   func run(on server: Server) throws {
      // Do work
   }
}
```

After running this command, make sure you:
1. Replace <NameThisGroup> at the top of your new file with a custom name
2. In your Flockfile, add `Flock.use(WhateverINamedIt)`

#### Hooking

If you wish to hook your task onto another task (i.e. always run this task before/after another task, just add an array of hook times to your Task:
```swift
class MigrateTask: Task {
   let name = "migrate"
   let namespace = "db"
   let hookTimes: [HookTime] = [.after("deploy:build")]
   
   func run(on server: Server) throws {
      // Do work
   }
}
```

#### Invoking another task
```swift
func run(on server: Server) throws {
      try invoke("other:task")
}
```

## Server dependencies
If your Swift server uses one of these popular libraries, there are Flock dependencies already available which will hook into the deploy process and restart the server after the new release is built.

- [VaporFlock](https://github.com/jakeheis/VaporFlock)
- [PerfectFlock](https://github.com/jakeheis/PerfectFlock)
- [ZewoFlock](https://github.com/jakeheis/ZewoFlock)
- [KituraFlock](https://github.com/jakeheis/KituraFlock)
