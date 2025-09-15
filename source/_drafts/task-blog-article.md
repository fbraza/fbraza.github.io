# Task: The Makefile Alternative That's Not Obvious at First

I recently switched from Makefiles to [Task](https://taskfile.dev/) for my Scala project, and while Task is great, the execution model isn't immediately clear. Here's what I learned.

## The Setup

Starting with a simple Scala project that needs to compile and test across multiple Scala versions (2.12, 2.13, 3), I created this basic `Taskfile.yml`:

```yaml
version: '3'

tasks:
  compile:
    desc: "Compile for all Scala versions"
    cmds:
      - sbt ++2.12.20 compile
      - sbt ++2.13.15 compile
      - sbt ++3.3.1 compile

  test:
    desc: "Test for all Scala versions"
    cmds:
      - sbt ++2.12.20 test
      - sbt ++2.13.15 test
      - sbt ++3.3.1 test

  clean:
    desc: "Clean build artifacts"
    cmds:
      - sbt clean
```

## The Confusion

Running `task` gave this "error":

```
task: Task "default" does not exist

task: Available tasks for this project:
* clean:         Clean build artifacts
* compile:       Compile for all Scala versions
* test:          Test for all Scala versions
```

**This isn't actually an error!** It's Task telling you there's no default task, then helpfully listing available tasks.

## How Task Actually Works

Unlike Make where you can run all targets, Task requires explicit task names:

- `task compile` - runs the compile task
- `task test` - runs the test task
- `task clean compile test` - runs multiple tasks in sequence
- `task -p compile test` - runs tasks in parallel

## The Solution: Default Task

To get "run everything" behavior like `make all`, add a default task:

```yaml
version: '3'

tasks:
  default:
    desc: "Run all tasks"
    deps: [clean, compile, test]
    
  # ... other tasks
```

Now `task` (no arguments) runs all tasks via dependencies.

## Key Takeaways

1. **No implicit "run all"** - you must specify which tasks to run
2. **Default task is optional** but useful for common workflows
3. **The "error" message** is actually helpful information
4. **Dependencies (`deps`)** are the clean way to orchestrate multiple tasks

Task is more explicit than Make, which can be both good (clearer) and confusing (more verbose) depending on your expectations.