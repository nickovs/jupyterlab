.. Copyright (c) Jupyter Development Team.
.. Distributed under the terms of the Modified BSD License.

Performance tricks
==================

Notebook documents
------------------

Performance analysis of notebook documents in JupyterLab point out to two main bottlenecks:
editors style computation and rendered markdown with lots of math expressions. The best
technique to alleviate those problems are to attach to the DOM only the cells in the viewport.
However code cell outputs bring a huge constraint that their internal state is within the
DOM node (e.g. the zoom level on a map view). So the outputs cannot be detached once displayed
With those considerations in mind, here is the algorithm coded for the notebook windowing.

When cell widgets are instantiated, their children are not created (i.e. the editor, the
outputs,…) and they are not attached to the DOM. The view is updated on scroll events following:

1. Get the scroll position in the notebook
2. Get the cells range to be displayed
3. Attach the cells in viewport to the notebook
   a. Before attaching the cell, if it was never attached, instantiate its children.
   b. For code cells previously attach, change its visibility and attach all children except the output area.
   c. Add the cell widget to a resize observer that trigger and viewport update
4. Detach cells that left the viewport
   a. Remove cell widget from resize observer.
   b. For code cells, the output area is not detached but hidden. All other children are detached.

Due to the code cells being kept in the DOM, we need to update the data attribute
``data-windowed-list-index`` when the cells list changes. This is required to attach at the
correct position the cell when the viewport changes.

The list windowing algorithm is coded in a dedicated widget ``WindowedList``. The notebook
extends it and uses a specialized layout to deal with code cells, ``NotebookWindowedLayout``.
The code cell layout is also customized ``CodeCellLayout`` to keep the output area attached
but not the other children.

.. warning::

    The current implementation does not handle *iframe* well. In Chrome, the iframe states are
    reset every time they leave and renter the viewport. In Firefox, it does not happen when
    scrolling comes from mouse scrolls. But the reset happens when jumping to a specific position
    that was out of viewport; e.g. searching for a word or navigating to a heading.

Side effect of the implementation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- It is still possible to search cell outputs DOM nodes. But when turning on that option,
  the UI may freeze as all outputs never rendered will need to be rendered.
- The cell editor is available only for cell that have been at least once in the viewport.
  The editor is not destroyed when the cells are out of the viewport. So its state can be modified.

Viewport state
^^^^^^^^^^^^^^

Each cell widget implements those three attributes:

- ``isPlaceholder`` will be false if the cell has never been attached (and therefore have no children instantiated).
- ``ready`` returns a promise that resolves when a cell is fully instantiated. After that the editor and any children will exist.
- ``inViewport`` whether the cell is currently in viewport or not. You can listen to the signal inViewportChanged.

The notebook widget has the following helper :

- ``scrollToItem`` to scroll to a specific cell index. Like for scrolling an element into view, various mode are available.
