# Transcript Verification Demo

A minimal Ethereum smart-contract demo that shows how a university can issue
academic credentials on-chain and how a third party (e.g. an employer) can
verify them. The full transcript is **never stored on-chain** — only a hash of
it, together with the issuer and student addresses. This keeps personal data
off the public ledger while still making the credential tamper-evident.

The project uses [Hardhat](https://hardhat.org/) for local development and
runs entirely on a local in-memory blockchain, so no real ETH or testnet
account is required.

---

## What the demo proves

1. A university issues a credential by writing a hash of the transcript to the
   contract, along with the student's address.
2. An employer can independently hash a transcript they receive off-chain and
   compare it to what is stored on-chain.
3. If the transcript has been tampered with, verification fails.
4. If a different (impostor) issuer is claimed, verification fails.
5. If the university revokes the credential, verification fails.

The blockchain is used as a **verification layer**, not as a transcript
database.

---

## Prerequisites

- [Node.js](https://nodejs.org/) 18 or newer
- npm (bundled with Node.js)

---

## 1. Project setup

In an empty directory:

```bash
npm init -y
npm install --save-dev hardhat@^2.14.0
npx hardhat
```

When prompted, choose **"Create an empty hardhat.config.js"**.

Then install the Hardhat toolbox:

```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox
```

---

## 2. `hardhat.config.js`

Replace the generated config with:

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

`initialBaseFeePerGas: 0` keeps gas costs at zero on the local chain, which
makes the demo easier to read.

---

## 3. Create the source folders

```bash
mkdir contracts
mkdir scripts
```

---

## 4. The smart contract — `contracts/TranscriptVerification.sol`

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

### How it works

- `issueCredential` — called by the issuer (university). Records the hash of
  the transcript and the student's address under a unique `credentialId`. The
  caller becomes the recorded issuer.
- `revokeCredential` — only the original issuer can revoke a credential.
- `verifyCredential` — a pure read function. Returns `true` only if the
  credential exists, is not revoked, and the expected issuer / student /
  transcript hash all match.
- `getCredential` — returns the raw record for inspection.

---

## 5. The demo script — `scripts/demo.js`

This script deploys the contract, issues a credential, verifies it, then
demonstrates that tampering, impersonation, and revocation all cause
verification to fail.

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
  const studentName = "Alice Doe";
  const schoolName = "Example University";
  const degree = "B.S. Computer Science";
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

---

## 6. Compile the contract

```bash
npx hardhat compile
```

---

## 7. Start a local blockchain node

```bash
npx hardhat node
```

This starts a local Ethereum node on `http://127.0.0.1:8545` with several
pre-funded test accounts. Leave this terminal running.

---

## 8. Run the demo

In a new terminal:

```bash
npx hardhat run scripts/demo.js --network localhost
```

You should see seven labelled steps in the output. Steps 3 should print
`true`; steps 4, 5, and 7 should print `false`.

---

## 9. Conceptual summary

- The university does **not** store the full transcript on-chain. It stores a
  hash of the transcript, plus the student address and the issuer (its own)
  address.
- An employer receiving the transcript off-chain re-hashes it independently
  and compares the hash to what is on-chain.
- If the hashes match and the issuer / student addresses match, the
  credential is valid.
- Any tampering with the transcript changes the hash, so verification fails.
- Impersonating a different issuer also fails, because the on-chain record
  pins the credential to the address that issued it.
- Revocation is honored: a revoked credential will never verify as valid
  again.

The result is a small, auditable trust anchor on-chain while keeping personal
data off-chain.

---

## 10. Optional: interactive console walkthrough

If you prefer to run the steps interactively instead of as a script, start
the node:

```bash
npx hardhat node
```

Then in a new terminal open the Hardhat console:

```bash
npx hardhat console --network localhost
```

Paste these commands one by one:

```js
const [deployer, university, student, employer] = await ethers.getSigners()
const TranscriptVerification = await ethers.getContractFactory("TranscriptVerification")
const contract = await TranscriptVerification.deploy()
await contract.deployed()

const transcriptString = "Alice Doe|Example University|B.S. Computer Science|2026|3.80"
const transcriptHash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(transcriptString))

const credentialString = `${student.address}|Example University|B.S. Computer Science|2026`
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

To test that tampering is detected:

```js
const badHash = ethers.utils.keccak256(
  ethers.utils.toUtf8Bytes("Alice Doe|Example University|B.S. Computer Science|2026|4.00")
)
await contract.connect(employer).verifyCredential(
  credentialId,
  university.address,
  student.address,
  badHash
)
```

The first `verifyCredential` call should return `true`; the second should
return `false`.

You can check the demo console result in demo.pdf

---

## Project layout

```
.
├── contracts/
│   └── TranscriptVerification.sol
├── scripts/
│   └── demo.js
├── hardhat.config.js
└── package.json
```

## License

MIT
