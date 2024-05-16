KIP-1023: Follower fetch from tiered offset
================

Created by [Abhijeet
Kumar](https://cwiki.apache.org/confluence/display/~abhijeetkumar), last
modified on [Apr 29,
2024](https://cwiki.apache.org/confluence/pages/diffpagesbyversion.action?pageId=293047026&selectedPageVersions=22&selectedPageVersions=23 "Show changes")

- [Status](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-Status)

- [Motivation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-Motivation)

- [Public
  Interfaces](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-PublicInterfaces)

  - [Broker Configuration
    options](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-BrokerConfigurationoptions)

  - [List Offsets
    API](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-ListOffsetsAPI)

- [Proposed
  Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-ProposedChanges)

  - [High-Level
    Design](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-High-LevelDesign)

  - [FetchOffet API
    Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-FetchOffetAPIChanges)

  - [ReplicaFetcher
    Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-ReplicaFetcherChanges)

  - [Follower Fetch
    Scenarios](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-FollowerFetchScenarios)

- [Upgrade](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-Upgrade)

- [Test
  Plan](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-TestPlan)

- [Rejected
  Alternatives](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1023%3A+Follower+fetch+from+tiered+offset#KIP1023:Followerfetchfromtieredoffset-RejectedAlternatives)

# Status

**Current state**: Accepted

**Discussion thread**:
[*here*](https://lists.apache.org/thread/ypwqnwyn7zn6xo781yjq4v99p44gv62h)*  
*

## **JIRA**: \*[![](https://issues.apache.org/jira/secure/viewavatar?size=xsmall&avatarId=21140&avatarType=issuetype)KAFKA-15433](https://issues.apache.org/jira/browse/KAFKA-15433)

Follower fetch from tiered offset <u>Open</u>  
\*

Please keep the discussion on the mailing list rather than commenting on
the wiki (wiki discussions get unwieldy fast).

# Motivation

As per the Tiered Storage feature introduced in
[KIP-405](https://cwiki.apache.org/confluence/display/KAFKA/KIP-405%3A+Kafka+Tiered+Storage),
the follower fetch protocol was modified for topics enabled with tiered
storage. For such topics, the empty follower finds the offset and leader
epoch, up to which the auxiliary state (i.e. leader epoch sequence,
producer snapshot state) needs to be built from the leader. The follower
then starts fetching the data from the leader starting from that offset.
In the KIP, it was decided to use the local-log-start-offset as the
offset for this purpose. Instead of using the local-log-start-offset as
the starting offset from where the follower starts replicating the data
from the leader, we could also use the last-tiered-offset as the
starting offset (technically we will use the next offset for
last-tiered-offset).

This offset is the last offset that has been uploaded to the remote
storage. With this option, the follower will only need to replicate a
small number of segments from the leader. This is because in most cases
where there is no lag, most of the inactive log segments are already
available on the remote storage. If this follower does become a leader
in the future, it can still serve older segments because they are
already on remote storage. Because the follower needs to copy only a
small number of segments from the leader, it can quickly catch up with
the leader and join the ISR list for the topic partition. In practice,
the amount of data the follower needs to replicate from the leader can
be as low as 10-15%. Therefore the time for the new follower to join the
ISR list is also 10-15% compared to before. What this implies is that
with this strategy an empty broker can quickly join the ISR list for the
topic partitions that are assigned to it.

A drawback of using the last-tiered-offset is that this new follower
would possess only a limited number of locally stored segments. Should
it ascend to the role of leader, there is a risk of needing to fetch
these segments from the remote storage, potentially impacting broker
performance. There can be several ways to prevent such followers from
transitioning to a leader role, such as having strict leader election
criteria, wherein followers with limited locally stored segments are not
allowed to become a leader, or having slightly relaxed criteria wherein
other followers are preferred for leadership over this follower. A
similar problem can also happen on clusters that are enabled with fetch
from the closest replica
([KIP-392](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica)).
For such clusters, the follower that was bootstrapped using the tiered
offset may need to serve fetch requests. To prevent the degradation of
the follower, we can build a similar mechanism to exclude/deprioritize
them for consumption requests. Incorporating these preventing measures
alongside the proposed utilization of last-tiered-offset can effectively
balance the advantage of rapid follower synchronization and mitigate
potential performance impacts on the broker, thereby optimizing the
overall efficiency and reliability.

In this KIP, we will focus only on the follower fetch protocol using the
last-tiered-offset, which will be helpful in quickly converting empty
followers to in-sync followers. This can drastically reduce the time
taken to run cluster rebalances. Similarly, it can drastically shorten
the time to replace a faulty broker.

# Public Interfaces

## *Broker Configuration options*

A new broker property will be added to enable/disable the feature of
using last-tiered-offset in the follower fetch.

follower.fetch.last.tiered.offset.enable

- Type: Boolean

- Mode: Dynamically configurable as cluster-default for all brokers in
  the cluster.

- Description: Whether the last tiered offset should be used as the
  start offset for bootstrapping an empty follower

- Default value: False

## *List Offsets API*

ListOffsets API gives the offset(s) for the given timestamp either by
looking into the local log or remote log time indexes.

If the target timestamp is

ListOffsetRequest.EARLIEST_TIMESTAMP (value as -2) returns
logStartOffset of the log.

ListOffsetRequest.LATEST_TIMESTAMP(value as -1) returns
log-stable-offset or log-end-offset based on the isolation level in the
request.

ListOffsetRequest.MAX_TIMESTAMP (value as -3) returns the offset
corresponding to the record with the highest timestamp on the partition.

ListOffsetRequest.EARLIEST_LOCAL_TIMESTAMP (value as -4) returns the
earliest offset stored in the local log.

ListOffsetRequest.LAST_TIERED_TIMESTAMP (value as -5) returns the latest
tiered offset.

This API will be enhanced with supporting new target timestamp with a
value of -6 which is called EARLIEST_PENDING_UPLOAD_OFFSET_TIMESTAMP.
There will not be any new fields added in the request and response
schemas but there will be a version bump to indicate the version update.
This offset represents that offset on the leader’s local log storage
that is the next one to be uploaded to the remote storage (all the
previous offsets have already been tiered).

# Proposed Changes

## High-Level Design

The following diagram provides a visual representation of the leader
topic partition’s log offsets and will help understand the new follower
fetch protocol.

<figure>
<img
src="https://cwiki.apache.org/confluence/download/attachments/97554472/Screenshot%202019-10-25%20at%207.14.08%20PM.png?version=1&amp;modificationDate=1572011075000&amp;api=v2"
alt="Screenshot 2019-10-25 at 7.14.08 PM.png" />
<figcaption aria-hidden="true">Screenshot 2019-10-25 at 7.14.08
PM.png</figcaption>
</figure>

*Lx  = Local log start offset           L<sub>z</sub>  = Local log end
offset            L<sub>y</sub>  = Last stable offset(LSO)*

*R<sub>y</sub>  = Remote log end offset / Last Tiered Offset     
R<sub>x</sub>  = Remote log start offset*

*`Lz >= Ly >= L`x`and Ly >= Ry >= Rx`*

Let us quickly recap the current follower fetch protocol. The follower
does the following:

- It requests data from the leader, beginning from its Fetch-Offset 

  - If the follower is empty, the Fetch-Offset is zero.

- The requested offset may or may not be a valid offset on the leader

  - Case 1: The requested offset is present on the leader

    - The leader responds with the data to the follower

    - Follower receives the data, appends records to the local disk,
      updates its Log-End-Offset and its Fetch-Offset

    - Follower requests data beginning from the new fetch offset

                     This case is also true when the leader has uploaded
the segments, but hasn’t cleared the segments from the local disk yet.
In such a case, the requested offset (offset 0) may still be available
on the leader and the leader will respond with the data to the follower.

- <div>

  - Case 2: The requested offset is not present on the leader. There are
    two possible scenarios here:

    - Case 2.1: The requested offset is lower than the earliest offset
      the leader knows

      - Leader response with OFFSET_OUT_OF_RANGE error

      - The follower fetches the earliest offset on the leader
        (Log-Start-Offset)

      - Follower updates its offsets

        - Log-Start-Offset → leader’s Log-Start-Offset

        - Log-End-Offset → leader’s Log-Start-Offset

        - Fetch-Offset → leader’s Log-Start-Offset

      - The follower requests data from the new fetch offset.

    - Case 2.2: The requested offset is greater than or equal to the
      earliest offset the leader knows, but the offset has moved to
      tiered storage.

      - Leader responds with OFFSET_MOVED_TO_TIERED_STORAGE error (also
        returns its Log-Start-Offset)

      - Follower fetches the leader’s Local-Log-Start-Offset

      - Follower builds the remote log auxiliary states for the leader
        offsets in the range \[Log-Start-Offset, 
        Local-Log-Start-Offset\]

      - Follower updates its offsets

        - Log-Start-Offset → Leader’s Log-Start-Offset

        - Log-End-Offset → Leader’s Local-Log-Start-Offset

        - Fetch-Offset → Leader’s Local-Log-Start-Offset

      - The follower requests data from the new fetch offset.

  </div>

We make the following changes to the follower-fetch protocol when the
requested offset is not present on the leader (Case 2). These changes
apply only when the follower is empty.

- Leader responds with either OFFSET_OUT_OF_RANGE or
  OFFSET_MOVED_TO_TIERED_STORAGE error

- Follower fetches the earliest offset on the leader (Log-Start-Offset),
  if not available in the leader response

- Follower fetches the earliest offset on the leader that is waiting to
  be uploaded (Earliest-Pending-Upload-Offset). This is the next offset
  after the last-tiered-offset.

- Follower builds the remote log auxiliary states for the leader offsets
  in the range \[Log-Start-Offset, Earliest-Pending-Upload-Offset)

- Follower updates its offsets

  - Log-Start-Offset → Leader’s Log-Start-Offset

  - Log-End-Offset → Earliest-Pending-Upload-Offset

  - Fetch-Offset → Earliest-Pending-Upload-Offset

- The follower requests data from the new fetch offset.

## FetchOffet API Changes

We will add a new API for the follower to be able to fetch the
pending-upload-offset from the leader. The leader already keeps track of
the highest offset that has been uploaded to the remote storage.

- The RLM task on the leader runs at frequent intervals to find new log
  segments that have become eligible for upload.

- When the task uploads a segment, it also updates the highest offset
  which has been uploaded to the remote storage.

- If the leader is newly elected, it may not have this information yet,
  but will learn about the highest offset on remote when the
  corresponding RLM Task runs for the first time.

- The new API returns the offset following the highest offset on the
  remote as the response to fetch the Earliest-Pending-Upload-Offset

## ReplicaFetcher Changes

On receiving an OFFSET_OUT_OF_RANGE or OFFSET_MOVED_TO_TIERED_STORAGE
error, ReplicaFetcher does the following:

- Check if the follower replica is empty and if the feature to use
  last-tiered-offset is enabled. If the check fails, continue with the
  old behaviour.

- Otherwise, do the following:

  - Fetch the leader’s log-start-offset

  - Fetch the leader’s Earliest-Pending-Upload-Offset using the API
    discussed in the previous section

  - If the Earliest-Pending-Upload-Offset is unknown (API returned -1),
    there are two possible cases:

    - No segments for the partition have been uploaded to the remote
      storage yet

      - It can be confirmed by checking if the leader’s Log-Start-Offset
        is the same as the Leader’s Local-Log-Start-Offset.

        - The follower will make another call to the leader to fetch the
          EarliestLocal offset (same as Leader’s Local-Log-Start-Offset)
          to be able to verify this scenario.

      - If confirmed, we use the Local-Log-Start-Offset as the
        Earliest-Pending-Upload-Offset

    - Segments are uploaded to the remote storage but the leader does
      not know yet.

      - In this case, the leader will eventually learn about the
        Earliest-Pending-Upload-Offset.

      - The ReplicaFetcher should throw an exception so that the
        partition can be retried after some time (while the leader
        learns about the offset)

  - The highest offset on the remote storage may be much lower than the
    leader’s Log-Start-Offset. Hence, the Earliest-Pending-Upload-Offset
    returned by the leader may also be less than the leader’s
    Log-Start-Offset. This may happen because of the following
    situation:

    - Tiered Storage was enabled for the topics and log segments were
      uploaded to the remote storage

    - Later, tiered storage was disabled for the topic and more messages
      were produced on the topic

    - Some time has elapsed and now the leader’s Log-Start-Offset is
      much ahead of the highest offset on the remote storage.

    - Now tiered storage is enabled for the topic, but no segments have
      been uploaded yet because no new segments are eligible to be
      uploaded yet 

                      Whenever this happens
(Earliest-Pending-Upload-Offset \< Log-Start-Offset), it simply means
that there are no valid log segments on the remote yet (otherwise, the
leader would return an offset \>= log-start-offset). Hence the
ReplicaFetcher simply truncates its log and starts replicating all data
the leader has locally.

- <div>

  - Otherwise, ReplicaFetcher builds the remote log auxiliary states for
    the offset range \[Log-Start-Offset,
    Earliest-Pending-Upload-Offset\], sets it Log-End-Offset and
    Fetch-Offset to Earliest-Pending-Upload-Offset and continues
    fetching the remaining data from the leader.

  </div>

## Follower Fetch Scenarios

Step 1:

Fetch remote segment info, and rebuild leader epoch sequence.

<table style="width:99%;">
<colgroup>
<col style="width: 15%" />
<col style="width: 57%" />
<col style="width: 12%" />
<col style="width: 12%" />
</colgroup>
<tbody>
<tr class="odd">
<td style="text-align: left;"><p>3: msg 3 LE-1</p>
<p>4: msg 4 LE-1</p>
<p>5: msg 5 LE-2</p>
<p>6: msg 6 LE-2</p>
<p>7: msg 7 LE-3 (HW)</p>
<p>leader_epochs:</p>
<p>LE-0, 0</p>
<p>LE-1, 3</p>
<p>LE-2, 5</p>
<p>LE-3, 7</p></td>
<td style="text-align: left;"><ol type="1">
<li><p>Fetch LE-1, 0</p></li>
<li><p>Receives OMTS</p></li>
<li><p>Receives ELO 3, LE-1</p></li>
<li><p>Fetches EPUO</p></li>
<li><p>Receives EPUO 6, LE-2</p></li>
<li><p>Fetch remote segment info and build local leader epoch sequence
until ELO</p></li>
</ol>
<p>leader_epochs:</p>
<p>LE-0,0</p>
<p>LE-1,3</p>
<p>LE-2,5</p></td>
<td style="text-align: left;"><p>seg-0-2, uuid-1</p>
<p>  log:</p>
<p>  0: msg 0 LE-0</p>
<p>  1: msg 1 LE-0</p>
<p>  2: msg 2 LE-0</p>
<p>  epochs:</p>
<p>  LE-0, 0</p>
<p>seg 3-5, uuid-2</p>
<p>  log:</p>
<p>  3: msg 3 LE-1</p>
<p>  4: msg 4 LE-1</p>
<p>  5: msg 5 LE-2</p>
<p>  epochs:</p>
<p>  LE-0, 0</p>
<p>  LE-1, 3</p>
<p>  LE-2, 5</p></td>
<td style="text-align: left;"><p>seg-0-2, uuid-1</p>
<p>segment epochs:</p>
<p>LE-0, 0</p>
<p>seg-3-5, uuid-2</p>
<p>segment epochs:</p>
<p>LE-1, 3</p>
<p>LE-2, 5</p></td>
</tr>
</tbody>
</table>

Step 2:

Continue fetching from the leader

<table style="width:98%;">
<colgroup>
<col style="width: 25%" />
<col style="width: 28%" />
<col style="width: 21%" />
<col style="width: 21%" />
</colgroup>
<tbody>
<tr class="odd">
<td style="text-align: left;"><p>3: msg 3 LE-1</p>
<p>4: msg 4 LE-1</p>
<p>5: msg 5 LE-2</p>
<p>6: msg 6 LE-2</p>
<p>7: msg 7 LE-3 (HW)</p>
<p>leader_epochs:</p>
<p>LE-0, 0</p>
<p>LE-1, 3</p>
<p>LE-2, 5</p>
<p>LE-3, 7</p></td>
<td style="text-align: left;"><p>Fetch from EPUO to HW</p>
<p>6: msg 6 LE-2</p>
<p>7: msg 7 LE-3 (HW)</p>
<p>leader_epochs:</p>
<p>LE-0,0</p>
<p>LE-1,3</p>
<p>LE-2,5</p>
<p>LE-3,7</p></td>
<td style="text-align: left;"><p>seg-0-2, uuid-1</p>
<p>  log:</p>
<p>  0: msg 0 LE-0</p>
<p>  1: msg 1 LE-0</p>
<p>  2: msg 2 LE-0</p>
<p>  epochs:</p>
<p>  LE-0, 0</p>
<p>seg 3-5, uuid-2</p>
<p>  log:</p>
<p>  3: msg 3 LE-1</p>
<p>  4: msg 4 LE-1</p>
<p>  5: msg 5 LE-2</p>
<p>  epochs:</p>
<p>  LE-0, 0</p>
<p>  LE-1, 3</p>
<p>  LE-2, 5</p></td>
<td style="text-align: left;"><p>seg-0-2, uuid-1</p>
<p>segment epochs:</p>
<p>LE-0, 0</p>
<p>seg-3-5, uuid-2</p>
<p>segment epochs:</p>
<p>LE-1, 3</p>
<p>LE-2, 5</p></td>
</tr>
</tbody>
</table>

# Upgrade

The feature will be guarded by a new metadata version and will not be
enabled by default during the rolling upgrade. Follow the steps
mentioned in [Kafka
upgrade](https://kafka.apache.org/documentation/#upgrade) to reach the
state where all brokers are running on the latest binaries with the new
“inter.broker.protocol” version.

# Test Plan

Unit Tests and Integration Tests for the various follower fetch
scenarios.

# Rejected Alternatives

The empty follower needs the offset (and the corresponding leader epoch)
that is the next in line to be uploaded to the remote storage. As
discussed above, we will introduce a new target timestamp in
ListOffsetsRequest API to fetch this offset and the leader epoch from
the leader.

Another way to get the same offset and leader epoch is to use the
existing target timestamp supported by ListOffsetsRequest API. For
non-compacted tiered-storage enabled topics,
EARLIEST-PENDING-UPLOAD-OFFSET is LAST-TIERED-OFFSET + 1. The follower
could fetch the LAST-TIERED-OFFSET from the leader and add 1 to it to
get the EARLIEST-PENDING-UPLOAD-OFFSET. However, it will also need the
leader epoch for EARLIEST-PENDING-UPLOAD-OFFSET and needs to make
another call to the leader to fetch the corresponding leader epoch. This
alternative was rejected because of the following reasons:

1.  It is more complicated than the proposed approach and requires
    multiple calls to the leader.

2.  In the future, if we support compacted topics on tiered storage, the
    logic of adding 1 to LAST-TIERED-OFFSET may not work because the
    offsets are not guaranteed to be contiguous. The follower protocol
    will need significant changes to handle this. Keeping the logic to
    discover EARLIEST-PENDING-UPLOAD-OFFSET on the leader instead will
    make it simpler to handle compacted topics in the future.
