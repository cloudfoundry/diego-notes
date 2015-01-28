## App Lifecycles: Static or Dynamic?

Lifecycles consist of two parts: a trinity of binaries (Builder, Runner, Healthcheck), and a Recipe-Generator interface which can generate a Diego recipe for the Staging Task (Builder) and DesiredLRP (Runner + Healthcheck).  

The binaries can simply be found in the blobstore.  But the Recipe-Generator is invoked directly by the CC(-Bridge).  So how do we identify the presence of a Lifecycle, and then invoke its Recipe-Generator? AKA, how do we load Lifecycles?  We see three possible choices:

**Choice #1:**  A set of Lifecycles is part of the CC(-Bridge) codebase.  The Recipe-Generators are directly compiled into the CC(-Bridge).  This is very simple and reliable.  But it is not very extensible without forking the code.

**Choice #2:** The CC(-Bridge) performs some kind of DLL / shelling-out shenanigans to a local set of Recipe-Generator binaries.  The options here are somewhat language-specific.  How would we load these binaries? Co-located bosh jobs?

**Choice #3:** Each Recipe-Generator runs as a service, which the CC(-Bridge) can discover dynamically.  This requires service discovery, but is otherwise cleanly decoupled.

The current implementation is a sort of hodgepodge of the above choices.  The available lifecycle are passed in as arguments to the CC-Bridge, but the Recipe-Generators are hardwired into the code.
