
### Code

- `lib/` contains main PDID procedures including the implementation of
  modified [OPAQUE](https://eprint.iacr.org/2018/163.pdf) and [HMQV](https://eprint.iacr.org/2005/176.pdf) protocols
- `lib/ecc.{c,h}` are from [Easy-ECC](https://github.com/esxgx/easy-ecc). It is
  modified (to be compatible with Intel SGX and its SDK) and extended by
PDID-related functions (prefixed with `pdid_`) 
- `lib/tweetnacl.{c,h}` are from [TweetNaCl](https://tweetnacl.cr.yp.to/) 
- `chaincode/` is the GPM smart contract to be deployed with [Hyperledger Fabric Private Chaincode](https://github.com/hyperledger-labs/fabric-private-chaincode) (FPC)
- `fpc-1.0-gitdiff.patch` is fix for building FPCv1.0 and adding an SGX's ocall getting local system time (used for performance measurements only)


### Local tests
Local (emulation) test should work out of box: `make && ./local_test`
For better performance, you can use [NaCl](https://nacl.cr.yp.to/) instead of TweetNaCl. Install NaCl, change the build path to yours
`export NACL_PATH=../../nacl-20110221/build/Latitude5280`
and compile with the following flags (for x86_64)
`-I${NACL_PATH}/include/amd64 -L${NACL_PATH}/lib/amd64 -lnacl -DWITH_NACL`

The integration test requires FPC deployment.


### Hyperledger deployment

1) Generate `./integration_test` by `make`
1) Install FPC as described [here](https://github.com/hyperledger-labs/fabric-private-chaincode)
(version v1.0.0-rc3 [branch](https://github.com/hyperledger/fabric-private-chaincode/tree/v1.0.0-rc3).)
```
export GOPATH=~/
export FPC_PATH=$GOPATH/src/github.com/hyperledger/fabric-private-chaincode
git clone --recursive https://github.com/hyperledger/fabric-private-chaincode.git $FPC_PATH
cd $FPC_PATH/utils/docker
make pull-dev 
make run-dev (enter container)
cd $FPC_PATH
make (Build Fabric Private Chaincode)
```

2) 
3) Copy `chaincode/` and `lib/` to FPC's `samples/`. Then `cd samples/chaincode && make`.
4) In another terminal, run `./integration_test` and follow its instructions 


