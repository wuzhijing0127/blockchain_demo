# Transcript Verification Demo in Your Class Style

## 1. Set up the project

In an empty directory:

```bash
npm init -y
npm install --save-dev hardhat@^2.14.0
npx hardhat
```

Choose:

- Create an empty `hardhat.config.js`

Then install toolbox like your lab does:

```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox
```

## 2. `hardhat.config.js`

Use this:

```js
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  defaultNetwork: "localhost",
  networks: {
    hardhat: {
      initialBaseFeePerGas: 0,
      chainId: 31337
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    }
  },
  solidity: "0.8.20",
};
```

This mirrors the same structure your lab uses with `localhost`, `31337`, and `initialBaseFeePerGas: 0`.

## 3. Make the folders

```bash
mkdir contracts
mkdir scripts
```

## 4. `contracts/TranscriptVerification.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract TranscriptVerification {

    struct Credential {
        address issuer;       // university wallet
        address student;      // student wallet
        bytes32 transcriptHash;
        uint256 issuedAt;
        bool exists;
        bool revoked;
    }

    mapping(bytes32 => Credential) public credentials;

    event CredentialIssued(
        bytes32 indexed credentialId,
        address indexed issuer,
        address indexed student,
        bytes32 transcriptHash,
        uint256 issuedAt
    );

    event CredentialRevoked(bytes32 indexed credentialId);

    function issueCredential(
        bytes32 credentialId,
        address student,
        bytes32 transcriptHash
    ) public {
        require(student != address(0), "Invalid student address");
        require(!credentials[credentialId].exists, "Credential already exists");

        credentials[credentialId] = Credential({
            issuer: msg.sender,
            student: student,
            transcriptHash: transcriptHash,
            issuedAt: block.timestamp,
            exists: true,
            revoked: false
        });

        emit CredentialIssued(
            credentialId,
            msg.sender,
            student,
            transcriptHash,
            block.timestamp
        );
    }

    function revokeCredential(bytes32 credentialId) public {
        require(credentials[credentialId].exists, "Credential does not exist");
        require(credentials[credentialId].issuer == msg.sender, "Only issuer can revoke");

        credentials[credentialId].revoked = true;
        emit CredentialRevoked(credentialId);
    }

    function verifyCredential(
        bytes32 credentialId,
        address expectedIssuer,
        address expectedStudent,
        bytes32 expectedTranscriptHash
    ) public view returns (bool) {
        Credential memory c = credentials[credentialId];

        if (!c.exists) return false;
        if (c.revoked) return false;
        if (c.issuer != expectedIssuer) return false;
        if (c.student != expectedStudent) return false;
        if (c.transcriptHash != expectedTranscriptHash) return false;

        return true;
    }

    function getCredential(bytes32 credentialId)
        public
        view
        returns (
            address issuer,
            address student,
            bytes32 transcriptHash,
            uint256 issuedAt,
            bool exists,
            bool revoked
        )
    {
        Credential memory c = credentials[credentialId];
        return (
            c.issuer,
            c.student,
            c.transcriptHash,
            c.issuedAt,
            c.exists,
            c.revoked
        );
    }
}
```

## 5. `scripts/demo.js`

This script deploys the contract, issues a credential, verifies it, shows tampering fails, and shows revocation fails verification too.

```js
const { ethers } = require("hardhat");

async function main() {
  const [deployer, university, student, employer, attacker] = await ethers.getSigners();

  console.log("=== Accounts ===");
  console.log("Deployer:   ", deployer.address);
  console.log("University: ", university.address);
  console.log("Student:    ", student.address);
  console.log("Employer:   ", employer.address);
  console.log("Attacker:   ", attacker.address);
  console.log("");

  const TranscriptVerification = await ethers.getContractFactory("TranscriptVerification");
  const transcriptVerification = await TranscriptVerification.deploy();
  await transcriptVerification.deployed();

  console.log("Contract deployed to:", transcriptVerification.address);
  console.log("");

  // Example off-chain transcript data
  const studentName = "Jimmy Wu";
  const schoolName = "Carnegie Mellon University";
  const degree = "B.S. Computational & Applied Mathematics";
  const gradYear = "2026";
  const gpa = "3.80";

  // The full transcript is NOT stored on-chain.
  // Instead, we hash the transcript data off-chain.
  const transcriptString = `${studentName}|${schoolName}|${degree}|${gradYear}|${gpa}`;
  const transcriptHash = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes(transcriptString)
  );

  // A credentialId can also be generated off-chain
  const credentialString = `${student.address}|${schoolName}|${degree}|${gradYear}`;
  const credentialId = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes(credentialString)
  );

  console.log("Transcript string:", transcriptString);
  console.log("Transcript hash:  ", transcriptHash);
  console.log("Credential ID:    ", credentialId);
  console.log("");

  // University issues credential
  console.log("=== Step 1: University issues credential ===");
  let tx = await transcriptVerification
    .connect(university)
    .issueCredential(credentialId, student.address, transcriptHash);
  await tx.wait();
  console.log("Credential issued.");
  console.log("");

  // Read stored credential
  console.log("=== Step 2: Read credential from chain ===");
  let credential = await transcriptVerification.getCredential(credentialId);
  console.log("Issuer:         ", credential.issuer);
  console.log("Student:        ", credential.student);
  console.log("TranscriptHash: ", credential.transcriptHash);
  console.log("IssuedAt:       ", credential.issuedAt.toString());
  console.log("Exists:         ", credential.exists);
  console.log("Revoked:        ", credential.revoked);
  console.log("");

  // Employer verifies correct transcript
  console.log("=== Step 3: Employer verifies correct credential ===");
  let isValid = await transcriptVerification
    .connect(employer)
    .verifyCredential(
      credentialId,
      university.address,
      student.address,
      transcriptHash
    );
  console.log("Verification result (should be true):", isValid);
  console.log("");

  // Tampered transcript example
  const fakeTranscriptString = `${studentName}|${schoolName}|${degree}|${gradYear}|4.00`;
  const fakeTranscriptHash = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes(fakeTranscriptString)
  );

  console.log("=== Step 4: Employer verifies tampered transcript ===");
  let tamperedValid = await transcriptVerification
    .connect(employer)
    .verifyCredential(
      credentialId,
      university.address,
      student.address,
      fakeTranscriptHash
    );
  console.log("Verification result (should be false):", tamperedValid);
  console.log("");

  // Wrong issuer example
  console.log("=== Step 5: Employer verifies with wrong issuer ===");
  let wrongIssuerValid = await transcriptVerification
    .connect(employer)
    .verifyCredential(
      credentialId,
      attacker.address,
      student.address,
      transcriptHash
    );
  console.log("Verification result (should be false):", wrongIssuerValid);
  console.log("");

  // Revoke credential
  console.log("=== Step 6: University revokes credential ===");
  tx = await transcriptVerification
    .connect(university)
    .revokeCredential(credentialId);
  await tx.wait();
  console.log("Credential revoked.");
  console.log("");

  // Verify after revocation
  console.log("=== Step 7: Employer verifies revoked credential ===");
  let revokedValid = await transcriptVerification
    .connect(employer)
    .verifyCredential(
      credentialId,
      university.address,
      student.address,
      transcriptHash
    );
  console.log("Verification result (should be false):", revokedValid);
  console.log("");

  console.log("=== Summary ===");
  console.log("1. University issued a credential hash on-chain.");
  console.log("2. Employer verified issuer, student, and transcript integrity.");
  console.log("3. Tampered transcript failed verification.");
  console.log("4. Wrong issuer failed verification.");
  console.log("5. Revoked credential also failed verification.");
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

## 6. Compile

Like your lab:

```bash
npx hardhat compile
```

## 7. Start the local node

Again, same style as your lab:

```bash
npx hardhat node
```

Leave this terminal running.

## 8. Run the demo script in a new terminal

```bash
npx hardhat run scripts/demo.js --network localhost
```

## 9. What this demo is showing

This demo is intentionally narrow, which matches your professor's advice. It focuses on the third issue he suggested: how an employer verifies that a credential came from the right institution and belongs to the right student.

The system works like this:

- The university does not store the whole transcript on-chain.
- Instead, it stores a hash of the transcript, plus the student address and issuer address.
- The employer receives the transcript off-chain and hashes it independently.
- If the hash matches what is on-chain, and the issuer/student also match, the credential is valid.

That helps you emphasize the main conceptual point:

> the blockchain is being used as a verification layer, not a full transcript database.

## 10. If you want a console-style demo instead of a script

Your lab also uses `npx hardhat console` and manual commands.

After starting `npx hardhat node`, run:

```bash
npx hardhat console --network localhost
```

Then paste these commands one by one:

```js
const [deployer, university, student, employer] = await ethers.getSigners()
const TranscriptVerification = await ethers.getContractFactory("TranscriptVerification")
const contract = await TranscriptVerification.deploy()
await contract.deployed()
const transcriptString = "Jimmy Wu|Carnegie Mellon University|B.S. Computational & Applied Mathematics|2026|3.80"
const transcriptHash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(transcriptString))
const credentialString = `${student.address}|Carnegie Mellon University|B.S. Computational & Applied Mathematics|2026`
const credentialId = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(credentialString))
await contract.connect(university).issueCredential(credentialId, student.address, transcriptHash)
await contract.getCredential(credentialId)
await contract.connect(employer).verifyCredential(
  credentialId,
  university.address,
  student.address,
  transcriptHash
)
```

Then you can test tampering:

```js
const badHash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes("Jimmy Wu|Carnegie Mellon University|B.S. Computational & Applied Mathematics|2026|4.00"))
await contract.connect(employer).verifyCredential(
  credentialId,
  university.address,
  student.address,
  badHash
)
```
