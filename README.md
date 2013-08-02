dcarsync
========

Datacenter-aware rsync wrapper

This project aims to provide a wrapper around rsync+ssh to provide flexible fan-out for content distribution.  This is often needed in multi-datacenter installations where there are incentives to minimize upstream bandwidth usage, parallelize rsync cost, etc...

Assumptions and requirements:
-----------------------------
* There's 1 "source server" that contains a dataset to be distributed to several servers
* The source server can successfully push the dataset using rsync, over ssh, passwordless, to *all* recipient servers
* ssh agent forwarding is not disabled

Intended behavior:
------------------
 * dcarsync will parse its input and internally build a directed acyclic graph.  Each graph node will be a single server or a "cluster" of servers
 * dcarsync will go about satisfying the DAG from the top, using either rsync(->ssh), or ssh->(rsync->ssh)
 * A graph node which turns green will satisfy its children in (a possibly configurable) level of parallelism

Request for Comments:
---------------------
The project is currently in a phase of evaluating the invocation grammar.   All future users interested are encouraged to comment on the proposed syntax at:
https://github.com/minaguib/dcarsync/issues/1

Proposed invocation syntax examples:
------------------------------------

The source server containing the content to be distributed is referenced as "."

To 1 other server:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ .:host11

To 1 other server with ssh credentials:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ .:deploy@host11

To 1 other server, with rsync options given

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ .[-avz]:host11

To 2 other servers, in parallel:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ .:host11 .:host12

To 2 other servers, in a serial chain (shows dependency resolution):

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ .:host11 host11:host12

To 3 other servers, in a serial chain:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ \
      .:host11 \
      host11:host12 \
      host12:host13

To 3 other servers, in a serial chain, same as above but more compact::

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ .:host11:host12:host13

To a cluster of 6 servers in a DC, with host11 explicitly named as intermediary:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ \
      .:host11 \
      host11:host12 \
      host11:host13 \
      host11:host14 \
      host11:host15 \
      host11:host16

To a cluster of 6 servers in a DC, using logical group with an unnamed intermediary:
This is similar to the above, but using an unnamed intermediary offers retry thereby eliminating host11 as a SPOF:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ '.:(host11,host12,host13,host14,host15,host16)'

To a cluster of 6 servers in a DC, with initial rsync options given:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ '.[-avz]:(host11,host12,host13,host14,host15,host16)'

To a cluster of 6 servers in a DC, with initial rsync options given, and secondary (intra-DC) rsync options given:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ '.[-avz]:[-av](host11,host12,host13,host14,host15,host16)'

To 3 distinct DCs:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ \
      '.[-avz]:[-av](host11,host12,host13,host14,host15,host16)' \
      '.[-avz]:[-av](host21,host22,host23)' \
      '.[-avz]:[-av](host31,host32,host33,host34,host35,host36)'

To 3 distinct DCs, but . has limited upstream so use host11 as an intermediary:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ \
      .[-avz]:host11 \
      'host11[-av]:[-av](host12,host13,host14,host15,host16)' \
      'host11[-avz]:[-av](host21,host22,host23)' \
      'host11[-avz]:[-av](host31,host32,host33,host34,host35,host36)'

To 3 distinct DCs, but . has limited upstream so use 1 unnamed intermediary:

    dcarsync /usr/local/me/dataset/ /usr/local/remote/dataset/ \
    '.:(host11,host12,host13,host14,host15,host16):(host21,host22,host23):(host31,host32,host33,host34,host35,host36)'


