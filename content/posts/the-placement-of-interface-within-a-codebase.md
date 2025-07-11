+++
title = 'The Placement of Interface Within a Codebase'
date = 2025-07-04T19:13:06+01:00
+++


The definition of an interface should be in the location of where it is required not where it is implemented.

Defining the interface next to the implementing functionality improves the readability of that functionality. You know that the calling code calls the methods described on the adjacent interface. If you are sure to have followed the Interface Segregation Principle you can also be sure of the exact methods used by the calling code.

The Interface Segregation Principle states;

`"Clients should not be forced to depend on interfaces they do not use."`

The only time I break this rule is when an identical interface is required in three or more places within a codebase. In times like this I would take the DRYness over the principles.