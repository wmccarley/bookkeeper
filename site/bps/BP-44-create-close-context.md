---
title: "BP-44: Enhance Ledger Metadata with Create & Close Context"
issue: https://github.com/apache/bookkeeper/<issue-number>
state: "Under Discussion"
release: ""
---

### Motivation

When running a Bookkeeper cluster with clients that create lots of ledgers of non-uniform size and non-uniform lifespan it is not uncommon to experience three problems:
1. sub-optimal ledger eviction performance and unnecessary re-writing during compaction
2. orphaned ledgers... ledgers where the delete operation never succeeded or was not properly attempted
3. no bookkeeper-specific ways to attribute ledger ownership (custom metadata solution is client-specific)

The first point is beyond the scope of this BP. However it is noted here because this BP contains changes that will facilitate future efforts to introduce more sophisticated algorithms for log creation & compaction.

The second and third points are addressed in this proposal by adding optional ledger context metadata to allow clients to provide information about ledger's intended lifecycle as well as it's ownership. 

### Public Interfaces

- This proposal is to add two new interfaces to `org.apache.bookkeeper.client.api` along with associated builder and impl classes: `LedgerCreateContext` and `LedgerCloseContext` a brief example demonstrating how to use these when opening and closing ledgers follows

		LedgerCreateContext createCtx = LedgerCreateContext.builder()
				.createdBy("Company X", "System y", "service.z", "host1.z.companyx.com")
				.partOfDataSet("persistent://public/default/test_topic") //Doesn't have to be a pulsar topic name, can be any string <=256 bytes
				.expectedMaxOpenDuration(Duration.ofHours(4))   // This application rotates ledgers every four hours
				.expectedMaxEntries(50000)                      // or every 50K entries
				.expectedMaxLength(262144000)                   // or when the aggregate ledger size reaches 250MB
				.build();
      
		WriteHandle wh = bk.newCreateLedgerOp()
				.withCreateContext(createCtx)
				.withDigestType(DigestType.CRC32)
				.withPassword(password)
				.withEnsembleSize(3)
				.withWriteQuorumSize(3)
				.withAckQuorumSize(2)
				.execute()          // execute the creation op
				.get();             // wait for the execution to complete		
		
		...
		
		LedgerCloseContext closeCtx = LedgerCloseContext.builder()
				.becauseInactive()                                               // Application has system policies that can be used to
				.expectReadsUntil(Instant.now().plus(1, ChronoUnit.HOURS))       // calculate when ledgers should become 'stale' (for ex. Pulsar backlog quota)
				.expectDeleteAfter(Instant.now().plus(24, ChronoUnit.HOURS))     // as well as then they should 'expire' (for ex. Pulsar retention policy)
				.build();
				
		wh.closeWithContext(closeCtx);

- In addition this proposal includes adding a `sealTime` property to the `LedgerMetadata` class to compliment the existing `ctime` property

- Introduce a new concept of 'orphaned ledgers.' A ledger is considered orphaned if:
    1. it was closed with `.expectDeleteAfter(delete_instant)` directive, and is still not deleted after `deleteInstant` + <server-side configured grace period>
    2. it was opened with `childOfLedger(...)` directive and the associated parent ledger does not exist

- Add a new CLI command for listing orphaned ledgers

        $ bin/bookkeeper shell listorphaned --printorphanreason
        100234 DELETE_OVERDUE 1615839671                 // ledger 100236 should have been deleted on or before Monday, March 15, 2021 8:21:11 PM
        100235 DELETE_OVERDUE 1615842003                 // ledger 100236 should have been deleted on or before Monday, March 15, 2021 9:00:03 PM
        100236 DELETE_OVERDUE 1615846881                 // ledger 100236 should have been deleted on or before Monday, March 15, 2021 10:21:21 PM
        100456 BROKEN_PARENT_REF 100277                  // ledger 100456 lists ledger 100277 as a parent but ledger 100277 does not exist

- Add a new CLI command for showing ownership & data set information about a given ledger

        $ bin/bookkeeper shell showowner 100456
        {
            "createdBy" : {
                "enterprise" : "Company X",
                "system" : "System y",
                "service" : "service.z",
                "instance" : "host1.z.companyx.com"
            },
            "onBehalfOf" : null,
            "dataSet" : "persistent://public/default/test_topic"
        }

- Add a new CLI command to predict a ledger's lifespan based on create and close context information provided by the client _if exists_

        $ bin/bookkeeper shell predictlifespan 100234
        {
            "createTime" : 1615815339,
            "expectedSealTime" : 1615829739,
            "actualSealTime" : 1615827501,
            "expectedReadUntilTime" : null,
            "expectedDeleteTime" : 1615839671
        }

### Proposed Changes

Two new builder interfaces (documented below) will generate immutable instances of `LedgerCreateContext` and `LedgerCloseContext` that can be attached to the LedgerMetadata at open and close time respectively.

Bookkeeper auditor will have another periodic audit that will flag orphaned ledgers under a separate ORPHANED z-node to make the `listorphaned` CLI call efficient. Frequency of orphaned ledger audit can be controlled by a configuration property that can be dynamically reconfigured.

	public interface LedgerCreateContextBuilder {
		
		/**
		 * <p>Provide contextual information to the bookkeeper service about the entity that created
		 * this ledger.
		 * 
		 * <p>This information may be used by the bookkeeper service to generate reports about
		 * aggregate ledger counts, stale/orphaned ledgers, or to rate usage at an enterprise,
		 * system, service, or instance level.
		 * 
		 * @param enterprise that created this ledger
		 * @param system that created this ledger
		 * @param service within the aforementioned system that created this ledger
		 * @param instance identifier uniquely identifying a specific service instance
		 */
		public LedgerCreateContextBuilder createdBy(String enterprise, String system, String service,
				String instance);
		
		/**
		 * <p>If the system that created this ledger is itself a multi-tenant environment then this
		 * call can be used to identify the 'original' owner for the purposes of reporting.
		 * 
		 * @see LedgerCreateContextBuilder#createdBy(String, String, String, String)
		 */
		public LedgerCreateContextBuilder createdOnBehalfOf(String enterprise, String system,
				String service, String instance);
	
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is part of a named data set.
		 * 
		 * <p><em>Note: data set names do not need to be globally unique however data set names
		 * should be meaningful within a given enterprise + system + service combination</em>
		 * 
		 * @param dataSetIdentifier
		 */
		public LedgerCreateContextBuilder partOfDataSet(String dataSetIdentifier);
		
		/**
		 * <p>Provide a hint to the bookkeeper service that this ledger will be closed on or before
		 * <em>create time + duration</em>
		 * 
		 * <p><em>Note: Bookkeeper <b>WILL NOT</b> automatically close the ledger if this duration is
		 * exceeded. This is simply a hint to the storage service about when the ledger is expected to
		 * close.
		 * 
		 * @param duration the ledger will be open for starting at the create time
		 */
		public LedgerCreateContextBuilder expectedMaxOpenDuration(Duration duration);
		
		/**
		 * <p>Provide a hint to the bookkeeper service about the average entry size and standard
		 * deviation.
		 * 
		 * @param sizeInBytes
		 * @param standardDeviation
		 */
		public LedgerCreateContextBuilder expectedAverageEntrySize(long sizeInBytes,
				double standardDeviation);
		
		/**
		 * <p>Provide a hint to the bookkeeper service of the maximum number of entries that will be
		 * written to the ledger before the client will permanently close the ledger.
		 */
		public LedgerCreateContextBuilder expectedMaxEntries(long entryCount);
		
		/**
		 * <p>Provide a hint to the bookkeeper service of the maximum total length of the ledger
		 * before the client will permanently close the ledger.
		 */
		public LedgerCreateContextBuilder expectedMaxLength(long length);
		
		/**
		 * <p>Provide a hint to the bookkeeper service about the expected rate at which entry writes to
		 * this ledger will occur.
		 * 
		 * <p><em>Note: Bookkeeper <b>WILL NOT</b> rate limit entries based on this setting. This
		 * is simply a hint to the storage service about the frequency of writes</em>
		 */
		public LedgerCreateContextBuilder expectedAverageEntryAddRatePerSecond(long entryCount);
		
		/**
		 * <p>Provide a hint to the bookkeeper service about the maximum rate at which entry writes to
		 * this ledger will occur.
		 * 
		 * <p><em>Note: Bookkeeper <b>WILL NOT</b> rate limit entries based on this setting. This
		 * is simply a hint to the storage service about the frequency of writes</em>
		 */
		public LedgerCreateContextBuilder expectedMaximumEntryAddRatePerSecond(long entryCount);
		
		/**
		 * <p>Provide a hint to the bookkeeper service that this ledger is part of an ordered stream of
		 * ledgers and that this ledger directly follows the supplied ledgerId
		 * 
		 * <p><em>Note: Bookkeeper <b>WILL NOT attempt</b> to verify that the supplied ledger id actually
		 * exists. However, Bookkeeper may generate ledger reports that flag ledgers with broken
		 * associations as potentially stale/orphaned</em>
		 * 
		 * @param precedingLedgerId the id of the ledger that precedes this ledger in the stream
		 */
		public LedgerCreateContextBuilder followsLedger(long precedingLedgerId);
	
		/**
		 * <p>Provide a hint to the bookkeeper service that there is a parent -> child relationship
		 * between this ledger and another ledger.
		 * 
		 * <p><em>Note: Bookkeeper <b>WILL NOT</b> attempt to verify that the supplied ledger id actually
		 * exists. However, Bookkeeper may generate ledger reports that flag ledgers with broken
		 * associations as potentially stale/orphaned</em>
		 * 
		 * @param parentLedgerId the id of the ledger that is the parent of this ledger
		 */
		public LedgerCreateContextBuilder childOfLedger(long parentLedgerId);
		
		public LedgerCreateContext build();
	}

	public interface LedgerCloseContextBuilder {
	
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed because the client
		 * is being shutdown.
		 * 
		 * <p><em>Note: if client shutdown is unexpected or abnormal the client can also call: 
		 * {@link LedgerCloseContextBuilder#closingAbnormally(string)} to provide the bookkeeper
		 * service with the message</em>
		 */
		public LedgerCloseContextBuilder becauseClientShutdown();
		
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed because the client
		 * determined that a maximum ledger length has been reached.
		 */
		public LedgerCloseContextBuilder becauseMaxLengthReached();
		
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed because the client
		 * determined that a maximum entry count has been reached.
		 */
		public LedgerCloseContextBuilder becauseMaxEntryCountReached();
		
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed because the client
		 * determined there is no more data to write.
		 * 
		 * <p><em>Note:<ol><li>If the ledger was opened with the intention of publishing a known
		 * quantity of data and all of the data has been written then this call should be used.</li><li>
		 * If the ledger was opened to publish a stream of data and that stream is being terminated then
		 * this call should be used.</li><li>If the ledger was opened to publish a stream of data and 
		 * that stream is being temporarily closed due to inactivity then
		 * {@link LedgerCloseContextBuilder#becauseInactive()} should be used</li></ol></em>
		 */
		public LedgerCloseContextBuilder becauseNoMoreData();
		
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed because the client
		 * determined the write handle has reached some pre-configured inactivity timeout.
		 */
		public LedgerCloseContextBuilder becauseInactive();
		
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed because the client
		 * determined that the time the ledger has been open has exceeded a client side limit.
		 */
		public LedgerCloseContextBuilder becauseTimeRotation();
		
		/**
		 * <p>Indicate to the bookkeeper service that this ledger is being closed under abnormal
		 * circumstances.
		 * 
		 * @param errorMessage optional string with length <=256 containing a message about the error
		 * that created this abnormal circumstance
		 */
		public LedgerCloseContextBuilder closingAbnormally(Optional<String> errorMessage);
		
		/**
		 * <p>Indicate to the bookkeeper service that after the specified instant the service can
		 * expect the ledger to be deleted.
		 * 
		 * <p><em>Note: the Bookkeeper service <b>WILL NOT</b> automatically delete the ledger when
		 * this instant is reached, this is just a hint to the storage service about expected time
		 * the ledger will be deleted.</em>
		 */
		public LedgerCloseContextBuilder expectDeleteAfter(Instant instant);
		
		/**
		 * <p>Indicate to the bookkeeper service that after the specified instant the service can
		 * expect no more reads to occur.
		 * 
		 * <p><em>Note: the Bookkeeper service will continue to serve reads beyond this instant, this
		 * is just a hint to the storage service about expected read behavior</em>
		 */
		public LedgerCloseContextBuilder expectReadsUntil(Instant instant);
		
		public LedgerCloseContext build();
	}

### Compatibility, Deprecation, and Migration Plan

Entire BP as proposed contains only new features and can be implemented with new classes/interfaces and creating new `METADATA_FORMAT_VERSION_4`

### Test Plan

This proposal should only require new unit tests, other tests must continue pass without any changes.

### Rejected Alternatives

Considered renaming `ctime` ledger metadata field to something more descriptive as part of this proposal but that would introduce backwards compatibility issues.

Considered combining `LedgerCreateContext` methods directly in to existing `CreateBuilder` however it seems like a good idea to separate attributes that the bookkeeper service uses directly (eg. Ack quorum size) from ledger context info like ownership or lifespan hints.

Considered introducing an explicit `expireOn(...)` attribute to ledgers to mitigate orphaned ledgers but it seems like it might be hard to implement the client in practice. For instance a Pulsar broker can calculate expiration of ledger based on backlog and retention policies, but policies can be changed long after the ledger is closed. Soft hints like `.expectDeleteAfter(...)` cause no issues because they don't _force_ bookkeeper to actually delete anything but give the Bookkeeper admin a good idea about what should happen to this ledger.