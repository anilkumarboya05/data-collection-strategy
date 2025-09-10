// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/**
 * @title Data Collection Strategy
 * @dev A smart contract for secure, transparent, and incentivized data collection
 * @author Blockchain Research Team
 */
contract DataCollectionStrategy {
    
    // State variables
    address public owner;
    uint256 public totalDataPoints;
    uint256 public rewardPool;
    
    // Data structure for collected information
    struct DataPoint {
        uint256 id;
        address contributor;
        string dataHash; // IPFS hash or data fingerprint
        string category;
        uint256 timestamp;
        uint256 reward;
        bool verified;
    }
    
    // Mappings
    mapping(uint256 => DataPoint) public dataRegistry;
    mapping(address => uint256[]) public contributorData;
    mapping(address => uint256) public contributorRewards;
    mapping(string => bool) public categoryExists;
    
    // Events
    event DataSubmitted(
        uint256 indexed dataId,
        address indexed contributor,
        string category,
        uint256 reward
    );
    
    event DataVerified(
        uint256 indexed dataId,
        address indexed verifier
    );
    
    event RewardsClaimed(
        address indexed contributor,
        uint256 amount
    );
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }
    
    modifier validDataId(uint256 _dataId) {
        require(_dataId > 0 && _dataId <= totalDataPoints, "Invalid data ID");
        _;
    }
    
    // Constructor
    constructor() {
        owner = msg.sender;
        totalDataPoints = 0;
        rewardPool = 0;
        
        // Initialize default categories
        categoryExists["research"] = true;
        categoryExists["market_analysis"] = true;
        categoryExists["technical_review"] = true;
    }
    
    /**
     * @dev Core Function 1: Submit Data
     * @param _dataHash IPFS hash or unique identifier for the data
     * @param _category Category of the data being submitted
     * @return dataId Unique identifier for the submitted data
     */
    function submitData(
        string memory _dataHash,
        string memory _category
    ) public returns (uint256) {
        require(bytes(_dataHash).length > 0, "Data hash cannot be empty");
        require(categoryExists[_category], "Invalid category");
        
        totalDataPoints++;
        uint256 newDataId = totalDataPoints;
        
        // Calculate reward based on category and current pool
        uint256 baseReward = 100; // Base reward in wei
        uint256 calculatedReward = calculateReward(_category, baseReward);
        
        // Create new data point
        dataRegistry[newDataId] = DataPoint({
            id: newDataId,
            contributor: msg.sender,
            dataHash: _dataHash,
            category: _category,
            timestamp: block.timestamp,
            reward: calculatedReward,
            verified: false
        });
        
        // Update contributor records
        contributorData[msg.sender].push(newDataId);
        
        emit DataSubmitted(newDataId, msg.sender, _category, calculatedReward);
        
        return newDataId;
    }
    
    /**
     * @dev Core Function 2: Verify Data
     * @param _dataId ID of the data to verify
     */
    function verifyData(uint256 _dataId) public onlyOwner validDataId(_dataId) {
        require(!dataRegistry[_dataId].verified, "Data already verified");
        
        dataRegistry[_dataId].verified = true;
        
        // Add reward to contributor's balance
        address contributor = dataRegistry[_dataId].contributor;
        uint256 reward = dataRegistry[_dataId].reward;
        
        contributorRewards[contributor] += reward;
        
        emit DataVerified(_dataId, msg.sender);
    }
    
    /**
     * @dev Core Function 3: Claim Rewards
     * @return amount Amount of rewards claimed
     */
    function claimRewards() public returns (uint256) {
        uint256 rewardAmount = contributorRewards[msg.sender];
        require(rewardAmount > 0, "No rewards to claim");
        require(address(this).balance >= rewardAmount, "Insufficient contract balance");
        
        // Reset contributor rewards
        contributorRewards[msg.sender] = 0;
        
        // Transfer rewards
        payable(msg.sender).transfer(rewardAmount);
        
        emit RewardsClaimed(msg.sender, rewardAmount);
        
        return rewardAmount;
    }
    
    // Helper Functions
    
    /**
     * @dev Calculate reward based on category and base amount
     */
    function calculateReward(string memory _category, uint256 _baseAmount) 
        private 
        pure 
        returns (uint256) 
    {
        // Different categories have different multipliers
        if (keccak256(bytes(_category)) == keccak256(bytes("technical_review"))) {
            return _baseAmount * 2; // Technical reviews get 2x reward
        } else if (keccak256(bytes(_category)) == keccak256(bytes("market_analysis"))) {
            return _baseAmount * 3; // Market analysis gets 3x reward
        } else {
            return _baseAmount; // Default multiplier
        }
    }
    
    /**
     * @dev Add new data category (owner only)
     */
    function addCategory(string memory _category) public onlyOwner {
        require(!categoryExists[_category], "Category already exists");
        categoryExists[_category] = true;
    }
    
    /**
     * @dev Fund the contract with ETH for rewards
     */
    function fundContract() public payable onlyOwner {
        rewardPool += msg.value;
    }
    
    /**
     * @dev Get contributor's data submissions
     */
    function getContributorData(address _contributor) 
        public 
        view 
        returns (uint256[] memory) 
    {
        return contributorData[_contributor];
    }
    
    /**
     * @dev Get data point details
     */
    function getDataPoint(uint256 _dataId) 
        public 
        view 
        validDataId(_dataId)
        returns (
            address contributor,
            string memory dataHash,
            string memory category,
            uint256 timestamp,
            uint256 reward,
            bool verified
        ) 
    {
        DataPoint memory data = dataRegistry[_dataId];
        return (
            data.contributor,
            data.dataHash,
            data.category,
            data.timestamp,
            data.reward,
            data.verified
        );
    }
    
    /**
     * @dev Get contract statistics
     */
    function getContractStats() 
        public 
        view 
        returns (
            uint256 totalData,
            uint256 contractBalance,
            uint256 totalRewardPool
        ) 
    {
        return (
            totalDataPoints,
            address(this).balance,
            rewardPool
        );
    }
}
