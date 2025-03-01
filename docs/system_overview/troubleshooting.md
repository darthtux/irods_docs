#

Some of the commonly encountered iRODS errors along with troubleshooting steps are discussed below.

## The Server Log

As of iRODS 4.3.0, the server log file is managed by syslog. When unexpected behavior occurs, inspecting the log file is the first place that should be checked. By default, the log file is located at **/var/log/irods/irods.log**.

iRODS provides default configuration files for **rsyslog** and **logrotate**. They can be found at the following locations:

- /etc/rsyslog.d/00-irods.conf
- /etc/logrotate.d/irods

Other implementations of syslog can be used, but that is out of scope of this documentation.

Each line within the log file is formatted as JSON. The following properties are guaranteed to exist:
```json
{
    // The server component that generated the message (e.g. "rule_engine", "database", "api", etc).
    "log_category": string,

    // This property can be ignored.
    "log_facility": string,

    // The severity of the message.
    //
    // This will be one of the following:
    // - trace
    // - debug
    // - info
    // - warn
    // - error
    // - critical
    "log_level": string,

    // The message detailing what actually happened within the server.
    "log_message": string,

    // The iRODS server that generated the message.
    "server_host": string,

    // The PID of the process that generated the message.
    "server_pid": integer,

    // The time when the message was generated.
    "server_timestamp": string,

    // The type of process that generated the message.
    //
    // This will be one of the following:
    // - server
    // - agent_factory
    // - agent
    // - delay_server
    "server_type": string
}
```

In addition to those, the following properties will be included if available:
```json
{
    // The human-readable name of the API operation.
    "request_api_name": string,

    // The integer value that identifies the API operation.
    "request_api_number": integer,

    // The API version of the iRODS client which sent the request (e.g. "d").
    "request_api_version": string,

    // The iRODS user who sent the API request.
    "request_client_user": string,

    // The IP or hostname of the client which sent the API request.
    "request_host": string,

    // The proxy iRODS user carrying out the operation on behalf of "request_client_user".
    "request_proxy_user": string,

    // The version of the iRODS client which sent the request (e.g. "rods4.3.0").
    "request_release_version": string
}
```

If an operation results in a program crash and a stacktrace is generated, the following properties will be included if available:
```json
{
    // The PID of the iRODS agent that crashed and generated the stacktrace.
    "stacktrace_agent_pid": integer,

    // The time when the crash occurred in UTC.
    "stacktrace_timestamp_utc": string,

    // The time when the crash occurred (seconds since epoch only).
    "stacktrace_timestamp_epoch_seconds": integer,

    // The time when the crash occurred (partial seconds since epoch only).
    // This property is meant to be paired with "stacktrace_timestamp_epoch_seconds".
    "stacktrace_timestamp_epoch_milliseconds": integer
}
```

## Log Categories and Levels

In iRODS 4.3.0, log output is managed via **server_config.json**. The server now provides several categories in regards to log output. This gives administrators the ability to choose different log levels for various components in an iRODS server. To do this, open **server_config.json** and find the **log_level** property. It should have the following structure:
```json
{
    // ... Other configuration properties ...

    "log_level": {
        // Controls log output related to generic Agent operations.
        "agent": "info",

        // Controls log output related to the Agent Factory process.
        "agent_factory": "info",

        // Controls log output related to API operations.
        "api": "info",

        // Controls log output related to authentication operations.
        "authentication": "info",

        // Controls log output related to database operations.
        "database": "info",

        // Controls log output related to the Delay Server process.
        "delay_server": "info",

        // Controls log output recorded via the rodsLog API.
        "legacy": "info",

        // Controls log output related to microservice operations.
        "microservice": "info",

        // Controls log output related to networking operations.
        "network": "info",

        // Controls log output related to resource operations.
        "resource": "info",

        // Controls log output related to rule engine plugin operations.
        "rule_engine": "info",

        // Controls log output related to the iRODS process overseeing all other
        // iRODS processes.
        "server": "info"
    },

    // ... Other configuration properties ...
}
```

Each category can be set to one of the following values (ordered from noisiest to quietest):

- trace
- debug
- info
- warn
- error
- critical

While the logging infrastructure has been modernized, use of it has not yet been applied throughout the iRODS server. This means that the **legacy** log category will be the primary setting for controlling log output (**rodsLog** API output is controlled by the **legacy** log category). This situation will improve with future releases.

The following table defines how **rodsLog** API log levels are mapped to the new logging API's log levels.

| rodsLog API Log Level   | New Logging API Log Level |
| ----------------------- | ------------------------- |
| LOG_DEBUG10             | trace                     |
| LOG_DEBUG9              | trace                     |
| LOG_DEBUG8              | trace                     |
| LOG_DEBUG7              | debug                     |
| LOG_DEBUG6              | debug                     |
| LOG_NOTICE              | info                      |
| LOG_WARNING             | warn                      |
| LOG_ERROR               | error                     |
| LOG_SYS_WARNING         | critical                  |
| LOG_SYS_FATAL           | critical                  |

To demonstrate how the two APIs overlap, let's say a message is logged at `LOG_DEBUG7` via the **rodsLog** API. That message will only be recorded in the log file if the **legacy** log category is set to `trace` or `debug`.

And because of this overlap, the next section still applies.

## Logging and the rodsLog API

The following environment variables can be set in the service account and require a server restart to take effect (`./irodsctl restart`):

- `spLogLevel=N` - This will send all log messages of `N` or more severe (`1`, or `LOG_SYS_FATAL` is most severe) to the log file. Increasing the log level will increase the number of messages sent to the log file. Setting `spLogLevel` to `8` or more will show the wireline XML packing instructions. This can also be set in the service account's `irods_environment.json` file as `irods_log_level` (and not require a server restart, as each rodsAgent reads this environment file on standup, per incoming iRODS connection).

 | Verbosity       | spLogLevel   |
 | --------------- | ------------ |
 | LOG_DEBUG10     | 10           |
 | LOG_DEBUG9      |  9           |
 | LOG_DEBUG8      |  8 (XML)     |
 | LOG_DEBUG       |  7           |
 | LOG_DEBUG6      |  6           |
 | LOG_NOTICE      |  5 (default) |
 | LOG_WARNING     |  4           |
 | LOG_ERROR       |  3           |
 | LOG_SYS_WARNING |  2           |
 | LOG_SYS_FATAL   |  1           |

- `spLogSql=1` - This will send the bind variables, the SQL query, and the return values going to and coming from the database to the log file. This needs to be set on the catalog service provider server. Setting this on a catalog service consumer will have no effect.

Additionally, client side environment variables will affect all new connections without a server restart:

- `irodsProt=1` - This client side environment variable will request the client and server both use the iRODS XML protocol for the entire connection.  Note that the initial iRODS handshake is always XML, so even without `irodsProt` set, a high `spLogLevel` setting on the server will cause some messages to be displayed.

## iRODS Server is down

!!! error
    USER_SOCK_CONNECT_TIMEDOUT -347000

!!! error
    CROSS_ZONE_SOCK_CONNECT_ERR -92110

!!! error
    USER_SOCK_CONNECT_ERR -305000

Common areas to check for this error include:

**ienv**

- The ienv command displays the iRODS environment in use.  This may help debug the obvious error of trying to connect to the wrong machine or to the wrong Zone name.

**Networking issues**

- Verify that a firewall is not blocking the connection on the iRODS ports in use (default 1247 and 1248) (or the higher ports for parallel transfer).

- Check for network connectivity problems by pinging the server in question.

**iRODS server logs**

If the iRODS environment issues and networking issues have been ruled out, the iRODS server/client logs may provide additional information with regards to the specifics of the error in question.

## Routing issue and/or an accidental use of localhost

!!! error
    SYS_EXCEED_CONNECT_CNT -9000

This error occurs when one of the iRODS servers fails to recognize itself as localhost (and probably the target of the request) and subsequently routes the request to another server (with its hostname).  This usually occurs because of a configuration error in /etc/hosts possibly due to:

1. DHCP lease renewal
2. shortnames used instead of fully qualified domain names
3. a simple typo

Every iRODS server in a Zone needs to be fully routable to and from every other iRODS server in the Zone.

There are two networking requirements for iRODS:

1. Each server in the Zone will be referred to by exactly one hostname, this is the hostname returned by `hostname`.

2. Each server in the Zone must be able to resolve the hostnames of all servers, including itself, to a routable IP address.


!!! error
    USER_RODS_HOSTNAME_ERR -303000

This error could occur if the gethostname() function is not returning the expected value on every machine in the Zone.  The values in the catalog service provider must match the values returned by gethostname() on each machine.

## Incorrect jumbo packet configuration

!!! error
    SYS_COPY_LEN_ERR -27000

This error could occur if the relevant switch (or node) settings are not enabled to allow jumbo packets.  It might appear in the logs with 'Connection reset by peer'.

## Incorrect server authentication

!!! error
    REMOTE_SERVER_AUTHENTICATION_FAILURE -910000

This error occurs when there is a `zone_key` mismatch (in **server_config.json**) between two servers in the same Zone.  The `zone_key` is a shared secret and must be the same on all servers within the same Zone.

## No such file or directory

Common areas to check for this error include:

1. Permissions - Verify that the iRODS user has 'write' access to the directory in question
2. FUSE error
3. Zero byte files


## No rows found in the iRODS Catalog

!!! error
    CAT_NO_ROWS_FOUND -808000

This error occurs when there are no results for the database query that was executed. This usually happens when either:

1. the query itself is not well-formed (e.g. syntax error), or
2. the well-formed query produced no actual results (i.e. there is no data corresponding to the specified criteria).

## Access Control and Permissions

!!! error
    CAT_NO_ACCESS_PERMISSION -818000

This error can occur when an iRODS user tries to access an iRODS Data Object or Collection that belongs to another iRODS user without the owner having granted the appropriate permission (usually simply read or write).

With the more restrictive "StrictACL" policy being turned "on" by default in iRODS 4.0+, this may occur more often than expected with iRODS 3.x.  Check the permissions carefully and use `ils -AL` to help diagnose what permissions *are* set for the Data Objects and Collections of interest.

Modifying the "StrictACL" setting in the iRODS server's `core.re` file will apply the policy permanently; applying the policy via `irule` will have an impact only during the execution of that particular rule.

## Credentials

!!! error
    CAT_INVALID_USER -827000

This error can occur when the iRODS user is unknown or invalid in some way (for instance, no password has been defined for the user, or the user does not exist in that Zone).  This error is most common while debugging configuration issues with Zone Federation.

## Incorrect or Non-Existent Default Resource

!!! error
    USER_NO_RESC_INPUT_ERR -321000

This error can occur when the iRODS server does not have a storage resource by the name specified by the user or, perhaps, incorrectly set by default by the server administrator (probably in core.re).

A client can use `iput -R` or update their `irods_environment.json` value of 'irods_default_resource' to override the default storage resource settings. 

## Rule Engine Plugin Framework Error

!!! error
    RULE_ENGINE_ERROR -1828000

This error can occur when a user sends a rule to the wrong rule engine plugin instance.

In the following case, the Python rule engine plugin is invoked (because it is listed first
in **server_config.json**), tries to interpret the iRODS Rule Language rule text it is given,
and then returns a `SyntaxError` since it is not valid Python:

```
$ irule -vF goodrule.r
rcExecMyRule: goodrule{
 writeLine("serverLog","testing...")
}

outParamDesc:
ERROR: rcExecMyRule error.  status = -1828000 RULE_ENGINE_ERROR
```

```
$ tail -n16 /var/lib/irods/log/rodsLog*
Oct 21 22:40:35 pid:21170 ERROR: caught python exception:   File "<string>", line 1
    goodrule{
            ^
SyntaxError: invalid syntax
Oct 21 22:40:35 pid:21170 ERROR: rsExecMyRule : -1828000, [-]   ../irods_rule_engine_plugin-python.cxx:999:irods::error exec_rule_text(irods::default_re_ctx &, std::string, std::list<boost::any> &, irods::callback) :  status [RULE_ENGINE_ERROR]  errno [] -- message [irods_rule_engine_plugin_python::irods::error exec_rule_text(irods::default_re_ctx &, std::string, std::list<boost::any> &, irods::callback) Caught Python exception.
  File "<string>", line 1
    goodrule{
            ^
SyntaxError: invalid syntax
]

Oct 21 22:40:35 pid:21170 NOTICE: readAndProcClientMsg: received disconnect msg from client
```

## iRODS Rule Language Rule Engine Variables

The iRODS Rule Language rule engine variable scoping rules are summarized as:

  1. All input and output variables have global scope
  2. All local variables have rule scope
  3. Except:
    1. The iterator variables of `foreach()` have the scope of the `foreach()` block
    2. Variables assigned by an assignment in a `let` expression have the scope of the `let` expression
    3. Variables assigned by a pattern match in a match expression have the scope of the corresponding alternative of match expression

## Parallel Transfer Port Contention

The iRODS Server will issue a LOG_NOTICE when it unsuccessfully attempts to claim an available port while setting up a parallel transfer portal.

```
Nov  1 11:20:24 pid:16308 NOTICE: setupSrvPortal: listen failed, errno: 98
Nov  1 11:23:11 pid:4482 NOTICE: setupSrvPortal: listen failed, errno: 98
Nov  1 11:24:58 pid:5328 NOTICE: setupSrvPortal: listen failed, errno: 98
Nov  1 11:27:10 pid:15614 NOTICE: setupSrvPortal: listen failed, errno: 98
Nov  1 11:30:25 pid:11650 NOTICE: setupSrvPortal: listen failed, errno: 98
```

This occurs when the server is hitting resource contention and may indicate that the server needs a larger parallel transfer port range defined in **server_config.json** (the default is 20000-20199).

## Schema Validation Warnings

To enable validation on Ubuntu 12, install the `jsonschema` module via Python's pip:

```
sudo pip install jsonschema
```

## Policy firing on unexpected server

Since iRODS servers make server-to-server connections when necessary, where policy is fired can be surprising within a Zone.  Every server in a Zone defines its own policy, and that policy is not required to be identical across the Zone.  There are valid use cases for some servers to handle things differently than their peers.

When a [server spins up an iRODS Agent to service an initial incoming API request from a client](process_model.md), that server handling the connection is the one where policy will be evaluated and executed.  Particular PEPs will fire on the server that is handling that part of the request.

This means that PEPs related to database connectivity (talking to the catalog) will only fire on the catalog provider, while PEPs related to the spinning disk where a replica is located will fire on the server handling that resource.

So, as an example, the `acTrashPolicy {msiNoTrashCan;}` policy can be set per server.  But whether the policy is in force depends on which server services the incoming connection (and therefore where the irodsAgent is running), not where the resources are hosted.

## Connection refused for data transfer over Federation

If you see something like this...

```sh
$ iput file /remote_zone/home/rods#local_zone/foo
remote addresses: XXX.XX.X.X ERROR: connectToRhostPortal: connectTo Rhost remote_zone.example.org port YYYYY error, status = -305111
remote addresses: XXX.XX.X.X ERROR: putUtil: put error for /remote_zone/home/rods#local_zone/foo, status = -305111 status = -305111 USER_SOCK_CONNECT_ERR, Connection refused
```

...please see [Limitations](federation.md#limitations) in the Federation section of the documentation.

## Dynamic PEP Signature Mismatches

When writing dynamic PEPs, getting the signature wrong will provide a hint in the log file:

```
Nov 12 09:57:30 pid:25245 DEBUG: error: cannot find rule for action "pep_resource_resolve_hierarchy_pre" available: 103.
line 0, col 0
pep_resource_resolve_hierarchy_pre(*ARG0,*ARG1,*ARG2,*ARG3,*ARG4,*ARG5,*ARG6)
^
```

This explains that `pep_resource_resolve_hierarchy_pre` expects exactly seven arguments.  Please check the [Available Dynamic PEP](../plugins/dynamic_policy_enforcement_points.md#available-dynamic-peps) tables to see what those arguments should be for the PEP you are trying to implement.

## No SSL for 4.x to 3.x Federation

!!! error
    ADVANCED_NEGOTIATION_NOT_SUPPORTED -1815000

Please see [No SSL for 4-to-3 Federation](ssl.md#no-ssl-for-4-to-3-federation)

## Using 3.x iCommands with a 4.0+ iRODS Server

3.x iCommands retain basic functionality when speaking with a 4.0+ iRODS Server.

However, operations much more complicated than simple puts and gets are likely to hit cases where the 3.x iCommands do not have sufficient information to continue or they do not recognize the results returned by the Server.

This is largely due to the SSL handshaking and resource hierarchies in 4.0+.

It is recommended to use the supported iCommands from 4.0+.

## Connecting to Oracle via ODBC on CentOS

There is a [documented problem with Oracle's ODBC driver on CentOS](https://bugzilla.redhat.com/show_bug.cgi?id=498311).

As a workaround, a symlink must be created on the Catalog Service Provider:

```
sudo ln -s /usr/lib64/libodbcinst.so.2 /usr/lib64/libodbcinst.so.1
```

## Database schema upgrade needs more memory

When upgrading to 4.2+ (schema 5) on MySQL, the default memory allocation of 8MB may not be sufficient to handle the conversion in R_DATA_MAIN from resource hierarchies as strings to resource id as integers.

!!! ERROR
    ERROR 1206 (HY000): The total number of locks exceeds the lock table size

On a database with 11 million rows in R_DATA_MAIN, the following setting allowed for the upgrade to complete.

Increase the memory allocated to a sufficient size in `/etc/my.cnf`:

```
innodb_buffer_pool_size=100M
```

## Variable out of scope in a remote block

When executing a microservice on a remote server, the output variable cannot be used in the original rule unless it has been previously declared.

!!! ERROR
    SYS_PACK_INSTRUCT_FORMAT_ERR -15000

This is a variable scoping problem.  The remote block has its own scope, and any output variables used for the first time within the remote block will not be available to the outer block.

The following empty string declaration will ensure `*out` is available for use by `OUTPUT`:

```
testRule {
    *out = "";
    remote("otherserver.example.org", "") {
        test_microservice(*out);
    }
}
INPUT null
OUTPUT *out
```

## Rebalance already running

!!! error
    REBALANCE_ALREADY_ACTIVE_ON_RESOURCE -1829000

This error is returned when a rebalance operation is attempted on a coordinating resource that is already running a rebalance operation.

```
A rebalance_operation on resource [resource_name] is still active (or stale) [hostname:pid] [timestamp]
```

If a rebalance is confirmed to be stale, remove the metadata from the resource:

```
imeta rm -R <resource_name> rebalance_operation <hostname:pid> <timestamp>
```

## Overwriting to a down resource

!!! error
    HIERARCHY_ERROR -1803000

This error can occur when the connected server is determining where to route an incoming read or write. In the case of an `iput -f` where the data object has only a single replica on a resource that is currently marked down, the list of available locations to route is empty.

This error can also occur if the client is asking to connect to a malformed location or resource hierarchy.

## Orphan file behavior

The orphan file behavior of iRODS occurs during two differing scenarios:

1. An attempt to delete a file from a resource that is not currently available
2. An attempt to write a file where there is already a file in the Vault (on disk)

The following cases document the current behavior:

1. Removing a Data Object, without the force flag, consisting of a single replica, on a resource currently marked down:

    - results in a `HIERARCHY_ERROR`
    - no catalog movement
    - no data movement on disk

2. Removing a Data Object, with the force flag, consisting of a single replica, on a resource currently marked down:

    - results in a `SYS_RESC_IS_DOWN` error
    - the Data Object is moved to `/[myZone]/trash/orphan/[myUser]#[myZone]/[filename].[10digitrandomsuffix]`
    - no data movement on disk

3. Removing a Data Object, without the force flag, consisting of one replica on a resource marked up, and the other on a resource marked down:

    - results in a `SYS_RESC_IS_DOWN` error
    - catalog is updated, Data Object now in the trash, as expected
    - replica on up resource, phypath updated, file moved on disk accordingly
    - replica on down resource, phypath untouched, file on disk untouched

4. Removing a Data Object, with the force flag, consisting of one replica on a resource marked up, and the other on a resource marked down:

    - results in a `SYS_RESC_IS_DOWN` error
    - catalog is updated, Data Object now at `/[myZone]/trash/orphan/[myUser]#[myZone]/[filename].[10digitrandomsuffix]`
    - replica on up resource is gone
    - replica on down resource, phypath untouched, file on disk untouched

5. Putting a Data Object, with or without the force flag, onto a resource where a file already resides where the new Data Object's replica will reside

    - results in `SUCCESS`
    - new data is where it is expected, catalog and disk
    - old/existing data is not in catalog (as expected), but has been moved aside in the vault area to `[physicalvaultpath]/orphan/home/[myUser]/[filename].[10digitrandomsuffix]`

6. Replicating a Data Object onto a resource where a file already resides where the new replica will reside

    - results in `SUCCESS`
    - new replica data is where it is expected, catalog and disk
    - old/existing data is not in catalog (as expected), but has been moved aside in the vault area to `[physicalvaultpath]/orphan/home/[myUser]/[filename].[10digitrandomsuffix]`

Turning off the trash can in iRODS has no effect on any of the above behavior in 4.1 or 4.2:
```
acTrashPolicy {msiNoTrashCan; }
```

## Misconfigured monitoring

This error is logged when the initial StartupPack of the iRODS protocol from the client is malformed.

```sh
ERROR: readWorkerTask - readStartupPack failed. -4000
```

This usually occurs when a monitoring system has been configured but [is not sending the proper `HEARTBEAT`](./tips_and_tricks.md#monitoring-status-of-irods-servers).

## Rsyslog and Message Truncation

This section applies to iRODS v4.3.0 and later.

Messages in iRODS can be very long. Messages that exceed the maximum message size defined by rsyslog will be truncated. To avoid this situation, rsyslog provides the following option:

```
$MaxMessageSize <size>
```

This is a global configuration option that affects all applications communicating with rsyslog. The default message size is 8k. See [rsyslog documentation](https://www.rsyslog.com/doc/v8-stable/configuration/global/index.html) for more information about this option.

## Data object stuck in locked (or intermediate) status

When an agent exits without successfully updating the replica statuses of an open data object, the replicas will be stuck in a locked status. This can happen in 1 of 2 known ways:

1. Agent crashes due to an uncaught exception or other serious failure which did not allow the agent to clean up after itself.
2. An open replica has been closed, but finalizing the data object fails due to an error updating the catalog.

Both of these cases are serious issues. If either of these are the case you are seeing, [file an issue on Github](https://github.com/irods/irods/issues).

In this situation, replicas can be modified to bring them out of the locked or intermediate status using `iadmin modrepl`. This tool allows a rodsadmin to modify a column for a particular replica of a particular data object. Simply run the following for each out-of-sorts replica:
```
$ iadmin modrepl logical_path <full_path_to_data_object> replica_number <replica_number_to_modify> DATA_REPL_STATUS <desired_replica_status>
```
For more details, see [iadmin documentation](../icommands/administrator/#modrepl).

The `desired_replica_status` in the usage snippet above is left to the reader to determine, but here are the definitions for the values of `DATA_REPL_STATUS` recognized by iRODS systems:

0. **stale**: The replica is no longer known to be good. Usually indicates that a more recently written-to replica exists. This does not necessarily mean that the data is incorrect but merely that it may not reflect the \"truth\" of this data object.
1. **good**: The replica is good. Usually the latest replica to be written. When a replica is good, all bytes and system metadata (size and checksum, if present) are understood to have been recorded correctly.
2. **intermediate**: The replica is actively being written to. The state of the replica cannot be determined because the client writing to it still has the replica opened. Replicas which are intermediate cannot be opened for read or write, nor can they be unlinked or renamed.
3. **write-locked**: One of this replica's sibling replicas is actively being written to but is itself at rest. Replicas which are write-locked cannot be opened for read or write nor can they be unlinked or renamed.

This technique should only be used to dig out of this serious situation or for testing purposes.

## Firewalls dropping long-idle connections during parallel transfer

iRODS 4.2 and prior have three different ways to move data: single-buffer, streaming, and parallel transfer.  Both single-buffer and streaming transfers are on port 1247 and never have an idle connection.  The parallel transfer method connects on port 1247, constructs a portal of high ports, and then moves data over those high ports.  Once the transfer is complete, control returns to port 1247 for the connection cleanup and bookkeeping.  If the transfer takes long enough, local network firewall timeouts may be tripped and the initial port 1247 connection may be severed.

To prevent the connection on port 1247 from being severed, the administrator can lower the `tcp_keepalive_time` kernel setting to make sure the connection remains active.  Note that this setting will affect every TCP connection on that machine.

Reference: [https://tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/#usingkeepalive](https://tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/#usingkeepalive)

Initially, the default is probably set to 7200 seconds (2 hours):
```
root# cat /proc/sys/net/ipv4/tcp_keepalive_time
7200
```

Drop it to 10 minutes (something less than the firewall timeout):
```
root# echo 600 > /proc/sys/net/ipv4/tcp_keepalive_time
root# cat /proc/sys/net/ipv4/tcp_keepalive_time
600
```

To make the setting rebootable... add this line to `/etc/sysctl.d/99-sysctl.conf` (or similar location):
```
net.ipv4.tcp_keepalive_time = 600
```
