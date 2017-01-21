---
title: A Fair Voting System
---

This is a quick note to describe a voting system that I've toyed around with a
while, and which I have implemented as an option for decision-making among
participants in an aftok. 

When I describe a "fair" system, I have a few specific properties in mind.
These may bear little relationship to fairness as described by Condorcet or
Arrow's Theorem, but they're the properties that I think are important. Many of
these properties are corrective in nature; they are derived from the flaws of
the systems that are in current use, or have been proposed.

Criteria
--------

* The system must admit any number of candidates/options without creating a
  "spoiler" effect (violated by plurality systems.)

* The system must not compel any voter to provide a ranking for a particular
  candidate (supported by range/approval voting, violated by Borda count, some
  variants of RCV.)

* It must be possible for a voter to vote in a way that accurately reflects
  their opinion of each candidate (supported by range, violated by approval and
  random ballot.)

* Every vote must be significant - that is to say, it must be possible (though
  potentially improbable) for the opinion of any voter, even a voter in a tiny
  minorty, to be reflected in the final result. (violated by everything except
  random ballot.)

To achieve this set of criteria, I propose the following system. It derives
significantly from both range voting and the random ballot. The system depends
upon a deterministic, cryptographically secure pseudorandom number generator
and a secure hash algorithm. The system proceeds in three phases:

Voting Procedure
----------------

Prior to the election, a randomly generated salt value must be published to the
electorate. 

For the vote itself, each voter completes a ballot as for [range
voting](https://en.wikipedia.org/wiki/Range_voting), having the option to
select a rating between 0 and 10 for each candidate. In addition, each ballot
provides a "disapprove" option for rating, which permits the user to state to
what degree they disapprove of all available candidates. "Disapprove" is, for
the remainder of the process, treated as any other candidate. If, upon
completion of the process, the "disapprove" candidate is selected, a new
election must be held, with all current candidates from the failed election
barred from running.

Prior to counting, the ratings selected by each voter for each candidate are
normalized based by dividing by the sum of that voter's ratings. This ensures
that each voter's aggregate contribution to the overall result is the same.

Ballots are counted as with range voting - the voters' ratings for each
candidate are summed to produce an integer-valued tally for each candidate.
The resulting totals are published. Along with these results, a timestamp
specifying some time in the near future (for example, a day from the time of
publishing) will also be published.

At the moment of the specified timestamp, a second salt value is derived from
some publicly visible, time-dependent but non-manipulable value (for example,
the hash of the most current block in the Bitcoin blockchain) will be recorded.

Starting with the candidate with the highest vote count, and proceeding to the
candidate to the lowest vote count, the hexidecimal string values of each
candidates' vote counts are concatenated. To the end of this string, two
additional values are concatenated: the salt value published prior to the
election, and the salt taken subsequent to the election.

The resulting string is used as the seed for the deterministic, secure random
number generator with uniform distribution. This generator is then used to
select a random value from the unit interval.

The published vote counts are then used to allocate break points in the unit
interval, from highest count to lowest count. Essentially, each candidate, from
highest to lowest, is allocated a percentage of the unit interval corresponding
to their rating sum. The winning candidate is the candidate within whose
portion of the unit interval the randomly selected value falls.

Justification
-------------

The system described here has several features not present in other voting
systems.  From its weighted pseudorandom selection, it derives a degree of
fairness not present in other systems - every candidate has the potential to
win, but a highly approved-of candidate will be selected with high probability.
The system is also, by virtue of this randomness, virtually immune to
manipulation by dishonest actors in the vote-counting process - any effort to
influence or subvert the vote-counting process is likely to be wasted.

Secondly, the value of tactical voting is minimized. If a voter approves of
several candidates, but disapproves of others, no significant advantage would
be gained by limiting their allocation of votes to any candidate whom they
approve of.

Due to the presence of a "disapprove" option, it is possible that no candidate
from a given round may win. This promotes a desirable features: candidates must
attempt to have broad appeal to have a reasonable shot at winning, but this
need to seek broad appeal is in tension with the possibility that the election
may need to be re-run. In our modern world of outsize campaign spending,
political parties take a risk by committing significant funds to the support of
a single candidate. In addition, this feature would hopefully limit the
influence of individual personalities in the election process - however,
whether this effect would materialize in practice is something that should be
empirically determined.

Objections
----------

One obvious objection that there is no guarantee that the most highly-approved
candidate be selected. This is in fact considered a feature by the author of
this system - in a closely contested race, there is little reason to believe
the public good is best served by a candidate who wins by a narrow margin.
From a utilitarian perspective, if an electorate is split nearly 50/50, there
is little reason to believe that one of the candidates will better serve the
good of the entire electorate better than the other.

A second objection is that, even though the probability is low, it is possible
that a despicable candidate might be selected. This complaint rests upon the
assumption that the supporters of such a widely despised candidate are not
worthy of fair representation. If we as a society are genuinely commited to the
equality of all people, then the opinions of all, however despicable, must have
at least some minimal chance of being represented.
