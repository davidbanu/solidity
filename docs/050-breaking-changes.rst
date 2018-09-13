********************************
Solidity v0.5.0 Breaking Changes
********************************

This section highlights the main breaking changes introduced in Solidity
version 0.5.0, along with the reasoning behind the changes and how to update
affected code.
For the full list check
`the release changelog <https://github.com/ethereum/solidity/releases>`_.

Semantic Only Changes
=====================

This section lists the changes that are semantic-only, thus potentially
hiding new and different behaviors in previous code.

* Signed right shift now uses proper arithmetic shift, i.e. rounding towards
  negative infinity. Signed and unsigned shift will have dedicated opcodes in
  Constantinople, and are emulated by Solidity for the moment.

* The ``continue`` statement in a ``do...while`` loop now jumps to the
  condition, which is the common behavior in such cases. It used to jump to the
  loop body. Thus, if the condition is false, the loop terminates.

Semantic and Syntactic Changes
==============================

This section highlights changes that affect syntax and semantics.

* The functions ``.call()`` (and family), ``keccak256()``, ``sha256()``,
  ``ripemd160()`` now accept only a single ``bytes`` argument. Moreover, the
  argument is not padded. This was changed to make more explicit and clear how
  the arguments are concatenated. Change every ``.call()`` to a ``.call("")``
  and every ``.call(signature, a, b, c)`` to use
  ``.call(abi.encodeWithSignature(signature, a, b, c))`` (the last one only
  works for value types).  Change every ``keccak256(a, b, c)`` to
  ``keccak256(abi.encodePacked(a, b, c))``. Even though it is not a breaking
  change, it is suggested that developers change
  ``x.call(bytes4(keccak256("f(uint256)"), a, b)`` to
  ``x.call(abi.encodeWithSignature("f(uint256)", a, b))``, that is, it is not
  anymore necessary to call a chain of functions for that purpose.

* Function ``.call()`` now returns ``(bool, bytes memory)``, since
  functions called with the opcode ``staticcall`` might need to return data.
  Change ``bool success = otherContract.call("f")`` to ``(bool success, bytes
  memory data) = otherContract.call("f")``.

* Solidity now implements C99-style scoping rules for function local variables,
  that is, a local variable declared in the function is only valid if declared
  in the same or parenting scope. Declare function variables in the same or
  higher scope as their usage.

Explicitness requirements
=========================

This section lists changes where the code now needs to be more explicit.

* Explicit function visibility is now mandatory.  Add ``public`` to every
  function and constructor, and ``external`` to every fallback or interface
  function that does not specify its visibility already.

* Explicit data location for all variables of struct, array or mapping types is
  now mandatory. This is also applied to function parameters and return
  variables.  For example, change ``uint[] x = m_x`` to ``uint[] storage x =
  m_x``, and ``function f(uint[][] x)`` to ``function f(uint[][] memory x)``
  where ``memory`` is the data location and might be replaced by ``storage`` or
  ``calldata`` accordingly.  Note that ``external`` functions require
  parameters with a data location of ``calldata``.

* Variables of contract type do not contain ``address`` members anymore in
  order to separate the namespaces.  Therefore, it is now necessary to
  explicitly convert values of contract type to addresses before using an
  ``address`` member.  Example: if ``c`` is a contract, change
  ``c.transfer(...)`` to ``address(c).transfer(...)``.

* The ``address`` type  was split into ``address`` and ``address payable``,
  where only ``address payable`` provides the ``transfer`` function.  An
  ``address payable`` can be converted to an ``address``, but the other way
  around is not allowed.

* Conversions between ``bytesX`` and ``uintY`` of different size are now
  disallowed due to ``bytesX`` padding on the right and ``uintY`` padding on
  the left which may cause unexpected conversion results.  The size must now be
  adjusted within the type before the conversion.  For example, you can convert
  a ``bytes4`` (4 bytes) to a ``uint64`` (8 bytes) by first converting the
  ``bytes4`` variable to ``bytes8`` and then to ``uint64``.

* Using ``msg.value`` in ``non-payable`` functions (or introducing it via a
  modifier) is disallowed as a security feature. Turn the function into
  ``payable`` or create a new internal function for the program logic that
  uses ``msg.value``.
