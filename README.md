<img src='img/vera.svg' width='200'>

# Project Verity
Verity is a decentralised and censorship-resistant data storage and distribution
platform on top of which fully georedundant, peer-to-peer-enabled applications
can be built. It leverages unique cube structures, each containing 1kB of data,
which are then synchronized across participating nodes. Different Cube types
offer predefined cryptographic schemes to ensure data integrity and authenticity.
Although light nodes are supported, nodes are encouraged to be operated as full
nodes, replicating all cubes.

As a demo application, this project also creates a decentralized and
censorship-resistant microblogging app application using the Verity platform,
akin to Twitter, Threads or Reddit.

Application developers should check out the [Client Application Guide](clientApplicationGuide.md)

For more in-depth information, see the [technical documentation](doc/verity.md).

## Installing Dependencies ##
```
npm install
```

## Building the demo Microblogging Web App ##
```
npm run webpack
```
Then open `distweb/index.html` in a browser. Or alternatively directly spawn a web server with:
```
npm run webpack serve
```

## Building the Support Node ##
Note that this runs a command line node intended to support the Verity network.
An end-user will typically not need this.

```
npm run build
npm run start
```

## Tests ##

### Node.js Tests ###
Run the main Node.js test suite:
```bash
npm test
```

Run a subset of tests (CI command):
```bash
npm test -- --run test/core/ test/cci/ test/web/controller/ test/app/zw/model/
```

### Browser Tests ###
Run Playwright browser tests:
```bash
# Run all Playwright browser tests
npm run test:playwright

# Interactive test runner with UI
npm run test:playwright:ui
```

