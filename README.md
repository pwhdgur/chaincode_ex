■ 참조 사이트 : https://hyperledger-fabric.readthedocs.io/en/latest/chaincode.html

■ Chaincode for Developers

< 1. Chaincode API >
- Init 메소드는 Chaincode가 Instantiate나 Upgrade와 같은 트랜잭션 메소드를 어플리케이션 State에 관한 초기화를 포함하여 Chaincode가 필요한 초기화를 위해서 사용됨
- Invoke 메소드는 Invoke 트랜잭션이 트랜잭션의 제안을 받았을 때 발생

< 2. Simple Asset Chaincode >
2.1 code의 디렉토리 설정하기
$ mkdir -p $GOPATH/src/sacc && cd $GOPATH/src/sacc
$ nano sacc.go

2.2 Housekeeping
- Simple Asset을 Chaincode Shim Function으로서 추가하는 코드 (sacc.go)
- Chaincode shim package와 peer protobuf package를 import

// Code Example
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

// SimpleAsset implements a simple chaincode to manage an asset
type SimpleAsset struct {
}

2.3 Chaincode 초기화
2.3.1 Init 함수를 실행

// Code Example
// Init is called during chaincode instantiation to initialize any data.
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {

}

2.3.2 Init 호출에서 ChaincodeStubInterface.GetStringArgs 메소드를 통해 매개변수를 돌려받습니다. 그리고 유효성을 체크함

// Code Example
// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data, so be careful to avoid a scenario where you
// inadvertently clobber your ledger's data!
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // Get the args from the transaction proposal
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }
}

2.3.3 원장에 초기 State를 저장 및 ChaincodeStubInterface.Putstate를 불러내어 Key와 Value를 인자로 전송

// Code Example
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
   // Get the args from the transaction proposal
   args := stub.GetStringArgs()
   if len(args) != 2 {
     return shim.Error("Incorrect arguments. Expecting a key and a value")
   }
 
   // Set up any variables or assets here by calling stub.PutState()
 
   // We store the key and the value on the ledger
   err := stub.PutState(args[0], []byte(args[1]))
   if err != nil {
     return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
   }
   return shim.Success(nil)
}

2.4 Chaincode 호출

2.4.1 Invoke 함수의 서명을 추가

// Code Example
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The 'set'
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {

}

2.4.2 set & get 해당 함수들을 통해서 asset의 값을 set할 수 있고, 또한 현재 State를 리턴 받음.

// Code Example
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

}


2.4.3 Chaincode 어플리케이션 함수를 적절한 Shim.success와 Shim.error 함수를 gRPC protobuf 메시지 형태로 응답하며 시리얼라이즈 출력을 하면서 호출

// Code Example
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else {
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    // Return the result as success payload
    return shim.Success([]byte(result))
}

2.4.4 Chaincode 어플리케이션 실행하기
- Invoke 함수를 통해서 두개의 함수(set, get)를 가진 Chaincode 어플리케이션을 실행
- 원장에 접근하기 위해선, ChaincodeStubInterface.Putstate와 ChaincodeStubInterface.Getstate 함수를 Chaincode shim API 사용

// Code Example
// Set stores the asset (both key and value) on the ledger. If the key exists,
// it will override the value with the new one
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// Get returns the value of the specified asset key
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}

2.4.5 모든 값을 가져오기
- main 함수 (shim.start 사용)

// 전체 Code Example
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

// SimpleAsset implements a simple chaincode to manage an asset
type SimpleAsset struct {
}

// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data.
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
    // Get the args from the transaction proposal
    args := stub.GetStringArgs()
    if len(args) != 2 {
            return shim.Error("Incorrect arguments. Expecting a key and a value")
    }

    // Set up any variables or assets here by calling stub.PutState()

    // We store the key and the value on the ledger
    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
    }
    return shim.Success(nil)
}

// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else { // assume 'get' even if fn is nil
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    // Return the result as success payload
    return shim.Success([]byte(result))
}

// Set stores the asset (both key and value) on the ledger. If the key exists,
// it will override the value with the new one
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// Get returns the value of the specified asset key
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}

// main function starts up the chaincode in the container during instantiate
func main() {
    if err := shim.Start(new(SimpleAsset)); err != nil {
            fmt.Printf("Error starting SimpleAsset chaincode: %s", err)
    }
}

2.5 Chaincode 만들기
- Chaincode를 컴파일
$ go get -u github.com/hyperledger/fabric-chaincode-go
$ go build

< 3. Hyperledger Fabric Samples 설치 및 환경 준비 >

open three terminals 준비
$ cd chaincode-docker-devmode

3.1 Terminal 1 - Start the network
- singleSampleMSPSolo의 주문자 정보와 함께 네트워크를 시작
- 피어를 개발자 모드로 실행 ( dev mode does not work with TLS. )
$ docker-compose -f docker-compose-simple.yaml up

3.2 Terminal 2 - Build & start the chaincode
- Chaincode가 피어와 함께 실행되었고, Chaincode 로그는 성공적인 등록을 피어와 마쳤을 것을 확인함.
- 이 단계에서 Chaincode가 어떤 채널과도 연관되지 않음을 주의.

$ docker exec -it chaincode bash
(Result) => root@d2629980e76b:/opt/gopath/src/chaincode#

$ cd sacc
$ go build
$ CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 ./sacc

3.3 Terminal 3 - Use the chaincode
- --peer-chaincodedev모드에 있더라도, Chaincode를 설치하셔야 chaincode 라이프 싸이클을 정상적으로 확인할 수 있습니다.
- install / instantiate 과정
- invoke : change the value of “a” to “20”.
- query : see a value of 20.

$ docker exec -it cli bash

$ peer chaincode install -p chaincodedev/chaincode/sacc -n mycc -v 0
$ peer chaincode instantiate -n mycc -v 0 -c '{"Args":["a","10"]}' -C myc

$ peer chaincode invoke -n mycc -c '{"Args":["set", "a", "20"]}' -C myc
$ peer chaincode query -n mycc -c '{"Args":["query","a"]}' -C myc


< 참고 명령어 : Clean Up >
cd /fabric-samples/first-network
./byfn.sh down
docker rm $(docker ps -aq)
docker rmi $(docker images dev-* -q)
docker network prune

< 강제로 삭제시 -f >
docker rm -f ID
