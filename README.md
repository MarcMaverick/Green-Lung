// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";

contract MahokiGreenLungRevolution is ERC20, ERC20Burnable, Ownable {
    uint256 private constant MAX_SUPPLY = 1_000_000_000 * 10**18; // 1 Milliarde MGLR
    uint256 public maxTransactionAmount = 10_000_000 * 10**18; // Whale-Schutz: max. 10 Mio. pro Transfer

    address public immutable ownerAddress = 0x9c6C6a3b902241911DB26c7d5a84A5872E6844A6;
    address public immutable spenderAddress = 0x5AF03D04451a9E7406C4B7f992192FA16264548F;
    address public immutable reserveAddress = 0x3c4ED40D6a0987B93Ae64d99ad62fA9A746E86C4;
    address public immutable newTeamAddress = 0xc34E9AcDa81374c0f9597b96a99C6ee1078714F2;

    mapping(address => uint256) private stakedBalances;
    mapping(address => uint256) private stakingTimestamps;

    event Staked(address indexed user, uint256 amount, uint256 timestamp);
    event Unstaked(address indexed user, uint256 amount, uint256 reward, uint256 timestamp);

    constructor() ERC20("Mahoki Green Lung Revolution", "MGLR") {
        _mint(ownerAddress, MAX_SUPPLY * 50 / 100); // 50% an den Eigentümer
        _mint(reserveAddress, MAX_SUPPLY * 30 / 100); // 30% für Reserve
        _mint(spenderAddress, MAX_SUPPLY * 10 / 100); // 10% für Spender
        _mint(newTeamAddress, MAX_SUPPLY * 10 / 100); // 10% für das neue Team
    }

    // 🛡️ Override Transfer-Funktion: Burning & Whale-Schutz
    function _transfer(address sender, address recipient, uint256 amount) internal override {
        require(amount <= maxTransactionAmount, "Transfer amount exceeds max limit");

        uint256 burnAmount = (amount * 2) / 100; // 2% verbrennen
        uint256 sendAmount = amount - burnAmount;

        super._burn(sender, burnAmount);
        super._transfer(sender, recipient, sendAmount);
    }

    // 🔒 Staking-Funktion
    function stake(uint256 amount) external {
        require(balanceOf(msg.sender) >= amount, "Not enough tokens to stake");
        _transfer(msg.sender, address(this), amount);
        stakedBalances[msg.sender] += amount;
        stakingTimestamps[msg.sender] = block.timestamp;
        emit Staked(msg.sender, amount, block.timestamp);
    }

    // 🎁 Staking-Belohnung abrufen (5% APY)
    function unstake() external {
        require(stakedBalances[msg.sender] > 0, "No tokens staked");

        uint256 stakedTime = block.timestamp - stakingTimestamps[msg.sender];
        uint256 reward = (stakedBalances[msg.sender] * 5 * stakedTime) / (365 days * 100);

        uint256 totalAmount = stakedBalances[msg.sender] + reward;
        stakedBalances[msg.sender] = 0;
        stakingTimestamps[msg.sender] = 0;

        _transfer(address(this), msg.sender, totalAmount);
        emit Unstaked(msg.sender, totalAmount, reward, block.timestamp);
    }

    // 🗳️ Governance: Abstimmung über Token-Änderungen
    function changeMaxTransactionAmount(uint256 newAmount) external onlyOwner {
        require(newAmount > 0, "Amount must be greater than zero");
        maxTransactionAmount = newAmount;
    }
}
