# SIP-2: Private Resources

- **Status:** Draft

## Summary

This RFC allows users of the network to declare access levels for their Resources and Resource functions. When a resource function is private, it can only be accessed by an allowlist of public keys representing the function’s possible callers. Callers sign their request with a private key and the provider verifies it before allowing access to the function.

The signature for a private function is still broadcast to the network, but accessing it is prohibited for clients who are not authorized using the function’s allowlist.

## Goals

- Resource owners declare an access level for each protected function of a Resource.
  - This access level is declared as part of the resource function declaration at compile time.
- Resource owners declare a list of allowed clients for that function.
  - This list is declared along with the resource function declaration at compile time.
- Clients can call a protected resource function, if they are in the allowlist.
- Clients write a signature using their private key and carry it with their request.
- The signature expires after its single use.
