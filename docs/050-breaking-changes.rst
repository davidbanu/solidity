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

Example
=======

The following example shows a contract and its updated version for Solidity
v0.5.0.
Old version:

::

   pragma solidity ^0.4.24;

   contract OtherContract {
      uint x;
      function f(uint y) external {
         x = y;
      }
      function() payable external {}
   }

   contract Old {
      OtherContract other;
      uint myNumber;

      // Function visibility not provided, not an error
      // Function mutability not provided, not an error
      function f(uint x) {
         y = -3 >> 1; // y = -1 (wrong, should be -2)

         do {
            x += 1;
            if (x > 10) continue;
         } while (x < 11); // causes infinite loop

         bool success = other.call("f"); // Call returns only a Bool
         if (!success)
            revert();
         else {
            int y; // Local variables could be declared after their use
         }
      }

      function g(uint[] arr, bytes8 x, OtherContract otherContract) public { // No need to explicit parameter location data
         otherContract.transfer(1 ether);

         // Since uint32 (4 bytes) is smaller than bytes8 (8 bytes),
         // the first 4 bytes of x will be lost. This is dangerous
         // since bytesX are right padded.
         uint32 y = uint32(x);
         myNumber += y + msg.value;
      }
   }

New version:

::

   pragma solidity >0.4.24;

   contract OtherContract {
      uint x;
      function f(uint y) external {
         x = y;
      }
      function() payable external {}
   }

   contract New {
      OtherContract other;
      uint myNumber;

      // Function visibility not provided, not an error
      // Function mutability not provided, not an error
      function f(uint x) public returns (bytes memory) {
         int y = -3 >> 1; // y = -2 (correct)

         do {
            x += 1;
            if (x > 10) continue; // jumps to the condition below
         } while (x < 11);

         // Call returns (bool, bytes)
         // Data location must be specified
         (bool success, bytes memory data) = address(other).call("f");
         if (!success)
            revert();
         return data;
      }

      using address_make_payable for address;
      function g(uint[] memory arr, bytes8 x, OtherContract otherContract) public payable { // No need to explicit parameter location data
         // 'contract.transfer' is not provided
         // 'address(contract).transfer' is not provided since 'address(contract)' is not 'address payable'.
         // So we need to convert variable 'contract' to type 'address payable'.
         address payable addr = address(otherContract).make_payable();
         addr.transfer(1 ether);

         // Since uint32 (4 bytes) is smaller than bytes8 (8 bytes),
         // the conversion is not allowed.
         // We need to convert to a common size first:
         bytes4 x4 = bytes4(x); // Padding happens on the right
         uint32 y = uint32(x4); // Conversion is consistent
         // 'msg.value' cannot be used in a 'non-payable' function
         // We need to make the function payable
         myNumber += y + msg.value;
      }
   }

   // We can define a library for explicitly converting ``address``
   // to ``address payable`` as a workaround.
   library address_make_payable {
      function make_payable(address x) internal pure returns (address payable) {
         return address(uint160(x));
      }
   }
