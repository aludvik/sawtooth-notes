# Fault Damage Control

## Distributed System Resiliency

A distributed, replicated system must be resilient to individual nodes failing
in order to have any value over running a centralized, non-replicated system.
The network should be considered operational if, at a minimum, any single node
can fail without clients losing service (they may have to switch which
replication they are talking to, which is okay).

Currently, Sawtooth's resiliency to single-node faults is very low. A single
bad node causes extra work across the network, which increases the likelihood
that other nodes will fail, ultimately crashing the entire network. Because of
this, the likelihood that a network will fail is high.

Given an N node network, where each node has p probability of success, and the
network is not resilient to faults, the probability of service being
interrupted is:

    P(failure) = 1 - P(success) = 1 - p^N

So for a 10 node network, where each node has a 90% chance of working over a
week long run, the probability of failure is ~65%

If the network is resilient to failure (meaning we allowed some number, F,
nodes to fail without declaring service lost), then the probability of service
being interrupted depends on the random variable, X, the number of nodes that
fail, which has a binomial distribution:

    X~Binomial(F, N, 1-p)
    P(success) = P(X <= F) = sum(choose(n, i) * (1-p)^i * p^(n-i) | i in (0, F))
    P(failure) = 1 - P(success)

So for a 10 node network, where each node has a 90% chance of working over a
week long run, and we allow 3 nodes to fail, the probability of failure is ~7%

## Making Sawtooth more Fault Resilient

Most of the recent efforts to improve the stability have been focused on
improving the probability that a single node is successful. As can be seen
above, focusing on this probability has diminishing returns as the network size
and test run duration increases.

An alternative direction for improving the stability of Sawtooth is to focus on
how the network as a whole reacts to the failure of a given node. This is a
complementary problem. Specifically, this direction asks "How can we make
a validator resilient against bad input from a faulty node?" and does not ask
"How can we make a validator never send bad input to another node?" This
direction has been mostly neglected up to this point.

Fault resiliency can take place at 3 layers:
- the networking layer
- the application layer
- the consensus layer

The networking layer can use rate limiting to improve resiliency against DoS
attacks.

The consensus layer is responsible for resiliency from an algorithmic
perspective.

The application layer (where the Sawtooth platform lives) can also be used to
add resiliency to the platform when there are rules that make sense across all
consensus algorithms.

## One idea for Application Layer Fault resilience

One way fault resilience could be added at the application layer is to track
for every block that is considered, who the peer that created it was and
whether it was ultimately committed. In the event that some peer is submitting
"a lot" of blocks that are ultimately being rejected, it could be inferred that
the validator is likely fault, and either dropped as a peer or ignored for some
cool down period.
