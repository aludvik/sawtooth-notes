# Pessimistic Publishing (p2)

Currently, validators decide whether to begin building a new block based solely
on whether or not the block publisher has any pending batches.

The process of creating a new block is expensive because it involves:
- Pulling batches out of the pending batches queue
- Creating a scheduler
- Executing the Scheduler

If the publisher makes it through building a valid block without the chain head
being updated, then the block is submitted to the Completer, which is even more
expensive because the block has to go through validation again and it is
broadcast to the rest of the network, which must consider it. This network wide
effort is sometimes avoided when the consensus module decides to abandon the
block (due to whatever internal rules it has).

Deciding whether to build a new a block based only on whether there are batches
in the pending batches queue is insufficient. Some consideration should also be
given to:

- Does the block have any chance of being committed by other validators?
- How far behind we are on block validation?

In order to give the block publisher insight into these two factors, a new
`PublishingLimiter` class should be created and passed to the BlockPublisher
which encapsulates the logic and references necessary to answer the above
question.

    class PublishingLimiter
      def check_build_candidate_block():
        """Return whether to begin building a candidate block or not."""
        ...

Before beginning to build a new candidate block, the BlockPublisher will call
the `check_build_candidate_block()` method and if it returns false, it will
abort.

The initial implementation of the `check_build_candidate_block()` method should
perform the following checks:

- Are there any batches in my pending batches queue? If no, return False.
- Are there any blocks in my block validation queue? If no, return True.
- Should I try to publish a block even though there are blocks in my block
  validation queue? If yes, return True.
    - Are we being DoS'ed?
    - Are all the blocks from long chains, or is there a high fan-out?
    - Do the blocks make sense from a fast fuzzy fork forecasting perspective?
- (Optional) For M >= 0, if the last M blocks that we have tried to publish
  have been rejected, return False.

Lastly, the polling rate for deciding whether to publish a block should be
decreased from its current value of 10 times per second.
