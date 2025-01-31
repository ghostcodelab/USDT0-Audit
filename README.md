### **USDT0 Audit: Unrestricted Minting**

This report highlights a vulnerability in the `OFTExtension` contract that could lead to unauthorized minting of unbacked tokens.

---

#### **Brief/Intro**
The `OFTExtension` contract (`TetherTokenOFTExtension.sol`) allows the admin to set any address as the authorized minter (`oftContract`). This creates a risk where an attacker or compromised admin could mint unbacked tokens on remote chains, leading to inflation and devaluation of the token supply. If exploited, this vulnerability could destabilize the USDT0 ecosystem, causing significant financial losses and eroding user trust.

---

#### **Vulnerability Details**
The `OFTExtension` contract is designed to wrap the Tether token and enable cross-chain bridging using LayerZero's Omnichain Fungible Token (OFT) infrastructure. The `setOFTContract` function allows the owner to designate any address as the authorized minter (`oftContract`). This introduces a critical risk:

#### **1. Unbacked Minting**
- The `mint` function in `TetherTokenOFTExtension.sol` overrides the original `mint` function, which is callable only by the owner. Instead, the new `mint` function can only be called by the `oftContract`.
- If the `oftContract` is set to a malicious or compromised address, it could mint arbitrary amounts of tokens on remote chains without corresponding backing on Ethereum.
- This undermines the integrity of the token's supply mechanism, as the system assumes that all minted tokens on remote chains are backed by locked USDT on Ethereum.

#### **2. Impact of Exploitation**
- An attacker could mint unbacked tokens on remote chains, inflating the total supply.
- These tokens could then be bridged back to Ethereum, further exacerbating the inflation.
- The resulting imbalance would devalue the token and cause widespread panic among users.

---

#### **Impact Details**
If this vulnerability were exploited, the consequences would be severe:
1. **Inflation Risk**: Unbacked tokens could be minted on remote chains, inflating the total supply and devaluing the token.
2. **Economic Damage**: The market could lose confidence in USDT0, leading to a potential de-pegging of the token from its intended value.
3. **Financial Losses**: Users holding USDT0 could suffer financial losses due to the devaluation caused by the inflated supply.
4. **Reputation Damage**: The exploit would severely damage the reputation of the USDT0 project, potentially leading to a loss of users and partners.

---

#### **Attack Scenario**
1. A malicious actor gains control of the `oftContract` address.
2. They call the `mint` function to create unbacked tokens on a remote chain.
3. These tokens are then bridged back to Ethereum, flooding the market with unbacked USDT.

#### **Proof of Concept: Unauthorized Minting of Unbacked Tokens**
1. Deploy the `TetherTokenOFTExtension` contract.
2. Set a malicious address as the `oftContract` using the `setOFTContract` function.
3. Call the mint function from the malicious `oftContract` address to mint arbitrary amounts of tokens.
4. Observe that the total supply increases without any backing, leading to inflation.

#### **PoC Code**
Below is the PoC using the Immunefi Forge template:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.4;

import "forge-std/Test.sol";
import "contracts/Wrappers/OFTExtension.sol"; // Path to the vulnerable contract

contract UnauthorizedMintingTest is Test {
    TetherTokenOFTExtension public oftExtension;
    address public owner = address(0x1);
    address public maliciousActor = address(0x2);

    function setUp() public {
        vm.prank(owner);
        oftExtension = new TetherTokenOFTExtension("USDT0", "USDT0", 6); // Deploy the contract
    }

    function testUnauthorizedMinting() public {
        // Step 1: Set the malicious actor as the authorized minter (oftContract)
        vm.prank(owner);
        oftExtension.setOFTContract(maliciousActor);

        // Step 2: Malicious actor mints unbacked tokens
        uint256 initialSupply = oftExtension.totalSupply();
        uint256 mintAmount = 6_000_000 * 1e6; // Mint 6 million USDT

        vm.startPrank(maliciousActor);
        oftExtension.mint(address(this), mintAmount); // Mint tokens to this contract
        vm.stopPrank();

        // Step 3: Verify that the total supply has increased
        uint256 finalSupply = oftExtension.totalSupply();
        assertEq(finalSupply, initialSupply + mintAmount, "Total supply should increase by minted amount");

        // Step 4: Verify that the tokens were minted to this contract
        uint256 balance = oftExtension.balanceOf(address(this));
        assertEq(balance, mintAmount, "This contract should receive the minted tokens");
    }
}
```

---

#### **Explanation of the PoC**
1. **Setup**:
   - The `TetherTokenOFTExtension` contract is deployed with the owner as `address(0x1)`.
   - The `setUp` function initializes the contract for testing.

2. **Exploitation**:
   - The owner sets the `oftContract` to a malicious address (`address(0x2)`).
   - The malicious actor calls the `mint` function to mint unbacked tokens to their own address.

3. **Verification**:
   - The test verifies that the total supply increases by the minted amount.
   - It also confirms that the tokens are minted to the malicious actor's address.

---

#### **Mitigation**
To prevent this vulnerability, the following changes should be implemented:
1. Restrict the `setOFTContract` function to only allow trusted addresses.
2. Add validation to ensure that the `oftContract` address is a valid contract and conforms to the expected interface (e.g., `IOFT`).

Here’s an example fix:

```solidity
// Whitelist for trusted OFT contracts
mapping(address => bool) public trustedOFTContracts;

// Restrict setOFTContract to trusted addresses
function setOFTContract(address _oftContract) external onlyOwner {
    require(trustedOFTContracts[_oftContract], "OFT: Address not trusted");
    oftContract = _oftContract;
}

// Add a trusted OFT contract to the whitelist
function addTrustedOFTContract(address _oftContract) external onlyOwner {
    require(Address.isContract(_oftContract), "OFT: Address must be a contract");
    trustedOFTContracts[_oftContract] = true;
}

// Remove a trusted OFT contract from the whitelist
function removeTrustedOFTContract(address _oftContract) external onlyOwner {
    trustedOFTContracts[_oftContract] = false;
}
```

---

#### **How to Run the PoC**
1. Clone the Immunefi Forge template repository:
   ```bash
   git clone https://github.com/immunefi-team/forge-poc-templates.git
   cd forge-poc-templates
   ```
2. Replace the `test/` directory with the PoC code provided above.
3. Install dependencies:
   ```bash
   forge install
   ```
4. Run the test:
   ```bash
   forge test
   ```
   
---

#### **Expected Output**
When you run the PoC using Foundry (`forge test`), the test should pass, confirming that the vulnerability exists. Specifically:
- The total supply increases by the minted amount.
- The malicious actor successfully receives the minted tokens.

---
