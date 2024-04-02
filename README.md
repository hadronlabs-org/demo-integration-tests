# Demo integration tests
It is demo repository with sample contract and integration tests for this contract. The testing is based on top of [Cosmopark](https://github.com/neutron-org/cosmopark/) and [Contracts2ts](https://github.com/neutron-org/contracts2ts). 

## Cosmopark

Cosmopark – is a tool that allows to run multiple networks on the same machine. Under the hood it uses docker containers and require docker images for a network you want to run. It can spin up `Hermes Relayer` and `Neutron Query Relayer` for a deployment if required.

## Contracts2ts

Contracts2ts – is a tool that allows to generate typescript clients for set of contracts. It uses json generated schemas from contracts with `write_api` method.

## How to use

1. Clone the [repository](https://github.com/hadronlabs-org/demo-integration-tests)
2. Place your own contracts source code in the `contracts` folder. The `pump` contract is just an example which can be removed
3. Run `make schema` to generate json schemas for your contracts
4. `make build` - Build your contracts
5. `cd integration-tests`
6. `yarn`
7. `yarn build-images` - Build docker images for the networks used
7. `yarn build-ts-client` - Build TS client for your contracts
8. Implement you own tests in the `src/testcases` folder. The `pump` test files are examples which can be removed
9. `yarn test`

## What's inside the tests (`integration_tests` folder)

`src/testSuite.ts` contains configuration of the networks used with defined network params and docker image names.

`src/testcases` folder contains the tests for the contracts. Each test is a separate file with a set of tests for a contract. Please check the existing tests to understand how to write your own.

`src/vite.config.ts` contains of the configuration for the tests.

## Environment variables

`MAX_THREADS` - maximum threads to run tests in parallel

## How to write your own tests

Let's consider you have written your contract and the contract is located in the `contracts` folder. The contract has a `write_api` method which generates a json schema for the contract. The schema is used to generate a TS client for the contract.

### Prepare TS client for your contracts

You can see `build-ts-client` script in the `package.json` file. It generates TS client for your contracts. The generated client is compiled ton the `src/contracts` folder.

### Docker images

Depending on your contracts you may need to build docker images for the networks. You can see `build-images` script in the `package.json` file and `dockerfiles` folder. 

#### build.sh

Every network in `dockefiles` folder has `build.sh` script. It should contain the build steps for the docker image. Also there you can differ the build steps for the `CI` and `local` environments.

#### options.json

Every network in `dockefiles` folder may have optional `options.json` file. There you can redefine the default network params.
Here is full list of possible params:
```js
{
    "commands": {
        "addGenesisAccount": "genesis add-genesis-account",
        "gentx": "genesis gentx",
        "collectGenTx": "genesis collect-gentxs"
    },
    "genesisOpts": {
        "app_state.slashing.params.downtime_jail_duration": "10s",
        ...
    },
    "upload": ["./artifacts/scripts/init-gaia.sh", ...],
    "postUpload": ["/opt/init-gaia.sh > /opt/init-gaia.log 2>&1", ...],
    "denom": "stake",
    "prefix": "cosmos",
    "binary": "gaiad"
}
```

### Networks configuration

You can see `src/testSuite.ts` file. There you can define the networks configuration. The `networkConfigs` object contains the configuration of the networks used in the tests. Here you can define the network params, docker image names, and the deployment of the contracts. Also you can define the `relayers` for the networks.

### Tests
#### BeforeAll

In the `beforeAll` method you can see the configuration of the networks and relayers.
You can do it this way:
```ts
const context: {
    park?: Cosmopark;
};
beforeAll(async (t) => {
    context.park = await setupPark(
        t, // test context, is used to generate container names based on the test file name
        ['neutron', 'gaia'], // networks to run
        {
            gaia: {
                genesis_opts: {
                    'app_state.staking.params.unbonding_time': '20s'
                }
            }
        }, // custom networks configuration
        { 
            neutron: true, // may be boolean
            hermes: {
                config: {
                    'chains.1.trusting_period': '2m0s',
                    'chains.1.unbonding_period': '20s'
                }
            } // or object with custom configuration
         }, // relayers to run
    );
});
```
#### AfterAll
In the `afterAll` method you can see the Cosmopark teardown.
```ts
afterAll(async (t) => {
    await context.park?.stop();
});
```
#### Tests
In the `src/testcases` folder you can see example tests for the example contracts. At first you setup wallets:
```ts
context.wallet = await DirectSecp256k1HdWallet.fromMnemonic(
    context.park.config.wallets.demowallet1.mnemonic,
    {
    prefix: 'neutron',
    },
);
```
Then you create a client for the contract and upload the contract:
```ts
context.client = await SigningCosmWasmClient.connectWithSigner(
    `http://127.0.0.1:${context.park.ports.neutron.rpc}`,
    context.wallet,
    {
    gasPrice: GasPrice.fromString('0.025untrn'),
    },
);

const res = await client.upload(
    account.address,
    fs.readFileSync(join(__dirname, '../../../artifacts/example_pump.wasm')),
    1.5,
);

```

After that you can instantiate the contract and create TS client
```ts
const instantiateRes = await ExamplePump.Client.instantiate(
    client,
    account.address,
    res.codeId,
    {
        connection_id: 'connection-0',
        dest_address: neutronSecondUserAddress,
        dest_channel: 'channel-0',
        dest_port: 'transfer',
        ibc_fees: {
        timeout_fee: '10000',
        ack_fee: '10000',
        recv_fee: '0',
        register_fee: '1000000',
        },
        local_denom: 'untrn',
        refundee: neutronSecondUserAddress,
        timeout: {
        local: 100,
        remote: 100,
        },
        owner: account.address,
    },
    'label',
    [],
    'auto',
);
context.contractAddress = instantiateRes.contractAddress;
context.contractClient = new ExamplePump.Client(
    client,
    context.contractAddress,
);
```

Now you are ready to interact with the contract:
```ts
it('register ICA', async () => {
    const { contractClient, neutronUserAddress } = context;
    const res = await contractClient.registerICA(
        neutronUserAddress,
        1.5,
        undefined,
        [
        {
            amount: '1000000',
            denom: 'untrn',
        },
        ],
    );
    expect(res).toBeTruthy();
    expect(res.transactionHash).toHaveLength(64);
});
```