### URLs:

  http://raa.ruby-lang.org/project/session/
  http://www.codeforpeople.com/lib/ruby/session/


### Name:

  `Session
    ::Sh
    ::Bash
    ::Shell
    ::IDL`

### Synopsis:

`Session::*` offers a set of classes built upon `Open3::popen3` for driving
external progams via pipes.  It offers a significant abstraction over
`Open3::popen` in that the stdout/stderr of each command sent can be deliniated:

open3:

```ruby
  i,o,e = Open3::popen3 '/bin/sh'

  i.puts 'ls'
  i.puts 'echo 42'
```

Now, how to determine the boundry between the output from `ls` and `echo`?
the only (simple) way is start a process for each command:

```ruby
  i,o,e = Open3::popen3 '/bin/sh'
  i.puts 'ls'
  i.close
  stdout, stderr = o.read, e.read

  i,o,e = Open3::popen3 '/bin/sh'
  i.puts 'echo 42'
  i.close
  stdout, stderr = o.read, e.read
```

### Session:

```ruby
sh = Session::new

stdout, stderr = sh.execute 'ls'
stdout, stderr = sh.execute 'echo 42'
```

Both `stderr` and `stdout` can be redirected, and the `exit_status` of each command
is made available:

```ruby
bash = Session::Bash.new
stdout, stderr = StringIO::new, StringIO::new

bash.execute 'ls', :stdout => stdout, :stderr => stderr
# bash.execute 'ls', 1 => stdout, 2 => stderr           # same thing
# bash.execute 'ls', :o => stdout, :e => stderr         # same thing

exit_status = bash.exit_status
```

A block form can be used to specify a callback to be invoked whenever output
has become availible:

```ruby
bash = Session::Bash.new

bash.execute( 'long_running_command.exe' ) do |out, err|
  logger << out if out
  elogger << err if err
end
```

Sessions are Thread safe (in the sense that they do not block on io
operations) allowing commands spawned from guis to update widgets with output
while running in the background.

```ruby
button.configure 'action' => lambda do
  sh = Session::new
  sh.execute(cmd) do |o,e|
    out_widget.update o if o
    err_widget.update e if e
  end
end
```

### Samples:

see [sample/*](/sample)

### History:
##### 3.1.0:
- patches from @headius

##### 3.0.0
- move to github

##### 2.4.0:
- added ability to specify stdin for `Session::Bash` and `Session::Sh`

```ruby
        sh = Session::new

        sh.execute 'cat', :stdin => io
        sh.execute 'cat', :stdin => string
        sh.execute 'cat', :stdin => stringio
```

##### 2.3.0:
- fixed warning of @debug being un-initialized

##### 2.2.0:
- added a private munged version of `Open3::open3`.  the builtin one causes
  the child process to become a child of init, this was very inconvenient
  because it was sometimes hard to crawl proces trees - the parent was lost.
  now the seesion is a child process that has been detached using
  `Process::detach`.  this results in less suprising behaviour; for instance
  sending signal TERM to a process results in any sessions it had open dying
  as well.  you can use `Session::use_open3=true` or
  `ENV['SESSION_USE_OPEN3']='1'` for the old behaviour if you need it.
- added `Session::Bash::Login` class.  this class opens a session which has
  all the normal settings of a bash loging shell (.bashrc is sourced).  this
  if often convenient as paths, aliases, etc. are set as normal.
- moved the Spawn module inside the Session module. Now the Session module
  is the namespace for everything so using session pollutes namespace less.

##### 2.1.9:
- fixed bug where setting track history after creation caused later failure in
  execute (@history =[] only in ctor).  thanks leon breedt
  <bitserf@gmail.com>!
- updates to README
- included session-x.x.x.rpa file - thanks batsman <batsman.geo@yahoo.com>
- to_str/to_s/to_yaml for History/Command is now valid yaml (updated samples
  to reflect this)
- inspect for History/Command is now ruby's default

##### 2.1.8:
- greatly simplified read loop using two reader threads, one for stderr and
  one for stdout alongside a mutex to protect data.  this streamlined the code
  alot vs. the old select method including allowing removal of the linbuffer
  class.  the interface remains exactly as before however.

##### 2.1.7:
- improved thread safe non-blocking read method
- gemspec

##### 2.1.6:
- wrapped send_command in a Thread (send async) so output processing can
  commend immeadiately.  this was o.k. before, but had strange behaviour when
  using popen3 from threads.  thanks to tanaka akira for this suggestion.
- iff `ENV['SESSION_USE_SPAWN']` is set Session uses `Spawn::spawn` instead of
  `Open3::popen3`. Also noted that spawn seems to be a bit faster.
- added tests for threads.
- run `sh SESSION_USE_SPAWN=1 ruby test/session.rb` to test using spawn
- added test for idl so it's test is not run if system doesn't have it, all
  that should be required for `ruby test/session.rb` should be sh'
- removed sample/tcsh and note about Tcsh and Csh in README - stderr
  redirection/separation is flaky in those shells

##### 2.1.5:
- added `Session.use_spawn=`, `AbstractSession.use_spawn=`, and an `:use_session=>
  option` to AbstractSession#initialize. If any of them are set the code uses
  `Spawn::spawn` to create external processes instead of `Open3::popen3`.
  `Spawn::spawn` uses named pipes (fifos) for IPC instead of forking and pipes.
  the fork used in popen3 can cause strange behaviour with multi-threaded apps
  (like a tk app).  see source for details

##### 2.1.4:
- added `Thread.exclusive{}` wrapper when io is read to works in multi-threaded apps


### Author:

ara.t.howard@gmail.com
