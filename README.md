Bitqueen.sol
# BITQUEEN--TOKAN
Official BITQUEEN (BITQ) BEP20 Token Smart Contract on Binance Smart Chain
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 @title BITQUEEN (BITQ) — Full-featured BEP20/ERC20 single-file contract for Remix
 NOTE: Educational / small-project use. For production, prefer audited OpenZeppelin contracts.
*/

contract BitQueenCoin {
    // ==== Token metadata (edit before deploy if needed) ====
    string public name = "BITQUEEN";
    string public symbol = "BITQ";
    uint8 public constant decimals = 18;

    // Example: pass initialSupplyTokens = 1000000000 (1 billion) to constructor
    uint256 public totalSupply;

    // ==== Access control ====
    address public owner;
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    modifier onlyOwner() {
        require(msg.sender == owner, "Ownable: caller is not the owner");
        _;
    }

    // ==== Pausable ====
    bool public paused;
    event Paused(address account);
    event Unpaused(address account);

    modifier whenNotPaused() {
        require(!paused, "Pausable: paused");
        _;
    }

    modifier whenPaused() {
        require(paused, "Pausable: not paused");
        _;
    }

    // ==== Balances & allowances ====
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    // ==== Events ====
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed ownerAddr, address indexed spender, uint256 value);

    // ==== Constructor ====
    /**
     * @param initialSupplyTokens number of whole tokens (without decimals), e.g. 1_000_000_000 for 1 billion
     */
    constructor(uint256 initialSupplyTokens) {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), owner);

        // compute total supply = initialSupplyTokens * 10^decimals
        totalSupply = initialSupplyTokens * (10 ** uint256(decimals));
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    // ==== ERC20 core functions ====
    function transfer(address to, uint256 value) external whenNotPaused returns (bool) {
        _transfer(msg.sender, to, value);
        return true;
    }

    function approve(address spender, uint256 value) external whenNotPaused returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address from, address to, uint256 value) external whenNotPaused returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        require(allowed >= value, "ERC20: insufficient allowance");
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - value;
        }
        _transfer(from, to, value);
        return true;
    }

    // ==== Allowance helpers ====
    function increaseAllowance(address spender, uint256 addedValue) external whenNotPaused returns (bool) {
        allowance[msg.sender][spender] += addedValue;
        emit Approval(msg.sender, spender, allowance[msg.sender][spender]);
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue) external whenNotPaused returns (bool) {
        uint256 current = allowance[msg.sender][spender];
        require(current >= subtractedValue, "ERC20: decreased allowance below zero");
        allowance[msg.sender][spender] = current - subtractedValue;
        emit Approval(msg.sender, spender, allowance[msg.sender][spender]);
        return true;
    }

    // ==== Mint & Burn (owner-restricted mint) ====
    event Mint(address indexed to, uint256 value);
    event Burn(address indexed from, uint256 value);

    function mint(address to, uint256 amount) external onlyOwner whenNotPaused returns (bool) {
        require(to != address(0), "mint to zero");
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Mint(to, amount);
        emit Transfer(address(0), to, amount);
        return true;
    }

    function burn(uint256 amount) external whenNotPaused returns (bool) {
        uint256 bal = balanceOf[msg.sender];
        require(bal >= amount, "ERC20: burn exceeds balance");
        balanceOf[msg.sender] = bal - amount;
        totalSupply -= amount;
        emit Burn(msg.sender, amount);
        emit Transfer(msg.sender, address(0), amount);
        return true;
    }

    function burnFrom(address from, uint256 amount) external whenNotPaused returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        require(allowed >= amount, "ERC20: burn amount exceeds allowance");
        uint256 bal = balanceOf[from];
        require(bal >= amount, "ERC20: burn exceeds balance");

        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowance[from][msg.sender] - amount;
        }
        balanceOf[from] = bal - amount;
        totalSupply -= amount;
        emit Burn(from, amount);
        emit Transfer(from, address(0), amount);
        return true;
    }

    // ==== Internal transfer implementation ====
    function _transfer(address from, address to, uint256 value) internal {
        require(to != address(0), "ERC20: transfer to zero");
        uint256 fromBal = balanceOf[from];
        require(fromBal >= value, "ERC20: transfer exceeds balance");
        unchecked {
            balanceOf[from] = fromBal - value;
            balanceOf[to] += value;
        }
        emit Transfer(from, to, value);
    }

    // ==== Ownership functions ====
    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is zero");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function renounceOwnership() external onlyOwner {
        emit OwnershipTransferred(owner, address(0));
        owner = address(0);
    }

    // ==== Pausable functions ====
    function pause() external onlyOwner whenNotPaused {
        paused = true;
        emit Paused(msg.sender);
    }

    function unpause() external onlyOwner whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }

    // ==== Rescue tokens (in case someone sends ERC20 to this contract) ====
    // Use low-level call to support non-standard ERC20s that don't return bool.
    function rescueERC20(address tokenAddress, address to, uint256 amount) external onlyOwner returns (bool) {
        require(to != address(0), "rescue to zero");
        require(tokenAddress != address(this), "cannot rescue this token");

        bytes memory payload = abi.encodeWithSignature("transfer(address,uint256)", to, amount);
        (bool success, bytes memory returnData) = tokenAddress.call(payload);
        require(success, "token transfer call failed");
        if (returnData.length > 0) {
            // some tokens return bool
            require(abi.decode(returnData, (bool)), "ERC20: transfer returned false");
        }
        return true;
    }

    // ==== Fallback / receive ====
    receive() external payable {
        // allow receiving BNB. Withdraw using withdrawBNB.
    }

    // Owner can withdraw native BNB accidentally sent to this contract
    function withdrawBNB(address payable to, uint256 amount) external onlyOwner returns (bool) {
        require(to != address(0), "withdraw to zero");
        require(address(this).balance >= amount, "insufficient balance");
        to.transfer(amount);
        return true;
    }

    // ==== Helpers (view) ====
    function isOwner(address account) external view returns (bool) {
        return account == owner;
    }
}
