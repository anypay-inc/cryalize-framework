# Cryalize Framework

## Motivation

Comply with regulations for security tokens with efficiency.

## Requirements

- MUST comply with the regulations across jurisdictions.
    - WANT to have a way of sharing logic or data when agreed among players such as primary issuer, KYC vendor, secondary exchange, broker dealer, transfer agent, etc.
- MAY require to associate off-chain data with on-chain data registry.
- MAY require to upgrade the implementation.

(Ones below are quoted from [EIP 1411](https://github.com/ethereum/EIPs/issues/1411))
- MUST have a standard interface to query if a transfer would be successful and return a reason for failure. 
- MUST be able to perform forced transfer for legal action or fund recovery. 
- MUST emit standard events for issuance and redemption.
- SHOULD be ERC20 and ERC777 compatible.

## Architecture

- **Token** *has one* **Token Implimentation**

This is to make the functions upgradable. [ERC1538](https://github.com/ethereum/EIPs/issues/1538) may be applicable here.

- **Token Implimentation** *refers to many* **Regulator Services**

**Regulator Service** is responsible for logics to cover all regulations for each type of token.

- **Regulator Service** *referes to a* **Registry**

**Registry** is a kind of key-value store containing investors' information. Optionally a uri and a document hash of off-chain data can be associated. Only simple functions to add/remove/modify the data should be provided. Each key has a set of Ethereum addresses authorized to change the value.

## Specification

### Token

#### Transfer Validity

##### canSend

This function is supposed to simply call *canSend* of the implementation contract.

(Quoted from [EIP 1411](https://github.com/ethereum/EIPs/issues/1411), except the part that the second parameter of the response is an array)
Transfers of securities may fail for a number of reasons.
The function will return both a ESC (Ethereum Status Code) following the EIP-1066 standard, and additional bytes32 parameters that can be used to define application specific reason codes with additional details for an invalid transfer.

##### Notes

Although the transferability depends on the *from* address and the amount as well as the *to* address in the end, there are some cases that an application wants to figure out if an address is qualified to hold any amount of the token before the user tries to find a counterpart to transfer it with.

We thought about adding another function which takes single parameter of the address. However, we eventually decided not to do that due to the complexity caused by that on the overall framework. For this usecase, you are suggested to call *canSend(<An address which you're sure that has some balances and is qualified to transfer>, _address, 1)*.


#### interface

```solidity
interface IST is IERC777 {
    function canSend(address _from, address _to, uint256 _amount) external view returns (byte, bytes32[]);
}
```

### Token Implementation

#### example

```solidity
contract STImplSample {

    IRegulatorService[] public regulatorServices;
    
    constructor(IRegulatorService[] _regulatorServices) public {
        regulatorServices = _regulatorServices;
    }

    function canSend(address _from, address _to, uint256 _amount) external view returns (byte, bytes32[]) {
        bytes32[] additionalInvalidCodes;
        for (uint256 i = 0; i < regulatorServices.length; i++) {
            additionalInvalidCodes.push(regulatorServices[i].getInvalidReasonCodes(_from, _to, _amount));
        }
        
        // FIXME return more appropriate return code
        if (additionalInvalideCodes.length == 0) {
            return (0x01, additionalInvalidCodes);
        } else {
            return (0x00, additionalInvalidCodes);
        }
    }
    
    function getRegulatorServices() external view returns (IRegulatorService[]) {
        return regulatorServices;
    }
    
}
```

### Regulator Service

#### getInvalidReasonCodes

Check if the transfer is compliant with the regulation of the token and return a set of invalid reason codes. Return empty array if valid.

#### interface

```solidity
interface ISTRegulatorService {
    function getInvalidReasonCodes(address _from, address _to, uint256 _amount) external view returns (bytes32[]);
}
```

#### example

```solidity
contract TheRegDRegulatorService is ISTRegulatorService {
    
    IRegistry public registry;
    
    address owner;
    mapping(address => bool) users;
    
    modifier onlyOwner {
        require(msg.sender == owner);
        _;
    }
    
    modifier isAuthorized {
        require(users[msg.sender]);
        _;
    }
    
    constructor(IRegistry[] _registry) public {
        registry = _registry;
        owner = msg.sender;
        users[msg.sender] = true;
    }
    
    function authorizeUser(address _address) onlyOwner {
        users[_address] = true;
    }
    
    /*
     * @dev Check if the transfer is compliant with RegD and return set of invalid reason codes. Return empty array if valid.
     */
    function getInvalidReasonCodes(address _from, address _to, uint256 _amount) external view isAuthorized returns (bytes32[]) {
        bytes32[] additionalInvalidCodes;
        var (toAccreditedUsaValue, toAccreditedUsaUri, toAccreditedUsaDocumentHash) = registry.getAttribute(_to, "accredited_usa");
        if (toAccreditedUsaValue != "true") {
            additionalInvalidCodes.push(0x123);
        }
        
        // TODO check other factors
        
        return additionalInvalidCodes;
    }
    
}
```

### Registry

**Registry** stores investors' information. Off-chain document can be associated with the data by using uri and hash of the data.

```solidity
contract STRegistrySample {
    
    struct attributeValue {
        bytes32 value;
        string uri; // optional in some cases
        bytes32 documentHash; // optional in some cases
    }
    
    address owner;
    
    /*
     * @dev: Example:
     * @dev: 0x12.. => ["residency" => attributeValue("Japan", "https://...." (where KYC document is stored), "7dbd0c39...")]
     */
    mapping(address => mapping(bytes32 => attributeValue)) attributes;
    mapping(bytes32 => bool) availableAttributes;
    mapping(address => mapping(bytes32 => bool)) authorizedAttributes;
    
    modifier onlyOwner {
        require(msg.sender == owner);
        _;
    }
    
    modifier isAuthorized(bytes32 _attribute) {
        require(authorizedAttributes[msg.sender][_attribute]);
        _;
    }
    
    constructor() {
        owner = msg.sender;
    }
    
    function authorizeAttribute(address _address, bytes32 _attribute) onlyOwner {
        authorizedAttributes[_address][_attribute] = true;
    }
    
    function unauthorizeAttribute(address _address, bytes32 _attribute) onlyOwner {
        authorizedAttributes[_address][_attribute] = false;
    }
    
    function getAttribute(address _address, bytes32 _attribute) public view isAuthorized(_attribute) returns(bytes32, string, bytes32) {
        attributeValue attr = attributes[_address][_attribute];
        return (attr.value, attr.uri, attr.documentHash);
    }
    
    function setAttribute(address _address, bytes32 _attribute, bytes32 _value, string _uri, bytes32 _documentHash) public isAuthorized(_attribute) returns (bool) {
        attributes[_address][_attribute] = attributeValue(_value, _uri, _documentHash);
        return true;
    }
    
}
```
