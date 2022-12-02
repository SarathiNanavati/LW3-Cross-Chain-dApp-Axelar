# LW3 Axelar cross-chain dApp examples

## Requested Links/Details

-   DistributionExecutable Contract Address on Polygon Testnet : [0x57A757ceCaF2E8Eb8BD3a2B67e9c4a724137CB8D](https://mumbai.polygonscan.com/address/0x57A757ceCaF2E8Eb8BD3a2B67e9c4a724137CB8D)
-   DistributionExecutable Contract Address on Avalanche Testnet : [0xAe9ce94c7eBb3cF22A06470d06D84Cb7Ce7c55E5](https://testnet.snowtrace.io/address/0xAe9ce94c7eBb3cF22A06470d06D84Cb7Ce7c55E5)
-   Axelar Transaction Detail on Testnet : [0xed3b2d4ffd826b507f31d942c94774a8ca3b8c39529bf7c2be9031e9adc0769c](https://testnet.axelarscan.io/gmp/0xed3b2d4ffd826b507f31d942c94774a8ca3b8c39529bf7c2be9031e9adc0769c:96)

## Followed Steps

1.  All necessary steps followed, as mentioned in [axelar-local-gmp-examples](https://github.com/axelarnetwork/axelar-local-gmp-examples) repository. Action taking in this step is

    -   Executed `git clone https://github.com/axelarnetwork/axelar-local-gmp-examples.git` to clone the repository.
    -   Executed `npm ci` to install all dependency
    -   Executed `cp .env.example .env` and add private-key details in `.env` file.

2.  Updated `/examples/call-contract-with-token/DistributionExecutable.sol`.

    -   Firstly Added `TransactionInfo` struct to in-order to define layout of Payment Information.

    ```solidity
    struct TransactionInfo {
        address sender;
        address tokenAddress;
        uint256 amount;
        string message;
    }

    ```

    -   Added two mappings recipientsToTransactions and recipientsTransactionCounter to store recipient's transaction info and recipient's transaction count respectively.

    ```solidity
    mapping(address => TransactionInfo[]) public recipientsToTransactions;
    mapping(address => uint) public recipientsTransactionCounter;
    ```

    -   There were two updates made `sendToMany` function. 1, to add a function parameter for message and 2, to accomodate message and sender address into payload

    ```solidity
    function sendToMany(
        string memory destinationChain,
        string memory destinationAddress,
        address[] calldata destinationAddresses,
        string memory symbol,
        uint256 amount,
        string memory message //   <---- Added paramter for message
    ) external payable {
        address tokenAddress = gateway.tokenAddresses(symbol);
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);
        IERC20(tokenAddress).approve(address(gateway), amount);
        bytes memory payload = abi.encode(destinationAddresses, message, msg.sender); //  <-- Included message and sender details in
        if (msg.value > 0) {
            gasReceiver.payNativeGasForContractCallWithToken{ value: msg.value }(
                address(this),
                destinationChain,
                destinationAddress,
                payload,
                symbol,
                amount,
                msg.sender
            );
        }
        gateway.callContractWithToken(destinationChain, destinationAddress, payload, symbol, amount);
    }

    ```

    -   Final change was made to `_executeWithToken` function. first change is to correct type detail in `abi.decode` function call to extract message and sender address
        and second change is to create and store TransactionInfo on chain

    ```solidity
    function _executeWithToken(
        string calldata,
        string calldata,
        bytes calldata payload,
        string calldata tokenSymbol,
        uint256 amount
    ) internal override {
        (address[] memory recipients, string memory message, address sender) = abi.decode(payload, (address[], string, address)); // <-- updated decode types to accomodate message and sender
        address tokenAddress = gateway.tokenAddresses(tokenSymbol);
        uint256 sentAmount = amount / recipients.length;
        for (uint256 i = 0; i < recipients.length; i++) {
            IERC20(tokenAddress).transfer(recipients[i], sentAmount);
            TransactionInfo memory txnInfo = TransactionInfo(sender, tokenAddress, sentAmount, message); // Creating transactionInfo
            recipientsToTransactions[recipients[i]].push(txnInfo); //   <--- Storing TransactionInfo for that recipient
            recipientsTransactionCounter[recipients[i]]++; //   <--- Incrementing recipient counter
        }
    }

    ```

3.  Code change in `//examples/index.js` were made to add below features

    -   Added **`message`** parameter to get message passed as argument from command line.
    -   Change to support **`Multiple Recipients`** was also added.
    -   Update **`sendToMany`** function call to include message paramater.
    -   Few Cosmatic change we also done to increase readability of command output.

4.  Testing was done on local env

    -   Executed `node scripts/createLocal.js` to start local node and keep that command running on terminal.
    -   Executed `node scripts/deploy.js examples/call-contract-with-token local` to deploy **DistributionExecutable** contract on local node.
    -   Executed `node scripts/test examples/call-contract-with-token local "Polygon" "Avalanche" 100 0x438d67e825D31D4a9910241074025B75b08470e1,0x57E2355F3CD8CB932952e773a5C57b64cE692e76 "Hello World"` to transfer 100 aUSDC to `two different accounts` from `Polygon` to `Avalanche`. Amount will be divided equally among recipients.

    ![image](https://user-images.githubusercontent.com/56193257/205261853-cd10108e-441f-4318-95ce-4939cc9a8348.png)

5.  Finally, after all testing was done, deployment and execution was done on testnet.

    -   Executed `node scripts/deploy.js examples/call-contract-with-token testnet` to deploy **DistributionExecutable** contract on local node. It will also update `/info/testnet.json` with correct **DistributionExecutable** address.
    -   Executed `node scripts/test examples/call-contract-with-token testnet "Polygon" "Avalanche" 1 0x438d67e825D31D4a9910241074025B75b08470e1,0x57E2355F3CD8CB932952e773a5C57b64cE692e76 "Hello World"` to transfer 1 aUSDC to `two different accounts` from `Polygon` to `Avalanche`. Amount will be divided equally among recipients.

    ```bash
        $ node scripts/test examples/call-contract-with-token testnet "Polygon" "Avalanche" 1 0x438d67e825D31D4a9910241074025B75b08470e1,0x57E2355F3CD8CB932952e773a5C57b64cE692e76 "Hello World"

        --- Initially ---
        Source : 0x3B70368d0FEEbb7c3B32551b82E499c777e76c22 has 12.7 aUSDC
        Destination 1: 0x438d67e825D31D4a9910241074025B75b08470e1 has 4.5 aUSDC
        Destination 2: 0x57E2355F3CD8CB932952e773a5C57b64cE692e76 has 0.5 aUSDC
        Transaction Hash : 0xed3b2d4ffd826b507f31d942c94774a8ca3b8c39529bf7c2be9031e9adc0769c

        --- Waiting Period Started ---
        Waited for 13 minutes
        --- After ---
        Source(After Transaction) : 0x3B70368d0FEEbb7c3B32551b82E499c777e76c22 has 11.7 aUSDC

        ------------For Account 0x438d67e825D31D4a9910241074025B75b08470e1------------
        Destination(Before Transaction) 1 : 0x438d67e825D31D4a9910241074025B75b08470e1 has 4.5 aUSDC
        Destination(After Transaction) 1: 0x438d67e825D31D4a9910241074025B75b08470e1 has 5 aUSDC
                Details of TransactionInfo
                ---------------------------
                Sender           : 0x3B70368d0FEEbb7c3B32551b82E499c777e76c22
                TokenAddress     : 0x57F1c63497AEe0bE305B8852b354CEc793da43bB
                Amount           : 0.5
                Message          : Hello World

        ------------For Account 0x57E2355F3CD8CB932952e773a5C57b64cE692e76------------
        Destination(Before Transaction) 2 : 0x57E2355F3CD8CB932952e773a5C57b64cE692e76 has 0.5 aUSDC
        Destination(After Transaction) 2: 0x57E2355F3CD8CB932952e773a5C57b64cE692e76 has 1 aUSDC
                Details of TransactionInfo
                ---------------------------
                Sender           : 0x3B70368d0FEEbb7c3B32551b82E499c777e76c22
                TokenAddress     : 0x57F1c63497AEe0bE305B8852b354CEc793da43bB
                Amount           : 0.5
                Message          : Hello World

    ```

    ![image](https://user-images.githubusercontent.com/56193257/205265294-e515b7cf-ab1d-41ea-a4ef-a642f9de81c4.png)

## Challenges Faced

-   Took sometime to understand the contract & flow.
-   Time was also consumed in order to transact on local node.
-   Took sometime for code amendment.
-   After making all development related changes, testing on local env took entire time as transferred amount was not getting divided correctly between recipients. See image in Step 4.
