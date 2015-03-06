# Better LRP Deduping

## The Problem

Our performance tests (chronicled [here](https://docs.google.com/a/pivotal.io/document/d/10JUzSRMM_3oh5JavLEErwz2Z4bmwOqXmoeycPcyMgRg/edit#heading=h.52e2udfgnzho) - earch for the "Disappearing Cells" heading for the section relevant to this discussion) have revealed some interesting behavior.

We are seeing Cells spuriously disappear.  We have a story to investigate the root cause [here](https://www.pivotaltracker.com/story/show/89826352): either etcd is starting to show signs of strain, or there is a source of slowness in the Rep.  Once we understand the root cause we'll investigate solutions to fix this issue.

However, thanks to this issue we now have some insight into how Diego behaves when a Cell disappears then quickly reappears:

- When the Cell disappears the Converger notices immediately (thanks to an etcd watch) and runs a convergence loop.
- This results in the ActualLRPs formerly on the Cell being auctioned off.
- When the Auction actually runs we typically see that the Cell has returned (!).  The Auctioneer proceeds to distribute ActualLRPs across the cluster, typically preferring not to place them on the original Cell (as it has an ActualLRP with the same ProcessGuid on it).

We are now in a situation where we have duplicate ActualLRPs running on the cluster. What happens next?   We have a race between the Rep of the “disappeared” cell (let’s call it Disappeared-Cell) and the Rep of the Cell to which the LRP was allocated (let’s call it Allocated-Cell).  There are two scenarios:

1. Disappeared-Cell’s bulk loop runs *before* the LRP starts running on Allocated-Cell.  This will cause Disappeared-Cell to override the UNCLAIMED/CLAIMED LRP in the BBS and mark it as running on Disappeared-Cell.  When Allocated-Cell finally starts running the LRP it will attempt to transition it to RUNNING in the BBS and fail.  (We see evidence in the logs of this happening).
2. Disappeared-Cell’s bulk loop runs *after* the LRP starts on Allocated-Cell.  At this point the LRP is said to be running on Allocated-Cell and Disappeared-Cell will tear its copy of the LRP down.

If scenario 1 occurs, the distribution of LRPs across the cluster does not change.  If scenario 2 occurs, however, we end up with an imbalance: Disappeared-Cell is now substantially emptier than when it started and Allocated-Cell has to bear the burden.

Note that the actual behavior is a bit more complex - the ActualLRPs typically end up on all the other Cells in the cluster and there are multiple races between the Disappeared-Cell and the various Allocated-Cells.  The end result is that the Disappeared-Cell typically loses ActualLRPs and to some of the Allocated-Cells.  Again: we end up with an imbalanced distribution.

So, **the problem**: when Cells spuriously disappear then reappaer we end up with an imbalanced distribution.

## A Solution

This problem is not too surprising.  Our current deduping logic is very naive.

I propose a straightforward solution: if we add a notion of `Container-Creation-Time` to the ActualLRP (kind of like `Since`, though `Since` is tied to state transitions) then we could have some logic like this:

    - Rep `α` performs its bulk loop and sees an ActualLRP that is `RUNNING-β`.
    - `α` compares the `Container-Creation-Time` of its container to that of `β`.
        - If `α`'s `Container-Creation-Time` > `β`'s `α` performs a CAS: `RUNNING-β->RUNNING-α`
        - Otherwise, `α` deletes the container.

This logic will tends to prefer containers that have been around longer. Since these wil typically correspond to the initial placement performed by the auction we are more likely to preserve good distributions.