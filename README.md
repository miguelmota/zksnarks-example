# zkSNARKs Example

> An example of how generate zero-knowledge proofs and verify using an Ethereum smart contract.

## Tutorial

- **Install [circom](https://github.com/iden3/circom) and [snarkjs](https://github.com/iden3/snarkjs):**

    ```bash
    npm install -g circom
    npm install -g snarkjs
    ```

    NOTE: for this demo I used `circom@0.0.24` and `snarkjs@0.1.11`

  - Circom is a language for writing arithmetic circuits that can be used in zero-knowledge proofs. Circom stands for circuit compiler.
  - Snarkjs is a JavaScript implementation of zkSNARK schemes which is used to generate proofs given circuits generated with circom.

- **Create a circuit:**

    We'll create a circuit that tries to prove we are able to factor a number.

    - The circuit will have 2 private inputs and 1 output.
    - The output signal `c` must be the value of signal `a` times signal `b`.
    - We instantiate the circuit and assign it to the main component. The component `main` must always exist.

    `circuit.circom`

    ```circom
    template Multiplier() {
      signal private input a;
      signal private input b;
      signal output c;
      c <== a*b;
    }

    component main = Multiplier();
    ```

    Ths only thing the circuit enforces is that `c` equals `a` times `b` with the statement `c <== a*b` which defines the constraint.

- **Compile the circuit with circom:**

    ```bash
    circom factor/circuit.circom -o circuit.json
    ```

    This will output `circuit.json`

- **Run setup for the circuit:**

    Running the setup method will instantiate key files we'll need:

    ```bash
    snarkjs setup --circuit=circuit.json
    ```

    The setup will generate a proving key and a verification key, `proving_key.json` and `verification_key.json` respectively.

- **Calculate a witness:**

    We'll create a file containing the input parameters.

    - Provide a file with the inputs and it will execute the circuit.
    - Snarkjs will calculate the intermediate signals and the output.
    - The calculated set of signals is the witness.

    For this example, we'll be proving we can factor `56` without revealing the two factor numbers.

    We'll prove that we know that `7` and `8` multiply to `56`:

    `input.json`

    ```json
    {
      "a": 7,
      "b": 8
    }
    ```

    - The zero knowledge proofs prove that we know a set of signals that match the constraints.
    - These signals are the witness.
    - The witness doesn't reveal any of the signals except the public inputs and the outputs.

    We can calculate witness now:

    ```bash
    snarkjs calculatewitness --circuit=circuit.circom --input=input.json
    ```

    This generates `witness.json` with all the signals.

- **Generate proof:**

    We'll use the proving key and witness to generate a proof that we the inputs that factor into `56`:

    ```bash
    snarkjs proof --witness=witness.json --provingkey=proving_key.json
    ```

    This generates `proof.json` and `public.json` which is the values of the public inputs and outputs. In this case it's just the output because our inputs were private.

    ```json
    [
     "56"
    ]
    ```

- **Verify proof:**

    We can verify a proof using snarkjs:

    ```bash
    $ snarkjs verify --proof=proof.json --verificationkey=verification_key.json --public=public.json
    OK
    ```

    This will output `OK` if correct or `INVALID` if incorrect.

- **Smart contract to verify proofs:**

    snarkjs provides a method to generate a contract to verify proofs onchain:

    ```bash
    snarkjs generateverifier --verificationkey=verification_key.json
    ```

    This outputs `verifier.sol`

    The contract method we'll be using is `verifyProof`:

    ```bash
    function verifyProof(
            uint[2] a,
            uint[2] a_p,
            uint[2][2] b,
            uint[2] b_p,
            uint[2] c,
            uint[2] c_p,
            uint[2] h,
            uint[2] k,
            uint[1] input
        ) view public returns (bool r) {
        Proof memory proof;
        proof.A = Pairing.G1Point(a[0], a[1]);
        proof.A_p = Pairing.G1Point(a_p[0], a_p[1]);
        proof.B = Pairing.G2Point([b[0][0], b[0][1]], [b[1][0], b[1][1]]);
        proof.B_p = Pairing.G1Point(b_p[0], b_p[1]);
        proof.C = Pairing.G1Point(c[0], c[1]);
        proof.C_p = Pairing.G1Point(c_p[0], c_p[1]);
        proof.H = Pairing.G1Point(h[0], h[1]);
        proof.K = Pairing.G1Point(k[0], k[1]);
        uint[] memory inputValues = new uint[](input.length);
        for(uint i = 0; i < input.length; i++){
            inputValues[i] = input[i];
        }
        if (verify(inputValues, proof) == 0) {
            return true;
        } else {
            return false;
        }
    }
    ```

    As you can there is a lot going on. Snarkjs conveniently provides a method to generate the transaction input parameters.

    ```bash
    snarkjs generatecall --proof=proof.json --public=public.json
    ```

    The output will look similar to this:

    ```json
    ["0x225074f4c52d6abfbe39070488e9ad4a614c17e954f6dd1f95df298a0eef0a6c", "0x16c7e4b0c01a120fe48c8eb2339a7209e77796328a61e56547b73567016fd75e"],["0x250a2c4eb6c8d434b3fa3c408c1a3ab4a2db6d63c7ba917c9849083c86812832", "0x2f9c617819fec52b3417745eee59319eb56d846b5e428cb81d90a9a1f09beaf4"],[["0x19ab9c29a7e580507cc3054f36f12506d429112faa75e751ddc4c6237344ab1a", "0x099afd70d77234c35b15112b7a9352f6ec2b3030408125b66654631de21c078e"],["0x2d67175e322ab6e6a98921e29c94cdd1ab07db28d62de7e104199361e5092851", "0x0bdc9fd000cdbc79b344fb307c1ea679b2a14f1756a47cbccefe6349265fdd9d"]],["0x2f17a8a96f4f1a37dcf99fceddbb942e94a3a83547b80f4061eaa295d969dccc", "0x086cc7c64c083849eba6161be31f794b359e735e83c25a7ee9475d9943d9b8f5"],["0x26f172ffa1f027799863c43c640ccb632fc664c918138fea026127bdf07c6c0a", "0x0e0dd693317beda6b8bf1ac9024af15a35bee071cc5bdcb2784926df18ba3fb0"],["0x1c23be59028e8095a430b2e5b3768303ec5163a49e2a9b4da7f3a6b30b78a5f1", "0x0318a6a3bd9a7c36e35fc44030fda9dd594965b1cba90c942211028c3f308fb6"],["0x1f84acf96a6d7b112c097feb438dadd8bc56c59f5e466e7d17e33560df87c893", "0x257a236d53fb9a129d5e8239d592bc3173f99ab378df5f5c5a00d4e3fb6983b2"],["0x1c55a602b2237a439ccc899ae6a49e1d08ca66300283500188e5823d246a9321", "0x1c297b898c386684bc32e504317dbfdb09971227d413541f74659fcc2cd8a9a5"],["0x0000000000000000000000000000000000000000000000000000000000000038"]
    ```

    The verify function returns `true` if the public inputs satisfy the proof.

- **Compiling verifier contract:**

    Head over to the [Ethereum Remix IDE](https://remix.ethereum.org) and paste in the smart contract.

    - Compile the smart contract using the correct solidity compiler version.

    <img width="900" alt="" src="https://user-images.githubusercontent.com/168240/53067171-e8a70500-3487-11e9-9cca-cfb940523725.png">

- **Deploying verifier contract:**

    Use the [MetaMask](https://metamask.io/) extension and connect to a testnet. I chose the Rinkeby testnet for this example.

    - Select *Injected Web3* as the environment.
    - Select the *Verifier* contract and click *Deploy*.

    <img width="900" alt="" src="https://user-images.githubusercontent.com/168240/53067181-f2c90380-3487-11e9-8e2f-2d9dde9d0e1a.png">

    Deployment transaction: [https://rinkeby.etherscan.io/tx/0x6b7c99bc9b6972e11c3a2918d8fe00b40f0036ebf18d28918a20a80b8d306947](https://rinkeby.etherscan.io/tx/0x6b7c99bc9b6972e11c3a2918d8fe00b40f0036ebf18d28918a20a80b8d306947)

- **Verify proofs using the deployed contract**:

    Use the `generatecall` helper like we did in a previous step and paste it in to the *verifyProof* input field under *Deployed Contracts*:

    <img width="900" alt="" src="https://user-images.githubusercontent.com/168240/53067186-f8264e00-3487-11e9-903a-04c010e28a35.png">

    This is a *pure* function meaning it doesn't perform any writes. You'll get back a boolean response as you can see in the bottom right below the method input field.

## Additional

- To show general statistics of this circuit, you can run:

  ```bash
  snarkjs printconstraints --circuit=circuit.json
  ```

- Print the constraints of the circuit by running:

  ```bash
  snarkjs info --circuit=circuit.json
  ```

## Resources

- [circom and snarkjs tutorial](https://iden3.io/blog/circom-and-snarkjs-tutorial2.html)

## License

[MIT](LICENSE)
