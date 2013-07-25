This is the Garantia Data custom Redis version, which is based on the publicly
available Redis 2.6 with various additional features we use internally.

This file provides some high-level documentation of the additional features.

Conditional Replication
-----------------------

Conditional replication allows slaves and masters to avoid replication if the
data on both ends is identical.  This is done by maintaining a 'dbversion'
value which gets updated on every write (a kind of incremental hash). The
SYNC command is extended to allow slaves to announce their dbversion and
request replication from scratch only if it mismatches the master's dbversion.

The 'dbversion' value is stored in the RDB, which makes it incompatible with
stock Redis RDB files.


Script Replication
------------------

This version of Redis provides a mechanism to store scripts in RDB files. As
a results, scripts are as persistent as the database itself.  Also, this
allows slave servers to receive scripts when initiating a replication link.

In addition, Redis behaves a bit differently in order to fully support
script replication:
- EVAL command with a DB-modifying script will be replicated as-is.
- EVAL command with a non-DB modifying script will be replicated as a 
  SCRIPT LOAD command (to save processing on the slave).
- The same applies to EVALSHA, which is replicated as an EVAL or SCRIPT LOAD.


Different BGSAVE Types
----------------------

Redis uses the same BGSAVE mechanism for persistence and for slave-trigerred
replication.  In order for this to work well while persistence policies are
managed, we introduce the following changes:

- A 'syncdbfilename' configuration parameter allows us to use a different
  RDB file for SYNC handling (on the master side).  If not specified, the
  'dbfilename' paramter will be used instead.

- When 'syncdbfilename' is defined, background saves performed for the
  purpose of replication don't overwrite the usual RDB file and do not
  affect the 'rdb_last_save_time', 'rdb_changes_since_last_save',
  and 'rdb_last_bgsave_...' fields.

In addition, we provide a dedicated BGSAVETO command which triggers a
background save operation to a specific file with optional database number
argument. This operation also doesn't modify the fields above.


Slave output buffer throttling
------------------------------

The slave output buffer throttling is used to slow down incoming requests
when a slave initiates replication and the output buffer grows too fast
and beyond certain thresholds.

Using the 'slave-output-buffer-throttling' configuration parameter, it is
possible to throttle down write requests in order to control the rate in
which slave server buffers grow during replication.


Additional Redis Configuration Parameters
-----------------------------------------

load-on-startup
  Provides the ability to disable auto-loading of RDB/AOF files on startup,
  even if they exist.  This is useful on slaves that would immediately need
  to replicate from masters anyway.

preload-file
  Provides a mechanism to load an arbitrary rdb or aof file on server
  startup, regardless of of snapshot/appendonly configuration.

optionally-include
  Works like 'include' but silently ignores non-existing files.


Additional Statistics
---------------------

keyspace_read_hits
keyspace_write_hits
keyspace_read_misses
keyspace_write_misses
  These parameters replace the original keyspace_hists/keyspace_misses and
  provide more information about hits/miss ratios of read-only vs. update
  operations.

aof_rewrites
  Counts the number of AOF rewrites performed.

rdb_saves
  Counts the number of regular background saves performed.  This includes
  explicit BGSAVE as well as automatic saves due to 'save' configuration
  parameter thresholds, but not saves to dedicated SYNC file or BGSAVETO.


New/Modified Redis Commands
---------------------------

SYNC <dbversion>
  When sent by a slave, the master will compare the requested dbversion to
  its own.  If identical, it will respond with "+INSYNC" and skip creation
  of RDB file.  Instead, it will begin sending the command stream as if
  the RDB creation and transmission was complete.

  dbversion is a hash-like value that gets updated whenever a write
  operation is performed and represents a database "state".  It is also
  stored in RDB files.

SYNCNOW
  Initiates a slave-like SYNC operation, but requires the server to proceed
  only if no other SYNC is in progress.  Essentially this means the RDB we
  will be receiving represents the current database state without applying
  any additional queued commands.

  The server responds with an ERROR if another SYNC or BGSAVE are already
  in progress.  Otherwise, a 2-part multibulk is returned immediately with
  the first part being a "+SYNC STARTED" status.  The 2nd part is sent
  as soon as the RDB is created and includes the contents of the RDB, and
  the connection continues to behave as a standard slave connection. 

BGSAVETO <filename>
  Initiate an ad-hoc BGSAVE operation to a specific file.  The file will
  be created in the working directory specified by the global 'dir'
  parameter.  This background save operation is transparent and does not
  update the regular save parameters (rdb_last_save_time,
  rdb_changes_since_last_save, rdb_last_bgsave_status, etc.) so it does not
  disrupt on-going persistence policy.

DRAIN
  Provide a safety mechanism to guarnatee that all slaves are up to date.
  This includes: (a) verifying that all slaves are online; (b) verifying
  that the network buffers of slave connections are empty.

  DRAIN returns "+DRAINED" if all slaves are up-to-date as described above.
  Otherwise, "+DRAINING" is returned and the server enters a draining-state
  in which it will not accept any command other than INFO, PING or DRAIN
  (which is expected to be used by the caller to poll the server state).

HIDECONNECTION
  Flags the current client connection hidden, so that its commands will
  not show up in MONITOR output.

MONITOR TRUNCATED
  The 'TRUNCATED' suffix flag can be appended to the standard MONITOR
  command in order to request limited-length command output.  Currently
  all commands are truncated to max. 1024 bytes (hard coded).

SLOWLOG GET
  The slowlog now includes some additional information showing values
  which affected the calculation complexity of the slow command. This
  is useful if the command operated on keys that don't exist anymore
  or if the command had more parameters than allowed in the slow log.


Build Tree Changes
------------------

Targets are built into architecture-specific directories named 'obj.<ARCH>'.
It is possible to explicitly specify target architecture by passing a
BUILD_ARCH= variable to make.  For example, to build both 64-bit and 32-bit
versions in the same source tree in Linux:

  make BUILD_ARCH=x86_64
  make BUILD_ARCH=i686

When installed, i686 executables also have a .i686 suffix to allow simple
installation into a single location:

  make BUILD_ARCH=x86_64 install
  make BUILD_ARCH=i686 install

Will (by default) result with a /usr/local/bin/redis-server 64-bit version
and a /usr/local/bin/redis-server.i686 32-bit version.


[![githalytics.com alpha](https://cruel-carlota.pagodabox.com/ce4c1161c17a84e88ed541d89e4edf5f "githalytics.com")](http://githalytics.com/GarantiaData/redis)

