# Groth16 from Circom to Lambdaworks tutorial

In this small tutorial, we will perform all the steps needed to prove a computation using Groth16.

As said above, our goal is to prove a computation. In this case we want to prove the Fibonacci sequence.

If you don´t know what Groth16 does, you can read our [post](https://blog.lambdaclass.com/groth16/).

Let's begin by creating the circuit and the input for the program using [Circom2](https://docs.circom.io/getting-started/installation/). 


> [!IMPORTANT]  
> This tutorial will work within `lambdaworks/provers/groth16` to make it easier to follow along. If you prefer, you can create a new rust project, but you will need to import all the crates from the library.

As we want to cover all the steps from creating a program to prove it using Groth 16, the first thing to do is to create the folder for our program.

Create a file named `fibonacci` inside `test_files`.This is where our program will be.

The directory should look like this:
```
provers/
└── groth16/
    ├── arkworks-adapter/
    ├── circom-adapter/
    ├── src/
    │   ├── integration_tests.rs
    │   ├── lib.rs
    │   └── README.md
    ├── test_files/
    │   ├── fibonacci/ (or any other name)
    │   ├── poseidon/
    │   └── vitalik_example/
    └── Cargo.toml
```

### Understanding the Fibonacci Circuit
The Fibonacci sequence is defined as follows:

$F(0) = a \quad \text{(starting value 1)}$

$F(1) = b \quad \text{(starting value 2)}$

$F(n) = F(n-1) + F(n-2) \quad \text{for } n \geq 2$ 

In our circuit, we start with two initial numbers, 
a and b, and compute the nth Fibonacci number. The circuit checks if the output is correctly computed based on these inputs

```circom
pragma circom 2.0.0;

// Fibonacci with custom starting numbers
template Fibonacci(n) {
  assert(n >= 2);
  signal input in[2];
  signal output out;

  signal fib[n+1];
  fib[0] <== in[0];
  fib[1] <== in[1];
  for (var i = 2; i <= n; i++) {
    fib[i] <== fib[i-2] + fib[i-1];
  }

  out <== fib[n];
}

// Instantiate Fibonacci as the main circuit with a specific n value
component main = Fibonacci(10);
```
In our case this will be named `fibonacci.circom`.


In this circuit:
`in[0]` and `in[1]` are the initial values a and b

`out` will hold F(10), the 10th number in the Fibonacci Sequence

### Creating inputs for the Circuit

Once we have defined the circuit, the next step is to provide the inputs needed for the proving process. In this case, our inputs consists of the starting numbers for the Fibonacci sequence.

As we want to use F(0) = 1 and F(1) = 1, the input JSON file will look like this;

```json
{
  "in": [1, 1]
}
```

The file should be named `input.json`.

Inside the same directory where those files are run

```bash
circom fibonacci.circom --r1cs --wasm -p bls12381
```
Here
`r1cs` generates the R1CS file
`wasm` generates the WebAssembly code
`-p bls12381` sets the elliptic curve.

That will create a `fibonacci_js` directory and a `fibonacci_r1cs `file

> [!WARNING]
> Do not skip the -p bls12381 flag, as this is the only field supported by the adapter right now. If not specified, the default is bn128 and the proving will not be possible


To compute the `witness` execute

```
node fibonacci_js/generate_witness.js fibonacci_js/fibonacci.wasm input.json witness.wtns
```

As our program inputs are json files we need to export the witness and r1cs files into the same format. To do that run:

```bash
snarkjs wtns export json witness.wtns
```
and
```bash
snarkjs r1cs export json fibonacci.r1cs fibonacci.r1cs.json
```

Now we have `witness.json` and `fibonacci.r1cs.json`

The folder should look like:


```
fibonacci/
├── fibonacci_js/
├── fibonacci.circom
├── fibonacci.r1cs
├── fibonacci.r1cs.json
├── input.json
├── witness.json
└── witness.wtns
```

We only need `fibonacci.r1cs.json` and `witness.json` so, if you want, you can delete the unnecessary files by running:
```
rm -rf fibonacci_js fibonacci.circom fibonacci.r1cs witness.wtns input.json
```

All at once, you can copy-paste the following to the terminal in the same directory as your circuit

```bash
circom fibonacci.circom --r1cs --wasm -p bls12381;
node fibonacci_js/generate_witness.js fibonacci_js/fibonacci.wasm input.json witness.wtns;
snarkjs wtns export json witness.wtns witness.json;
snarkjs r1cs export json fibonacci.r1cs fibonacci.r1cs.json;
rm -rf fibonacci_js fibonacci.circom fibonacci.r1cs witness.wtns input.json; # Delete unnecessary artifacts
```

## Using Lambdaworks Circom Adapter

As mentioned in the blog post, the main goal of Groth16 (or any other prover) is to prove computation.
The function `circom_to_lambda` will take the `fibonacci.r1cs.json` and the `witness.json` data and translate it into the format needed for Lambdaworks.

The `circom_to_lambda` function is responsible for converting the R1CS constraints and witness generated by Circom into a format that Lambdaworks can use to construct a Quadratic Arithmetic Program (QAP). The QAP representation is essential for Groth16 proof generation, as it represents the arithmetic circuit in terms of polynomials.

### How `circom_to_lambda` works

The function `circom_to_lambda` does the following:

1. Parse JSON Data: It parses the R1CS constraints and witness files from JSON format into Rust data structures using serde_json.

2. Build LRO Matrices: It extracts the L,R,O matrices which represent the variables involved in each constraint.

3. Adjust Witness: It adjusts the order of inputs and outputs in the witness to match Lambdaworks's format. Circom and Lambdaworks have different conventions for witness ordering, so this adjustment ensures compatibility.

4. Construct QAP: It uses the L,R,O matrices to build a Quadratic Arithmetic Program (QAP). This QAP is used in the Groth16 proving process.


## Generating the proof with Groth16

After converting the data using `circom_to_lambda`, you can use the resulting QAP and witness to generate a Groth16 proof. This process involves two key steps: creating the proving key and then using it to generate the proof based on the QAP. Here’s a detailed explanation of the process:

We will put all together inside `integration_test`.

Step 1: Read the R1CS and Witness Files.
First, you need to read the R1CS (Rank-1 Constraint System) file and the witness file generated by Circom. 

```rust
let test_dir = format!("{TEST_DIR}/fibonacci");

let (qap, w) = circom_to_lambda(
    &fs::read_to_string(format!("{test_dir}/fibonacci.r1cs.json"))
        .expect("Error reading the R1CS file"),
    &fs::read_to_string(format!("{test_dir}/witness.json"))
        .expect("Error reading the witness file"),
);
```

`test_dir`: Specifies the directory where the R1CS and witness files are stored. This allows for easy organization and reuse of test data.

`circom_to_lambda` : Converts the content of the R1CS and witness files into a format compatible with Lambdaworks. This function parses the JSON content and produces a QAP and a corresponding witness vector.

Step 2: Generate the Proving and Verifying Keys:

```rust
let (pk, vk) = setup(&qap);
```

- Proving Key (pk): Used to create a proof that the computation was carried out correctly.
- Verifying Key (vk): Used to verify the validity of the proof, ensuring that it matches the expected computation.


Step 3: Generate the Proof 

With the proving key in hand, you can generate a proof using the `Prover::prove` function:

```rust
let proof = Prover::prove(&w, &qap, &pk);
```
`Prover::prove`: This function takes the witness, QAP, and proving key to generate a proof that certifies the correctness of the computation represented by the circuit.

Step 4: Verify the Proof
To ensure the proof is valid, use the `verify` function:

```rust 
let accept = verify(&vk, &proof, &w[..qap.num_of_public_inputs]);
```

`verify`: Takes the verifying key, proof, and public inputs to check if the proof is valid. It ensures that the proof matches the expected output of the computation without revealing the private inputs.
Public Inputs: Extracted from the witness slice `&w[..qap.num_of_public_inputs]` to provide the data that is publicly visible during verification.

Step 5: Assert the Verification Result

Finally, check if the verification was successful

```rust
assert!(accept, "Proof verification failed.");
```
This line ensures that the proof verification returns `true`. If the verification fails, the test will produce an error, which helps catch issues during development.


#### Putting altogether
Inside `integration_tests.rs` add

```rust
#[test]
fn fibonacci_verify() {
    // Define the directory containing the R1CS and witness files
    let test_dir = format!("{TEST_DIR}/fibonacci");

    // Step 1: Parse R1CS and witness from JSON files
    let (qap, w) = circom_to_lambda(
        &fs::read_to_string(format!("{test_dir}/fibonacci.r1cs.json"))
            .expect("Error reading the R1CS file"),
        &fs::read_to_string(format!("{test_dir}/witness.json"))
            .expect("Error reading the witness file"),
    );

    // Step 2: Generate the proving and verifying keys using the QAP
    let (pk, vk) = setup(&qap);

    // Step 3: Generate the proof using the proving key and witness
    let proof = Prover::prove(&w, &qap, &pk);

    // Step 4: Verify the proof using the verifying key and public inputs
    let accept = verify(&vk, &proof, &w[..qap.num_of_public_inputs]);

    // Step 5: Assert that the verification is successful
    assert!(accept, "Proof verification failed.");

    println!("Proof verification succeeded. All steps completed.");
}
```
and run:

```rust
cargo test -- --test fibonacci_verify
```
If everything is set up correctly, you should see a success message confirming that the proof has been verified:

```bah
Proof verification succeeded. All steps completed.
```

## Summary

Congratulations! You have successfully set up a Groth16 proof for a Fibonacci circuit using Circom and Lambdaworks. This tutorial walked you through the entire process—from defining the circuit in Circom, generating the R1CS and witness, converting the data for use in Rust, and finally, creating and verifying the proof using Lambdaworks.

