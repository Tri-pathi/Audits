### Use assembly to validate `msg.sender`

We can use assembly to efficiently validate msg.sender for the `onlyBuyer`, `onlyBuyerOrSeller` and `onlyArbiter` modifier with the least amount of opcodes necessary. Additonally , we can use `xor()`instead of `iszero(eq())` saving 3 gas.
We can also potentially save gas on the unhappy path by using `scratch space` to store the error selector, potentially avoiding memory expansion costs

```solidity

File : src/Escrow.sol

 /// @dev Throws if called by any account other than buyer.
    modifier onlyBuyer() {
        if (msg.sender != i_buyer) {
            revert Escrow__OnlyBuyer();
        }
        _;
    }

    /// @dev Throws if called by any account other than buyer or seller.
    modifier onlyBuyerOrSeller() {
        if (msg.sender != i_buyer && msg.sender != i_seller) {
            revert Escrow__OnlyBuyerOrSeller();
        }
        _;
    }

    /// @dev Throws if called by any account other than arbiter.
    modifier onlyArbiter() {
        if (msg.sender != i_arbiter) {
            revert Escrow__OnlyArbiter();
        }
        _;
    }

```

```solidity

modifier onlyBuyer(){
    address buyer= i_buyer;
    assembly {
            if xor(caller(), buyer) {
                //bytes4(keccak256("Escrow__OnlyBuyer()"))
                mstore(0x00, 0xc5e79f0e)
                revert(0x1c, 0x04)
            }
        }
}

modifier onlyBuyerOrSeller(){
    address buyer= i_buyer;
    address seller=i_seller;
    assembly {
            if or (eq(caller(), buyer)),eq (eq(caller(), seller)) {

            }else{
                //bytes4(keccak256("Escrow__OnlyBuyerOrSeller()"))
                mstore(0x00, : 0xc3ff666b)
                revert(0x1c, 0x04)
            }
        }
}
modifier onlyArbiter(){
    address arbiter= i_arbiter;
    assembly {
            if xor(caller(), arbiter){
                //bytes4(keccak256("Escrow__OnlyArbiter()"))
                mstore(0x00,  0xada78556)
                revert(0x1c, 0x04)
            }
        }
}


```
