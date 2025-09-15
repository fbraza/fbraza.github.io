# From Makefiles to Taskfiles: Streamlining Scala CI with GitHub Actions

I recently modernized my Scala project's build process by replacing Makefiles with [Task](https://taskfile.dev/) and integrating it into GitHub Actions. Here's what I learned.

## The Challenge

My Scala DFS library needed to compile and test across three Scala versions (2.12, 2.13, 3). The traditional approach would involve either:
- Complex Makefiles with SBT commands
- Verbose GitHub Actions with repeated SBT calls
- Manual testing across versions

## Enter Taskfile

Task provides a cleaner alternative to Make with YAML syntax. I created a simple `Taskfile.yml`:

```yaml
version: '3'

tasks:
  default:
    desc: "Run all tasks"
    deps: [clean, compile, test]

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

**Key insight**: The `default` task with dependencies means running just `task` executes the full pipeline.

## GitHub Actions Integration

The GitHub workflow became remarkably clean:

```yaml
name: CI

on:
  push:
    branches: [master, main]
  pull_request:
    branches: [master, main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
      - uses: arduino/setup-task@v2
        with:
          version: 3.x
      - uses: sbt/setup-sbt@v1
      - uses: actions/cache@v4
        with:
          path: |
            ~/.sbt
            ~/.ivy2/cache
            ~/.coursier/cache
          key: ${{ runner.os }}-sbt-${{ hashFiles('**/build.sbt') }}
          
      - run: task clean
      - run: task compile  
      - run: task test
```

## Testing Locally with wrkflw

I used [wrkflw](https://github.com/wrkflw/wrkflw) to test the workflow locally:

```bash
# Validate syntax
wrkflw validate .github/workflows/ci.yml

# Test execution (with limitations)
wrkflw run --runtime emulation .github/workflows/ci.yml
```

**Lesson learned**: wrkflw is excellent for validation but has limitations with complex GitHub Actions setups locally.

## Benefits Achieved

1. **Consistency**: Same commands locally (`task test`) and in CI
2. **Simplicity**: Clear, readable build definitions
3. **Maintainability**: Easy to add new Scala versions or tasks
4. **Speed**: SBT caching + dependency caching in CI

## The Hidden Win

The biggest surprise? Solving verbose logging from Hadoop MiniDFS in tests by adding a `log4j.properties` in `src/test/resources` - a side quest that dramatically cleaned up test output!

## Takeaways

- **Task > Make** for modern projects - better syntax, cross-platform
- **GitHub Actions + Task** creates clean, maintainable CI
- **Local testing tools** help but aren't perfect - trust the real CI environment
- **Small improvements compound** - cleaner builds, cleaner logs, better DX