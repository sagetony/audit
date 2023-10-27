## Severity

High Risk

## Relevant GitHub Links

https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L26

https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L14

## Summary

The PasswordStore contract yielded an aggregated total of 3 unique vulnerabilities. Of these vulnerabilities, 2 were HIGH severity.

Additionally, the analysis included 1 has an issue with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

## Vulnerability Details

## High Risk Issues

1. The PasswordStore.sol/setPassword() can be called by anybody which should be only by the owner.

2. People can also see the password even though it is written in with private visibility when the contract is deployed. It should be encrypted.

### Low Risk and Non-Critical Issues

It is best practice to use a modifier to manage who called the functions and restrict it to only the owner.

## Impact

The contract is highly risky, it can’t secure the password from both the blockchain and the contract itself. Anyone can call the setPassword(). The password can also be viewed by anyone on the blockchain, so the goal of the contract is defeated.

## Tools Used

VsCode

## Recommendations

1. Make use of a modifier to manage and restrict the functions to only be called by the owner.

   modifier onlyOwner() {
   if (s*owner != msg.sender) revert PasswordStore\_\_NotOwner();
   *;
   }

   function setPassword(string memory newPassword) external onlyOwner {
   s_password = newPassword;
   emit SetNetPassword();
   }

   function getPassword() external view onlyOwner returns (string memory) {
   return s_password;
   }

2. To make the password not visible to other persons, it is best you use an encrypted method to make the password secure.
