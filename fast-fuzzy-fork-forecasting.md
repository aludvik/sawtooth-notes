# Fast Fuzzy Fork Forecasting (f4)

- Add a stage to block pipeline which gates blocks from further consideration
  until some fuzzy check determines they have a chance of being committed.
- Need to purge this stage, or memory leaks
- How will it interact with the BlockManager


[4:28]
you might have more than one fork, so from a fast fork validity check
perspective, you can sort the candidate forks into ‘most credible’ order and
perform the expensive work of validating the most credible one first

[4:28]
but if it has more than zero credibility, then you should not be publishing
blocks

amundson
[4:29 PM]
you could use the variety of publishing nodes in the fork to determine whether
it's real and not a DoS

mitchell [4:29 PM]
credibility means you have consensus payloads that are not bogus

[4:29]
right, that’s a good one

[4:29]
network population

[4:29]
essentially an aggregate metric for credibility

amundson [4:29 PM]
can we call this the reverse-z-test?

[4:33]
if forks are sorted in credibility order, then my assumption would be that the
loser forks would fast fail on consensus comparison once the most credible fork
is fully validated and applied

[4:33]
which has the nice side effect of never processing their garbage transactions

[4:36]
and the publisher would also refrain from publishing a block until there were
no remaining credible forks pending in the chain controller (or whatever it’s
called)
