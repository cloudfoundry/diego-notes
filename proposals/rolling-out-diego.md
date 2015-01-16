# Rolling out Diego

The vision here is simple.  We roll out Diego in PCF (any environment, really) incrementally.  We'll have a couple of PCF versions where Diego is deployed *alongside* the DEAs.  Operators will be told to encourage their developers to try the new backend and send us feedback, etc.

After a handful of point-releases we release a PCF version where Diego is the only show in town.  This release could include a migration that forces all applications onto the Diego stack though, ideally (to avoid downtime) most applications will be running on Diego by then.

## How do developers deploying to CF opt into Diego?

Our use of environment variables is a source of complexity and is difficult to audit.  We propose switching over to using the `stack` property of the application.  Applications on the `diego` stack will both stage and run on Diego.

With this in place we can remove the complexity around supporting mixed-mode operation where Diego supports applications staged on DEAs and vice-versa.

We also propose removing the configuration flags from CC.  Diego suppport is turned on in CC with no way to turn it off.  Are there any concerns with that?

Note that this also ends up forcing a restage to end up on Diego.  Are there any concerns with that?

## How do operators audit/enforce a Diego policy?

Operators will use the CC API (we may need to add to it?) to:

- list applications by stack.
- modify the stack for an application (and trigger a restage?)

OSS tooling can be built on top of this to slowly bleed load from the DEAs onto Diego.  For example, PWS might use this tooling to move applications from the DEAs to Diego in a controlled manner.  We could investigate downtime-less approaches to doing this.  Such a tool would not be necessary for the initial PCF release.