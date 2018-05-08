# Rolling out Diego

The vision here is simple.  We roll out Diego in a given environment incrementally.  We'll have a couple of versions where Diego is deployed *alongside* the DEAs.  Operators will be told to encourage their developers to try the new backend and send us feedback, etc.  Operators will be able to choose whether opting applications into Diego is under *their* control or under their user's control.

After some period of having both backends deployed, we deploy CF so that Diego is the only show in town.  This deployment could be accompanied by a migration that forces all applications onto Diego, though ideally (to avoid downtime) most applications will be running on Diego by then.

## How do applications opt into Diego?

Our use of environment variables is a source of complexity and is difficult to audit.  

We propose introducing a new application column to the DB called, simply, `diego`.  This would be a boolean settable via the CC API (though see below).  The boolean would be around for the transitional CF releases and will then be removed with Diego is the only backend.

Since Diego can run DEA-staged droplets, modifying the `diego` boolean can automatically move an application from the DEA stack to the Diego stack.  I would propose that modifying the `diego` boolean always sends an event to the chosen backend to run the application and relies on eventual consistency to wind down the applications on the former backend.  To be clear:

`diego => true`: emits a start message to Diego and allows the DEAs to clean up naturally via HM9000.
`diego => false`: emits a start message to the DEAs and allows Diego to clean up naturallly via NSYNC.

This minimizes downtime and complexity.

We also propose removing the configuration flags from CC.  Diego support is turned on in CC with no way to turn it off.

### Who sets the `diego` boolean?

Some operators may want to give their users control over this.  Others may want to do it themselves.  Some may want to have a graduated phase-in plan where they give developers control to begin with but then grab control themselves later on.

To accomplish this I propose we add a bosh-configurable option called `users_can_select_backend`.  If `true` then all users can modify the `diego` boolean.  If `false` then *only* operators can modify the `diego` boolean.  Note that flip-flopping this option does not modify where the applications are running -- only *who* can modify this state.

### How do operators audit the `diego` boolean?

Operators/admins will use the CC API (we may need to add to it?) to list all applications with `diego=true` or `diego=false`

OSS tooling can be built on top of this to slowly bleed load from the DEAs onto Diego.  For example, public clouds might use this tooling to move applications from the DEAs to Diego in a controlled manner.  We could investigate downtime-less approaches to doing this.  Such a tool would not be necessary for the initial deployment of Diego alongside the DEAs.

### How do we get visibility into the chosen backend?

The CLI could be modified such that `cf apps` and `cf app` list the selected backend.  Not sure that this is MVP, however as it would be transitory (when Diego is the only game in town we'd rip this feature out).
