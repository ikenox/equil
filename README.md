# Equil

Equil is a very simple utility for setup/provisioning your computer.  
It equilibrates your computer to the state you need.

# Why Equil

Equil is for the person who wants to automate provisioning of their personal computer and feels like:

- Pure shell scripts or Makefile don't have enough readability and writability for provisioning task definition.
- Popular provisioning tools (e.g. Ansible, Chef, ...) have rich features but a little harder to learn.

Equil is just a very thin wrapper of shell scripts for provisioning task definition, written in Ruby DSL.

# Usage

## Installation

You don't have to install because its source is small single file and got from the web by `curl` command when execution.

## Prepare task definition file

To provision your computer, you have to prepare task definition file, as follows.

```ruby
# equil-task.rb

def equil
  task :essentials do
    task :homebrew do
      task :install_brew, if_err('which brew'),
        '/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"'
      task :tap_cask, if_err('brew tap | grep caskroom/cask'), 'brew tap caskroom/cask'
    end
    task :git do
      task if_err('which git'), 'brew install git'
      task 'ln -si ~/.dotfiles/git/gitignore ~/.gitignore'
      task 'ln -si ~/.dotfiles/git/gitconfig ~/.gitconfig'
      task if_err("git config user.name"), -> {
        print "please type your git user.name: "
        name = STDIN.gets.chomp
        "git config -f ~/.gitconfig.local user.name '#{name}'"
      }
    end
  end

  task :some_optional do
    task 'echo "this is an optional task"'
  end
end
```

This is Ruby DSL, so if you are familiar with Ruby, you can write the definition as Ruby program.

If you need more practical example, see [ikenox/dotfiles/provision-tasks.rb](https://github.com/ikenox/dotfiles/blob/master/provision-tasks.rb).

### Task definition details

The basic task definition syntax is as follows.

```txt
task [TASK_NAME,] [EXEC_CONDITION,] {COMMAND|LAMBDA_FUNC|TASK_LIST}
```
`TASK_NAME` and `EXEC_CONDITION` are optional so it can be omitted.

Tell Equil on the each tasks,

1. What state the task needs to be executed (`EXEC_CONDITION`)
    - Example of `EXEC_CONDITION`:
        ```ruby
        if_err('which git')
        ```
2. What needs to be executed (`COMMAND` or `LAMBDA_FUNC` or `TASK_LIST`)
    - Example of `COMMAND`:
       ```ruby
       'brew install git'
       ```
    - Example of `LAMBDA_FUNC`:
       ```ruby
       -> {
         print "please type your git user.name: "
         name = STDIN.gets.chomp
         "git config -f ~/.gitconfig.local user.name '#{name}'"
       }
       ```
       `-> { ... }` is `LAMBDA_FUNC`, and this lambda function should return a string of shell commands.
    - Example of `TASK_LIST`:
        ```ruby
        do
          task ...
          task ...
          task ...
          ...
        end
        ```



## Task execution

If your task definition file `equil-task.rb` is on local computer, do the following to execute task `:essentials`.

```sh
$ bash -c 'ruby -e "$({ cat ./equil-task.rb; curl -fsSL https://raw.githubusercontent.com/ikenox/equil/0.1.0/equil.rb; })" essentials'
```

or, if your task definition file `equil-task.rb` is on the web (e.g. On GitHub), do the following to execute task `:essentials`.

```sh
$ bash -c 'ruby -e "$({ curl -fsSL https://raw.github.com/your/repository/master/equil-task.rb; curl -fsSL https://raw.githubusercontent.com/ikenox/equil/0.1.0/equil.rb; })" essentials'
```

### Output

```
[TASK]example -> EXECUTE
    [TASK]homebrew -> EXECUTE
        [TASK]install_brew -> SKIP
        [TASK]tap_cask -> SKIP
    [TASK]git -> EXECUTE
        [TASK]anonymous_task -> SKIP
        [TASK]anonymous_task -> EXECUTE
> ln -si ~/.dotfiles/git/gitignore ~/.gitignore
replace /Users/naoto.ikeno/.gitignore? y
        [TASK]anonymous_task -> EXECUTE
> ln -si ~/.dotfiles/git/gitconfig ~/.gitconfig
replace /Users/naoto.ikeno/.gitconfig? y
        [TASK]anonymous_task -> EXECUTE
please type your git user.name: Bob
> git config -f ~/.gitconfig.local user.name 'Bob'

[FINISHED]
```
- Each tasks are executed or skipped following `EXEC_CONDITION`.
- If the task has no `EXEC_CONDITION`, it will be always executed.

# Advanced

## Task alias

You can define a task alias to avoid repeat similar definition.

```ruby
def equil
  task :install_some_packages do
    task :brew_install_git, if_err("which git"), 'brew install git'
    task :brew_install_zsh, if_err("which zsh"), 'brew install zsh'
    task :brew_install_docker, if_err("which docker"), 'brew install docker'
  end
end
```

This is equivalent to as follows.

```ruby
def equil
  task :install_some_packages do
    task brew 'git'
    task brew 'zsh'
    task brew 'docker'
  end
end

def brew(package)
    task_alias "brew_install_#{package}".to_sym, if_err("which #{package}"), "brew install #{package}"
end
```

Task alias definition syntax is as follows. `TASK_NAME` and `EXEC_CONDITION` are optional.

```txt
task_alias [TASK_NAME,] [EXEC_CONDITION,] {COMMAND|LAMBDA_FUNC|TASK_LIST}
```

# License

MIT
