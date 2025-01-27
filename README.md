# Synopsis

`lib/lockfile.rb`: A ruby library for creating perfect and NFS safe lockfiles

`bin/rlock`: Ruby command line tool which uses this library to create lockfiles
and to run arbitrary commands while holding them.

For example: `rlock lockfile -- cp -r huge/ huge.bak/`

Run `rlock -h` for more info.

# Basic algorithm

* Create a globally uniq filename in the same filesystem as the desired
  lockfile - this can be nfs mounted.
* link(2) this file to the desired lockfile, ignore all errors.
* Stat the uniq filename and desired lockfile to determine if they are the
  same, use only stat.rdev and stat.ino - ignore stat.nlink as NFS can cause
  this to report incorrect values.
* If same, you have lock. Either return or run optional code block with
  optional refresher thread keeping lockfile fresh during execution of code
  block, ensuring that the lockfile is removed.
* If not same try again a few times in rapid succession (poll), then, if
  this fails, sleep using incremental backoff time. Optionally remove
  lockfile if it is older than a certain time, timeout if more than a certain
  amount of time has passed attempting to lock file.

# Basic usage

## With a block:
```ruby
Lockfile.new('file.lock') do
  p 42
end
```

## With a begin/ensure block:
```ruby
  lockfile = Lockfile.new 'file.lock'
  begin
    lockfile.lock
    p 42
  ensure
    lockfile.unlock
  end
```

## With options:
```ruby
require 'pstore'            # which is NOT nfs safe on its own

opts = {                    # the keys can be symbols or strings

  :retries        => nil,   # we will try forever to acquire the lock

  :sleep_inc      => 2,     # we will sleep 2 seconds longer than the
                            # previous sleep after each retry, cycling from
                            # min_sleep upto max_sleep downto min_sleep upto
                            # max_sleep, etc., etc.

  :min_sleep      => 2,     # we will never sleep less than 2 seconds

  :max_sleep      => 32,    # we will never sleep longer than 32 seconds

  :max_age        => 3600,  # we will blow away any files found to be older
                            # than this (lockfile.thief? #=> true)

  :suspend        => 1800,  # if we steal the lock from someone else - wait
                            # this long to give them a chance to realize it

  :refresh        => 8,     # we will spawn a bg thread that touches file
                            # every 8 sec. this thread also causes a
                            # StolenLockError to be thrown if the lock
                            # disappears from under us - note that the
                            # 'detection' rate is limited to the refresh
                            # interval - this is a race condition

  :timeout        => nil,   # we will wait forever

  :poll_retries   => 16,    # the initial attempt to grab a lock is done in a
                            # polling fashion, this number controls how many
                            # times this is done - the total polling attempts
                            # are considered ONE actual attempt (see retries
                            # above)

  :poll_max_sleep => 0.08,  # when polling a very brief sleep is issued
                            # between attempts, this is the upper limit of
                            # that sleep timeout

  :dont_clean     => false, # normally a finalizer is defined to clean up
                            # after lockfiles, settin this to true prevents this

  :dont_sweep     => false, # normally locking causes a sweep to be made. a
                            # sweep removes any old tmp files created by
                            # processes of this host only which are no
                            # longer alive

  :debug => true,           # trace execution step on stdout
}

pstore = PStore.new 'file.db'
lockfile = Lockfile.new 'file.db.lock', opts
lockfile.lock do
  pstore.transaction do
    pstore[:last_update_time] = Time.now
  end
end
```

## With debugging:
```ruby
Lockfile.debug = true
Lockfile.new('file.lock') do
  p 42
end
```
You can also enable debugging with the `LOCKFILE_DEBUG` environment variable,
e.g.: `LOCKFILE_DEBUG=true rlock lockfile`

## Simplified interface: no lockfile object required
```ruby
Lockfile('lock', :retries => 0) do
  puts 'only one instance running!'
end
```

# Samples

  * see samples/a.rb
  * see samples/nfsstore.rb
  * see samples/lock.sh
  * see bin/rlock

# Author

  Ara T. Howard

# Email

  Ara.T.Howard@noaa.gov

# Bugs

  bugno > 1 && bugno < 42

# History

### 2.0.0
- lock fires up a refresher thread when called without a block

### 1.4.3
- fixed a small non-critical bug in the require guard

### 1.4.2
- upped defaults for `max_age` to 3600 and suspend to 1800.
- tweaked a few things to play nice with rubygems

### 1.4.1
- Mike Kasick <mkasick@club.cc.cmu.edu> reported a bug whereby false/nil
  values for options were ignored. patched to address this bug
- added Lockfile method for high level interface sans lockfile object
- updated rlock program to allow nil/true/false values passed on command
  line. eg `rlock --max_age=nil lockfile -- date --iso-8601=seconds`

### 1.4.0
- gem'd up
- added `Lockfile::create` method which atomically creates a file and opens
  it:
```ruby
    Lockfile::create("atomic_even_on_nfs", "r+") do |f|
      f.puts 42
    end
```

- arguments are the same as those for `File::open`. note that there is no way
  to accomplish this otherwise since `File::O_EXCL` fails silently on nfs,
  flock does not work on nfs, and posixlock (see raa) can only lock a file
  after it is open so the open itself is still a race.

### 1.3.0
- added sweep functionality. hosts can clean up any tmp files left around
  by other processes that may have been killed using -9 to prevent proper
  clean up. it is only possible to clean up after processes on the same
  host as there is no other way to determine if the process that created the
  file is alive. in summary - hosts clean up after other processes on that
  same host if needed.
- added attempt/try_again methods

### 1.2.0
- fixed bug where stale locks not removed when retries == 0

### 1.1.0
- unfortunately i've forgotten

### 1.0.1
- fixed bugette in sleep cycle where repeated locks with same lockfile would
  not reset the cycle at the start of each lock

### 1.0.0
- allow rertries to be nil, meaning to try forever
- default timeout is now nil - never timeout
- default refresh is now 8
- fixed bug where refresher thread was not actually touching lockfile! (ouch)
- added cycle method to timeouts
    1-2-3-2-1-2-3-1...
  pattern is constructed using min_sleep, sleep_inc, max_sleep

### 0.3.0
- removed use of yaml in favour of hand parsing the lockfile contents, the
  file as so small it just wasn't worth and i have had one core dump when yaml
  failed to parse a (corrupt) file

### 0.2.0
- added an initial polling style attempt to grab lock before entering normal
  sleep/retry loop. this has really helped performance when lock is under
  heavy contention: i see almost no sleeping done by any of in the interested
  processes

### 0.1.0
- added ability of Lockfile.new to accept a block

### 0.0.0
- initial version
