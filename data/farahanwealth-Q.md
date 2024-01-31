Bug Description:
The code imports multiple modules without explicitly specifying their versions. This can potentially lead to security vulnerabilities if the imported modules have known security issues. It is recommended to specify the exact version of each module to ensure that only the intended and secure versions are used.

Steps to Reproduce:
1. Review the code in the  `shrine`  module.
2. Observe the import statements for various modules.
3. Note that the versions of the imported modules are not explicitly specified.

Expected Behavior:
The code should explicitly specify the versions of the imported modules to ensure that only secure and intended versions are used.

Actual Behavior:
The code imports multiple modules without explicitly specifying their versions, which can potentially lead to security vulnerabilities.


Severity:
This bug has a low severity as it does not directly impact the functionality of the code. However, it is still important to address this issue to ensure the security of the code.
