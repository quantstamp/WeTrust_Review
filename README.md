# Overview

This smart contract audit was prepared by [Quantstamp](https://www.quantstamp.com/), the protocol for securing smart contracts.

## Specification

Our understanding of the specification was based on the description of the white-paper dated February 26, 2017. The white-paper does not replace a proper specification, but the provided examples gave us an intuitive understanding of the protocol. We also reviewed the token sale provided in the [README.md](https://github.com/WeTrustPlatform/rosca-contracts/blob/develop/README.md) file in the github repository at the time of the audit. 

## Methodology

The review was conducted during 2017-Nov-01 thru 2017-Nov-10 by the Quantstamp
team, which included senior engineers Kacper Bak, Ed Zulkoski and Steven
Stewart.

Their procedure can be summarized as follows:

1. Code review
    * Review of the specification 
    * Manual review of code
    * Comparison to specification
2. Testing and automated analysis
    * Test coverage analysis
    * Symbolic execution (automated code path evaluation)
3. Best-practices review
4. Itemize recommendations

## Source Code

The following source code was reviewed during the audit.

| Repository                                                             | Commit                                                                                                        |
|------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| [rosca-contracts](https://github.com/WeTrustPlatform/rosca-contracts)  | [2cb0011](https://github.com/WeTrustPlatform/rosca-contracts/commit/2cb0011756ed92046c373fb7ecca4878cf0a8da1) |

# Security Audit

Quantstamp's objective was to evaluate the WeTrust ROSCA code for security-related issues, code quality, and adherence to best-practices.

Possible issues include (but are not limited to):

* Transaction-ordering dependence
* Timestamp dependence
* Mishandled exceptions and call stack limits
* Unsafe external calls
* Integer overflow / underflow
* Number rounding errors
* Reentrancy and cross-function vulnerabilities
* Denial of service / logical oversights

# Test coverage

We evaluated the test coverage using truffle and solidity-coverage. The below notes outline the setup and steps that were performed.

## Setup

Testing setup:
* Truffle v3.4.11
* TestRPC v4.1.3
* solidity-coverage v0.2.5
* oyente v0.2.7

## Steps

Steps taken to run the full test suite:

* Following instruction from the readme file [README.md](https://github.com/WeTrustPlatform/rosca-contracts/blob/develop/README.md). Specifically, running the command: ```tools/build/run-tests.sh```. First, it creates ROSCATest.sol with all the internal fields exposed as public to enable white-box testing. Second, it runs the tests via truffle. All the tests in the suite passed.

* No additional steps needed to run solidity-coverage on the test suite.

## Evaluation

Based on both automated and manual analysis, we noted that test coverage the ROSCA smart contract (ROSCA.sol and ROSCATest.sol) was fairly good, yet it missed some important edge cases. 

We did observe that the ```if``` or```else``` paths for numerous statements were not covered by the existing tests. These include:

* Lines 260, 452, 476, ROSCATest.sol.

Notably, the predetermined type of ROSCA was not exercised in the tests. The tests did not cover:

* Lines 241-242, 250-252, ROSCATest.sol.

Although we do not anticipate any issues, the functions that are meant to be used in an emergency, when the “Escape Hatch” is active, were not covered by the tests:

* Lines 142-145, 157-160, 547-578, ROSCATest.sol.

For more complete coverage, it is desirable to include tests that exercise all code paths, including those for which reversions of state will be triggered when a call to ```require()``` is encountered.

Oyente reported a potential time dependency attack for the function ```endOfROSCARetrieveSurplus()```, however we believe this to be a benign issue. A concern is that a miner could manipulate the timestamp such that the foreperson fails when retrieving the fees. However, even if that happens, the foreperson could just wait several minutes and try again. The timestamp cannot be manipulated in any way to prevent the foreperson from ever retrieving the surplus, nor to allow some other user to obtain it instead. Thus, we do not consider this an issue.


# Recommendations

## Avoiding rounding-errors with division

When calculating fees in ```recalculateTotalFees()``` and ```removeFees()```, amounts are multiplied by some fractional value (e.g. ```(1000 - serviceFeeInThousandths)) / 1000```). This may introduce rounding errors during the calculations, and invalidate some implicit invariants (e.g., that amount == totalFees + removeFees(amount)). If possible, we recommend storing all balances throughout the contract as values multiplied by 1000, in order to avoid any rounding issues from division.


## Restricting the start time of deployed ROSCA networks

On page 26 of the [WeTrust white paper](https://github.com/WeTrustPlatform/documents/blob/master/WeTrustWhitePaper.pdf), it was noted that a “ROSCA must be deployed at least three full days before this date, to protect against blockchain timestamp discrepancies.” This should be added as a require statement in the constructor of ROSCA.

## Code documentation

We noted that majority of the functions were well-documented. Besides description of the behavior, we recommend the usage of documentation tags such as ```@dev```, ```@param``` and ```@returns``` for all functions. Additionally, the contract uses both native and SafeMath functions; they perform computations on numbers that represent currency values. We recommend using SafeMath as the best practice. If native operations suffice, clearly document that the operations are guaranteed not to cause any overflow/underflow/exceptions.

## Visibility modifiers

We recommend always including visibility modifiers for functions and variables, even when they are public. In ROSCA.sol, the visibility modifiers for ```recalculateTotalFees()``` and ```emergencyWithdrawal``` are missing, which should be ```public```.

## Re-entrancy modifier

Whenever the contract issues a transfer of tokens (as in ```endOfROSCARetrieveFees()```), the balance is correctly set to zero beforehand, to avoid re-entrancy attacks. While we do not see any issues with the current functionality of the code, we suggest incorporating a ```nonReentrant``` modifier on the appropriate functions to clearly express the protection against re-entrancy:

```
bool private reentrancyLock = false;
modifier nonReentrant() {
   require(!reentrancyLock);
   reentrancyLock = true;
   _;
   reentrancyLock = false;
}
```

## Compiler warnings

The solidity compiler reported that the ROSCA function ```getParticipantBalance()``` is declared constant, but calls the functions ```requiredContribution()``` and ```removeFees()```, which are not explicitly declared constant. These functions can also be declared constant (or view) to remove the warnings.

## Proper usage of assert and require

We noticed that ```assert``` was used instead of ```require``` in a few places (see lines 220, 447, and 519, ROSCA.sol). Although both statements have similar runtime behavior, they communicate different intentions. The statement ```require``` shall be used for parameter validation, whereas ```assert``` for ensuring that the code itself works as expected.



# Appendix

## File Signatures

Below are SHA256 file signatures of the relevant files reviewed in the audit.

```
$ shasum -a 256 ./contracts/* ./contracts/*/* ./contracts/*/*/*
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./contracts/deps
27152e59ddefda399585be0084dfee34d29b89a0a5cd8ac89369c19000a2dd79  ./contracts/Migrations.sol
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./contracts/older
96ad84e85206b84d1a138b7add9aab74fd4f76adae31c775b74800fe77108d5d  ./contracts/ROSCA.sol
7a197021af8a8d7caa4a15aa2d25e163c9552b46438595fca88ccdfb92f1e729  ./contracts/ROSCATest.sol
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./contracts/test
4ab391b052697f07c296ca682e279e94e9d715e474991b0fc8ab58cce7864015  ./contracts/deps/ERC20TokenInterface.sol
c3ad669fcf5071fc59b91c02a63f80963be8c0b1f931021a19d6dce34b72e7d3  ./contracts/deps/SafeMath.sol
64c5ebb5d1ce32452f1510a2763ffbe48b6f48cc572296953ed15337713773b9  ./contracts/older/ROSCAv1.sol
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./contracts/test/deps
11a68ee7493e15c601912f23cbf501744a4eca27e1a3be5e8c64ff86ce0b48e2  ./contracts/test/ExampleToken.sol
1b99f6a17cea9a3fbd366f368a29c63090105749b558ba8b08e54c4fb9348135  ./contracts/test/TestReEntryAttack.sol
9e74df09a085cf864a1baa5774e50c333ebe52d28d0e5f214d6181d8db753429  ./contracts/test/deps/SafeMath.sol

$ shasum -a 256 ./test/* ./test/*/*
7fbb87d69b993497117964fad4a55c5896cdb0dcda896746ed9c347973cb9522  ./test/4MemberFullRoscaTest.js
dff13ed949d28e3e366b31aee7a19ce3e506708d46975c36cff4cc971b1fa333  ./test/addMemberUnitTest.js
290f2df3f42fcb63b2420ec11e9addd01d83dae9b86e2ee8c7bbe58dcb9dc4ff  ./test/bidUnitTest.js
38be5e786fa3b44ad24d91f2010f0332b0b892531733d4f01a6f53e895d6a647  ./test/cleanUpPreviousRoundUnitTest.js
28067f533690c795846f974003a4ca6b4548f89db8b0de09c7698c0f8bbcf34e  ./test/constructorUnitTest.js
cac32e5f75311b95b1487322cafa89555655fc2509161e96b271ef6719108c83  ./test/contributeUnitTest.js
9c149da4d8e30a6e58426cd01af25a03fb4ca7c99379715540f3e723af4e5165  ./test/endOfROSCAUnitTest.js
131a2dfc80426ff9479df67e916cea8115fd7fe09dd546758ee25caf178da82d  ./test/escapeHatchUnitTest.js
83cd90e4db1c14a9df57e045a63b14c2e76be1625456e4d908a9a8f97d29715c  ./test/feesUnitTest.js
5f325a25e3984c900677af29c80dbc1d51c93e4bc7585abc5702f82293b226cc  ./test/getBalanceUnitTest.js
a1278da2692b3ccd3e6eb99016833c932460e825e6863009a5bc68e1d849edf7  ./test/preDeterminedRoscaTest.js
59d84390b66ebf48e6a30573cc665f399a02013254cacec444671a285a107af7  ./test/randomSelectionROSCATest.js
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./test/reentry
a0a7abaab588b89e1d7d21b28d482627aebb8237d4a52cfb893ddd26c19b54b7  ./test/startRoundUnitTest.js
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./test/utils
79891108cf3f4da9efa12aeba4653745482e3bfc1ec7584d336e5a263ef6a203  ./test/withdrawUnitTest.js
5e027f6cac9b7f329882046f4df27b396a2f5784fedbc22affe5b5728dda0d48  ./test/reentry/reEntryAttackTest.js
a198e1bbf4e6e20a696d9f577312608179f8a2ef35a9276408e4a57bb2fb51c0  ./test/utils/consts.js
dc25f1421924706137857d820d88455dd4c3de46f82c63697df23618c0d3d52a  ./test/utils/roscaHelper.js
d797184fefb55cae4e33562a1f3dd07619167ab6fc7b0ffc90bbdb9f3deeb559  ./test/utils/utils.js


$ shasum -a 256 ./migrations/*
d2a79f32b01bff6750b2fb0df10664654a1511504d7bbcb1195ad55191f88346  ./migrations/1_initial_migration.js
dd4f9a2788819033d7134e72f5424a3a5ff134efe6b56845125dd45890e40217  ./migrations/2_deploy_contracts.js
ac600db652f005cd791fd71af3e72b6cf4fb83d3ef9af0b4d8542176007e19e8  ./migrations/3_deploy_test_contract.js
941b54695b580ad89aa10893d1866d0ba10e43acbf9e6896becfa6460a53fa49  ./migrations/4_deploy_example_token.js


```

# Disclosure

## Purpose of report

The scope of our review is limited to a review of Solidity code and only the source code we note as being within the scope of our review within this report. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks.

The report is not an endorsement or indictment of any particular project or team, and the report does not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset.

No third party should rely on the reports in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project.

## Links to other websites
You may, through hypertext or other computer links, gain access to web sites operated by persons other than Quantstamp Technologies Inc. (QTI). Such hyperlinks are provided for your reference and convenience only, and are the exclusive responsibility of such web sites' owners. You agree that QTI are not responsible for the content or operation of such web sites, and that QTI shall have no liability to you or any other person or entity for the use of third-party web sites. Except as described below, a hyperlink from this web site to another web site does not imply or mean that QTI endorses the content on that web site or the operator or operations of that site. You are solely responsible for determining the extent to which you may use any content at any other web sites to which you link from the report. QTI assumes no responsibility for the use of third-party software on the website and shall have no liability whatsoever to any person or entity for the accuracy or completeness of any outcome generated by such software.

## Timeliness of content
The content contained in the report is current as of the date appearing on the report and is subject to change without notice, unless indicated otherwise by QTI; however, QTI does not guarantee or warrant the accuracy, timeliness, or completeness of any report you access using the internet or other means, and assumes no obligation to update any information following publication.

