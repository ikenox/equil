# Equil

Equil is a very simple utility for setup/provisioning your computer.  
It equilibrates your computer to the state you need.

# Why Equil

Equil is for the person who wants to automate provisioning of their personal computer and who feels like:

- Pure shell scripts or Makefile doesn't have enough readability and writability for provisioning task definition.
- Popular provisioning tools (e.g. Ansible, Chef, ...) have rich features but a little harder to learn.

Equil is just a very thin wrapper of the shell scripts for provisioning task definition, written in Ruby DSL.

# Usage

1. Install

    You don't have to install because its source is small single file and you can get it from the web by `curl` command when execution.

2. Prepare task definition file

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
            "git config user.name '#{name}'"
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

3. Task execution

    If you put the task definition file `equil-task.rb` on your local computer, do the following to execute task `:essentials`.

    ```sh
    # bash
    $ ruby -e "$({ cat ./equil-task.rb; curl -fsSL https://raw.githubusercontent.com/ikenox/equil/0.1.0/equil.rb; })" essentials
    ```

    otherwise, if you put the task definition file on the web (e.g. on GitHub), do the following to execute task `:essentials`.

    ```sh
    # bash
    $ ruby -e "$({ curl -fsSL https://raw.github.com/your/repository/master/equil-task.rb; curl -fsSL https://raw.githubusercontent.com/ikenox/equil/0.1.0/equil.rb; })" essentials
    ```

4. Result

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

    Each tasks are executed or skipped depending on the condition.

# Details

## Task definition syntax

The basic task definition syntax is as follows.

```txt
task [TASK_NAME,] [EXEC_CONDITION,] {COMMAND|LAMBDA_FUNC|TASK_LIST}
```
`TASK_NAME` and `EXEC_CONDITION` are optional so it can be omitted. If the task definition has no `EXEC_CONDITION`, the task will be always executed.

Tell Equil on the each tasks,

1. What state the task needs to be executed (`EXEC_CONDITION`)

    Example `EXEC_CONDITION`:
    ```ruby
    if_err('which git')
    ```
2. What needs to be executed (`COMMAND` or `LAMBDA_FUNC` or `TASK_LIST`)

   Example `COMMAND`:
   ```ruby
   'brew install git'
   ```
   
   Example `LAMBDA_FUNC`:
   ```ruby
   -> {
     print "please type your git user.name: "
     name = STDIN.gets.chomp
     "git config -f ~/.gitconfig.local user.name '#{name}'"
   }
   ```
   `-> { ... }` is `LAMBDA_FUNC`, and this lambda function should return a string of shell commands.
   
   Example `TASK_LIST`:
    ```ruby
    do
      task ...
      task ...
      task ...
      ...
    end
    ```

# Advanced

## Task alias

You can define a task alias to avoid repeat similar definition.

The following two are equivalent.

a. 

```ruby
def equil
  task :install_some_packages do
    task :brew_install_git, if_err("which git"), 'brew install git'
    task :brew_install_zsh, if_err("which zsh"), 'brew install zsh'
    task :brew_install_docker, if_err("which docker"), 'brew install docker'
  end
end
```

b. 

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
