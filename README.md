# zkSNARKs Example

> An example of how generate zero-knowledge proofs and verify using an Ethereum smart contract.

## Tutorial

- **Install [circom](https://github.com/iden3/circom) and [snarkjs](https://github.com/iden3/snarkjs)**

    ```bash
    npm install -g circom
    npm install -g snarkjs
    ```

    NOTE: for this demo I used `circom@0.0.24` and `snarkjs@0.1.11`

  - Circom is a language for writing arithmetic circuits that can be used in zero-knowledge proofs. Circom stands for circuit compiler.
  - Snarkjs is a JavaScript implementation of zkSNARK schemes which is used to generate proofs given circuits generated with circom.

- **Create a circuit that tries to prove we are able to factor a number.**

    - circuit will have 2 private inputs and 1 output.
    - the output signal must be the value of signal a time signal b.
    - instantiate the circuit. The component `main` must always exist.

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

- **Compile the circuit with circom:**

    ```bash
    circom factor/circuit.circom -o circuit.json
    ```

    it outputed `circuit.json`

- **Run setup for circuit:**

    ```bash
    snarkjs setup --circuit=circuit.json
    ```

    The setup will generate a proving key and a verification key, `proving_key.json` and `verification_key.json` respectively.

- **Calculate a witness:**

    - provide a file with the inputs and it will execute the circuit.
    - snarkjs will calculate the intermediate signals and the output
    - the calculate set of signals is the witness

    - the zero knowledge proofs prove that we know  a set of signals that match the constraints.
    - this signals are the witness.
    - this doesn't reveal any of the signals expect the public inputs and the outputs

    example, proving we can factor `56` without revealing the two factor numbers.

    we'll prove that we know `7` and `8` multiply to `56`:

    `input.json`

    ```json
    {
      "a": 7,
      "b": 8
    }
    ```

    We can calculate witness now:

    ```bash
    snarkjs calculatewitness --circuit=circuit.circom --input=input.json
    ```

    generates `witness.json` with all the signals.

- **Generate proof:**

    we'll use the proving_key and witness to generate a proof

    ```bash
    snarkjs proof --witness=witness.json --provingkey=proving_key.json
    ```

    generates proof and `public.json` which is the values of the public inputs and outputs

- **Verify proof:**

```bash
$ snarkjs verify --proof=proof.json
OK
```

This will output `OK` if correct or `INVALID` if incorrect.

- **Smart contract to verify proofs:**

    snarkjs even provides function to generate a contract to verify proofs onchain

    ```bash
    snarkjs generateverifier --verificationkey=verification_key.json
    ```

    outputs `verifier.sol`

    the contract function name to verify is `verifyProof` and snarkjs conveniently provides a method to generate the transaction input parameters.

    ```bash
    snarkjs generatecall --proof=proof.json --public=public.json
    ```

    ```json
    ["0x225074f4c52d6abfbe39070488e9ad4a614c17e954f6dd1f95df298a0eef0a6c", "0x16c7e4b0c01a120fe48c8eb2339a7209e77796328a61e56547b73567016fd75e"],["0x250a2c4eb6c8d434b3fa3c408c1a3ab4a2db6d63c7ba917c9849083c86812832", "0x2f9c617819fec52b3417745eee59319eb56d846b5e428cb81d90a9a1f09beaf4"],[["0x19ab9c29a7e580507cc3054f36f12506d429112faa75e751ddc4c6237344ab1a", "0x099afd70d77234c35b15112b7a9352f6ec2b3030408125b66654631de21c078e"],["0x2d67175e322ab6e6a98921e29c94cdd1ab07db28d62de7e104199361e5092851", "0x0bdc9fd000cdbc79b344fb307c1ea679b2a14f1756a47cbccefe6349265fdd9d"]],["0x2f17a8a96f4f1a37dcf99fceddbb942e94a3a83547b80f4061eaa295d969dccc", "0x086cc7c64c083849eba6161be31f794b359e735e83c25a7ee9475d9943d9b8f5"],["0x26f172ffa1f027799863c43c640ccb632fc664c918138fea026127bdf07c6c0a", "0x0e0dd693317beda6b8bf1ac9024af15a35bee071cc5bdcb2784926df18ba3fb0"],["0x1c23be59028e8095a430b2e5b3768303ec5163a49e2a9b4da7f3a6b30b78a5f1", "0x0318a6a3bd9a7c36e35fc44030fda9dd594965b1cba90c942211028c3f308fb6"],["0x1f84acf96a6d7b112c097feb438dadd8bc56c59f5e466e7d17e33560df87c893", "0x257a236d53fb9a129d5e8239d592bc3173f99ab378df5f5c5a00d4e3fb6983b2"],["0x1c55a602b2237a439ccc899ae6a49e1d08ca66300283500188e5823d246a9321", "0x1c297b898c386684bc32e504317dbfdb09971227d413541f74659fcc2cd8a9a5"],["0x0000000000000000000000000000000000000000000000000000000000000038"]
    ```

The function returns true if the public inputs satisfy the proof.

- **Deploying smart contract:**

    Head over to []() and paste the smart contract:

    <img width="900" alt="" src="https://user-images.githubusercontent.com/168240/53067171-e8a70500-3487-11e9-9cca-cfb940523725.png">

    Deploy the smart contract:

    <img width="900" alt="" src="https://user-images.githubusercontent.com/168240/53067181-f2c90380-3487-11e9-8e2f-2d9dde9d0e1a.png">

    Verify the proof:

    <img width="900" alt="" src="https://user-images.githubusercontent.com/168240/53067186-f8264e00-3487-11e9-903a-04c010e28a35.png">

## Additional

- To show general statistics of this circuit, you can run:

  ```bash
  snarkjs printconstraints -c circuit.json
  ```

- You can also print the constraints of the circuit by running:

  ```bash
  snarkjs info -c circuit.json
  ```

## Resources

- [circom and snarkjs tutorial](https://iden3.io/blog/circom-and-snarkjs-tutorial2.html)

## License

[MIT](LICENSE)
