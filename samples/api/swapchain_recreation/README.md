<!--
- Copyright (c) 2023, The Khronos Group
-
- SPDX-License-Identifier: Apache-2.0
-
- Licensed under the Apache License, Version 2.0 the "License";
- you may not use this file except in compliance with the License.
- You may obtain a copy of the License at
-
-     http://www.apache.org/licenses/LICENSE-2.0
-
- Unless required by applicable law or agreed to in writing, software
- distributed under the License is distributed on an "AS IS" BASIS,
- WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
- See the License for the specific language governing permissions and
- limitations under the License.
-
-->
### Swapchain Recreation<br/>
A sample that implements best practices in handling present resources and swapchain recreation, for
example due to window resizing or present mode changes.

Before VK_EXT_swapchain_maintenance1, there is no straightforward way to tell when a semaphore
associated with a present operation can be recycled, or when a retired swapchain can be destroyed.
Both these operations depend on knowing when the presentation engine has acquired a reference to
these resources as part of the present job, for which there is no indicator.

In this sample, a workaround is implemented where a fence signaled by vkAcquireNextImageKHR is used
to determine when the _previous_ present job involving the same image index has been completed.
This is often much later than the point where the present resources can be freed.

Take the following shorthand notation:

- PE: Presentation Engine
- ANI: vkAcquireNextImageKHR
- QS: vkQueueSubmit
- QP: vkQueuePresentKHR
- W: Wait
- S: Signal
- R: Render
- P: Present
- SN: Semaphore N
- IN: Swapchain image N
- FN: Fence N

Assuming both ANI calls below return the same index:

    CPU: ANI  ... QS   ... QP         ANI  ... QS   ... QP
         S:S1     W:S1     W:S2       S:S3     W:S3     W:S4
         S:F1     S:S2                S:F2     S:S4
    GPU:          <------ R ------>            <------ R ------>
     PE:                           <-- P -->                    <-- P -->

The following holds:

    F2 is signaled
    => The PE has handed the image to the application
    => The PE is no longer presenting the image (the first P operation is finished)
    => The PE is done waiting on S2

At this point, we can destroy or recycle S2.  To implement this, a history of present operations is
maintained, which includes the wait semaphore used with that presentation.  Associated with each
present operation, is a fence that is used to determine when that semaphore can be destroyed.

Since the fence is not actually known at present time (QP), the present operation is kept in history
without an associated fence.  Once ANI returns the same index, the fence given to ANI is associated
with the previous QP of that index.

After each present call, the present history is inspected.  Any present operation whose fence is
signaled is cleaned up.

## Swapchain recreation

When recreating the swapchain, all images are eventually freed and new ones are created, possibly
with a different count and present mode.  For the old swapchain, we can no longer rely on a future
ANI to know when a previous presentation's semaphore can be destroyed, as there won't be any more
acquisitions from the old swapchain.  Similarly, we cannot know when the old swapchain itself can be
destroyed.

This issue is resolved by deferring the destruction of the old swapchain and its remaining present
semaphores to the time when the semaphore corresponding to the first present of the new swapchain
can be destroyed.  Because once the first present semaphore of the new swapchain can be destroyed,
the first present operation of the new swapchain is done, which means the old swapchain is no longer
being presented.

Note that the swapchain may be recreated without a second acquire.  This means that the swapchain
could be recreated while there are pending old swapchains to be destroyed.  The destruction of both
old swapchains must now be deferred to when the first QP of the new swapchain has been processed.
If an application resizes the window constantly and at a high rate, we would keep accumulating old
swapchains and not free them until it stops.
