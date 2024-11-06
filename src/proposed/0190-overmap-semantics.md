<!--
  SPDX-License-Identifier: CC-BY-SA-4.0
  Copyright 2024 UNSW
-->

# Overmap consistency and semantics

- Author: Alwin Joshy
- Proposed: 2024-10-21

## Summary


'Overmapping' refers to the the ability to overwrite an existing virtual memory mapping with a new
one _without_ having to explicitly remove the original mapping. This RFC proposes the removal of 
the 'overmap' behaviour of the PageMap invocation and consistent application of this to all
architectures.


## Motivation

The main motivation for this RFC is to address this inconsistency prior to the proposal of another RFC
on batched mapping/unmapping operations, which has been requested by the community. The batch mapping
operations should have the same semantics as normal page mapping, and this will be simplified if 
all of the architectures behave in the same way.

We believe that the best way to address this is to make all of the architectures consistent, and
choosing to either allow or prevent overmapping on all architectures. Between the two, we believe that
removing overmapping is the better solution.

In summary, this RFC has two main goals:

- Determine whether all architectures should have the same behaviour w.r.t overmapping
- Determine whether overmapping should be permitted


## Guide-level explanation

The current implementation of PageMap on the architectures that allows overmapping is confusing and
contradicts the API reference. The seL4 reference manual states that for all  architectures, their
respective PageMap operations should return `seL4_DeleteFirst` if

> A mapping already exists at vspace at vaddr

This is currently followed on all architectures if a page table already exists in the page table
slot where the mapping would be placed, but only returns the error if a page is already mapped in 
the slot on RISC-V.

## Reference level explanation

On architectures other than RISC-V, the existing mapping will be silently overwritten and replaced
with the new mapping, turning the capability used for the original mapping into a _stale_ capability.
This capability holds informations about a mapping, but is no longer used in the kernel for this
mapping, making it essentially something like a dangling pointer. This state is not unsafe, as a stale
capability can't be used to affect the new mapping (with an unmap for example), but seems like
a very easy footgun and avenue for memory leaks if care is not taken.

Another reason that overmapping should be prohibited is that it makes it impossible for a supervisor
to enforce a specific address space layout for a supervisee. A supervisor may rely on some fixed
mappings in a supervisee and not pass the capabilities for these frames to the supervisee, but provide
it with some other ones so that it can dynamically allocate working memory. With the ability to
overmap, the supervisee can overwrite mappings made by the supervisor without presenting the
original frame caps used for the mappings,

The main advantage for allowing overmapping would be for performance, as you can replace a mapping
with a single invocation instead of two, but even this isn't exactly the case due to the capability
that was used for the original mapping being left as stale. A stale capability cannot be used to create
a new mapping inside a different VSpace or even at a different virtual address without being
explicitly unmapped, meaning that overmapping does not eliminate the second invocation, just
defers it.

Note: We suggest retaining the behaviour where PageMap can be used to Remap (change the permissions 
of) an existing mapping by providing the same capability.

Draft implementation: https://github.com/seL4/seL4/pull/967

## Drawbacks

Any implementations that rely on overmapping would be broken.

## Rationale and alternatives

- Keep it as is: Seems overly complicated to have different implementations for different
architectures, especially if there's no good reason to, and if we do this, we should at least 
update the reference manual.
- Change all to allow overmapping: If we allow overmap on all architectures, we should also change
the implementation of PageMap to be better aware of stale capabilities to get performance benefits,
as this is the only reason we can think of to allow overmapping. However, it is not clear whether
the added performance from this is worth the extra complexity, more widespread use of error-prone
stale capabilities and possible verification impacts. 

## Prior art

On Aarch64, there used to be an additional Page_Remap invocation, which was used for changing the
protections of a page. When this was the case, overmapping was not permitted, but this was
eventually merged with PageMap, and it became permitted after this.

x86 and Aarch32 have permitted overmapping since at least 3.0.0

Stale capabilities are mentioned in the 
[caveats](https://github.com/seL4/seL4/blob/master/CAVEATS.md#re-using-address-spaces), but only 
in the context of deleting page tables without unmapping the pages inside them. 

Prior tangential discussion: https://sel4.discourse.group/t/pre-rfc-protectn/478

## Unresolved questions
