## QA
---

### Layout Order [1]

- The best-practices for layout within a contract is the following order: state variables, events, modifiers, constructor and functions.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol

```solidity
// event coming after functions
95:    event SetStatus(bytes32 indexed key, bool status);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol

```solidity
// modifier coming after functions
38:    modifier onlyExchange() {

// event coming after functions
107:    event AdjustPosition(
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol

```solidity
// modifier coming after functions
72:    modifier onlyExchange() {

// events coming after functions
278:    event CollectCollateral(uint256 indexed positionId, address collateral, uint256 amount);
284:    event SendCollateral(uint256 indexed positionId, address collateral, uint256 amount);
292:    event LiquidateCollateral(
301:    event ApproveCollateral(bytes32 indexed key, uint256 collateralRatio, uint256 liqRatio, uint256 liqBonus);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol

```solidity
// modifier coming after functions
27:    modifier onlyExchange() {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol

```solidity
// modifier coming after functions
23:    modifier onlyPool() {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol

```solidity
// events coming after functions
827:    event Deposit(address indexed user, uint256 amt, uint256 depositFees, uint256 tokens);
834:    event Withdraw(address indexed user, uint256 tokens, uint256 withdrawFees, uint256 amtReceived);
841:    event InitiateDeposit(uint256 depositId, address depositor, address user, uint256 amount);
849:    event ProcessDeposit(uint256 depositId, address user, uint256 amount, uint256 tokens, uint256 requestedTime);
856:    event InitiateWithdrawal(uint256 withdrawalId, address withdrawer, address user, uint256 tokens);
864:    event ProcessWithdrawal(uint256 withdrawalId, address user, uint256 tokens, uint256 amount, uint256 requestedTime);
872:    event ProcessWithdrawalPartially(
880:    event RegisterTrade(bytes32 indexed referralCode, uint256 totalFees, uint256 externalFees);
886:    event OpenLong(uint256 markPrice, uint256 amt, uint256 fees);
892:    event CloseLong(uint256 markPrice, uint256 amt, uint256 fees);
898:    event OpenShort(uint256 markPrice, uint256 amt, uint256 fees);
904:    event CloseShort(uint256 markPrice, uint256 amt, uint256 fees);
909:    event Liquidate(uint256 markPrice, uint256 amt);
915:    event HedgePositions(int256 currentPosition, int256 requiredPosition, int256 marginDelta);
920:    event RebalanceMargin(int256 marginDelta, int256 additionalDelta);
924:    event IncreaseMargin(uint256 marginAdded);
929:    event UpdateSynthetixTrackingCode(bytes32 oldCode, bytes32 newCode);
923:    event UpdateLeverage(uint256 oldLev, uint256 newLev);
939:    event UpdateStandardSize(uint256 oldStdSize, uint256 newStdSize);
944:    event UpdateFeeReceipient(address oldFeeReceipient, address newFeeReceipient);
947:    event UpdateFees(uint256 oldDepositFee, uint256 newDepositFee, uint256 oldWithdrawlFee, uint256 newWithdrawalFee);
950:    event UpdateBaseTradingFee(uint256 oldFee, uint256 newFee);
953:    event UpdateDevFee(uint256 oldFee, uint256 newFee);
958:    event UpdatePriceImpactDelta(uint256 oldDelta, uint256 newDelta);
965:    event UpdateDelays(
970:    event TransferMargin(uint256 marginDelta);
973:    event SubmitDelayedOrder(int256 sizeDelta);
976:    event ReceiveEther(address indexed from, uint256 amt);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol

```solidity
// events coming after functions
808:    event InitiateDeposit(uint256 depositId, address depositor, address user, uint256 amount);
816:    event ProcessDeposit(uint256 depositId, address user, uint256 amount, uint256 tokens, uint256 requestedTime);
823:    event InitiateWithdrawal(uint256 withdrawalId, address withdrawer, address user, uint256 tokens);
831:    event ProcessWithdrawal(uint256 withdrawalId, address user, uint256 tokens, uint256 amount, uint256 requestedTime);
839:    event ProcessWithdrawalPartially(
846:    event ProcessWithdrawalPartially(
853:    event UpdateFees(uint256 oldPerf, uint256 oldWithdraw, uint256 newPerf, uint256 newWithdraw);
858:    event UpdateSynthetixTrackingCode(bytes32 oldCode, bytes32 newCode);
863:    event UpdateReferralCode(bytes32 oldCode, bytes32 newCode);
868:    event UpdateMinDeposit(uint256 oldMinimum, uint256 newMinimum);
873:    event UpdateMaxDeposit(uint256 oldMax, uint256 newMax);
880:    event UpdateDelays(
887:    event UpdateLeverage(uint256 oldLeverage, uint256 newLeverage);
892:    event UpdateCollRatio(uint256 oldRatio, uint256 newRatio);
897:    event UpdatePriceImpactDelta(uint256 oldDelta, uint256 newDelta);
907:    event OpenPosition(
923:    event ClosePosition(
930:    event ClearOpenOrder(uint256 indexed positionId, uint256 clearedLong);
938:    event ReverseOpenOrder(
949:    event ClearCloseOrder(uint256 indexed positionId, uint256 clearedShort);
957:    event ReverseCloseOrder(
968:    event TransferPerpMargin(uint256 indexed positionId, int256 marginDelta);
973:    event AddCollateral(uint256 indexed positionId, uint256 additionalCollateral);
978:    event RemoveCollateral(uint256 indexed positionId, uint256 collateralRemoved);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol

```solidity
// events coming after functions
433:    event UpdateSkewNormalizationFactor(uint256 oldFactor, uint256 newFactor);
438:    event UpdateMaxFundingRate(uint256 oldRate, uint256 newRate);
444:    event OpenLong(address indexed trader, uint256 amt, uint256 totalCost);
453:    event OpenShort(
466:    event CloseLong(address indexed trader, uint256 amt, uint256 totalCost);
475:    event CloseShort(
491:    event Liquidate(
505:    event AddCollateral(
514:    event RemoveCollateral(
521:    event UpdateFundingRate(uint256 lastUpdated, uint256 newNormalizationFactor);
```

---

### Function Visibility [2]

- Order of Functions: Ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. Functions should be grouped according to their visibility and ordered: constructor, receive function (if exists), fallback function (if exists), public, external, internal, private. Within a grouping, place the view and pure functions last.

```solidity
// external function coming before public ones
48:    function adjustPosition(
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol

```solidity
// receive function should come before all the functions
708:    receive() external payable {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol

```solidity
// receive function should come before all the functions
454:    receive() external payable {
```

---

### natSpec missing [3]

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol

```solidity
18:    modifier onlyVault() {

25:    constructor(string memory name, string memory symbol) ERC20(name, symbol, 18) {}

27:    function mint(address _user, uint256 _amt) external onlyVault {

31:    function burn(address _user, uint256 _amt) external onlyVault {

35:    function setVault(address _vault) external {

46:    error OnlyVault(address thrower, address caller, address vault);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol

```solidity
47:    constructor(

62:    function init(

84:    function refreshSynthetixAddresses() public {

89:    function setStatusFunction(bytes32 key, bool status) public requiresAuth {

95:    event SetStatus(bytes32 indexed key, bool status);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol

```solidity
9:    contract SynthetixAdapter is ISynthetixAdapter {

13:    constructor(ISynthetix _synthetix, IExchangeRates _exchangeRates) {

18:    function getSynth(bytes32 key) public view override returns (address synth) {

22:    function getCurrencyKey(address synth) public view override returns (bytes32 key) {

26:    function getAssetPrice(bytes32 key) public view override returns (uint256, bool) {

30:    function getAssetPrice(address synth) public view override returns (uint256, bool) {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol

```solidity
26:    constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)

33:    function refresh() public {

38:    modifier onlyExchange() {

44:    function tokenURI(uint256 tokenId) public view override returns (string memory) {

48:    function adjustPosition(

92:    function transferFrom(address _from, address _to, uint256 _id) public override {

97:    function safeTransferFrom(address _from, address _to, uint256 _id) public override {

102:    function safeTransferFrom(address _from, address _to, uint256 _id, bytes calldata data) public override {

107:    event AdjustPosition(
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol

```solidity
51:    constructor(uint256 susdRatio, uint256 susdLiqRatio, uint256 susdLiqBonus, ISystemManager _systemManager)

64:    function refresh() public {

72:    modifier onlyExchange() {

// @return missing
121:    function liquidate(uint256 positionId, uint256 debt, address user)
153:    function getMinCollateral(uint256 shortAmount, address collateral)
176:    function getLiquidationBonus(address collateral, uint256 collateralAmount)
192:    function canLiquidate(uint256 positionId) public view override returns (bool) {
215:    function maxLiquidatableDebt(uint256 positionId) public view override returns (uint256 maxDebt) {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol

```solidity
15:    constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)

22:    function refresh() public {

27:    modifier onlyExchange() {

32:    function mint(address _user, uint256 _amt) public onlyExchange {

36:    function burn(address _user, uint256 _amt) public onlyExchange {

40:    function transfer(address _to, uint256 _amount) public override returns (bool) {

45:    function transferFrom(address _from, address _to, uint256 _amount) public override returns (bool) {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol

```solidity
7:    contract LiquidityToken is ERC20 {

12:    constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)

19:    function refresh() public {

23:    modifier onlyPool() {

28:    function mint(address _user, uint256 _amt) public onlyPool {

32:    function burn(address _user, uint256 _amt) public onlyPool {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol

```solidity
27:   contract LiquidityPool is ILiquidityPool, Auth, ReentrancyGuard, PauseModifier {

35:    struct QueuedDeposit {

43:    struct QueuedWithdraw {

163:    function refresh() public {

172:    modifier onlyExchange() {

// @return missing
340:    function getTokenPrice() public view override returns (uint256) {
379:    function orderFee(int256 sizeDelta) public view override returns (uint256) {
399:    function baseAssetPrice() public view override returns (uint256 spotPrice, bool isInvalid) {
404:    function getMarkPrice() public view override returns (uint256, bool) {
409:    function getSkew() external view override returns (int256) {
414:    function getDelta() external view override returns (uint256) {
419:    function getExposure() external view returns (int256) {
430:    function openLong(uint256 amount, address user, bytes32 referralCode)
462:    function closeLong(uint256 amount, address user, bytes32 referralCode)
494:    function openShort(uint256 amount, address user, bytes32 referralCode)
526:    function closeShort(uint256 amount, address user, bytes32 referralCode)
720:    function _getSkew() internal view returns (int256 skew) {
727:    function _getDelta() internal view returns (uint256 delta) {
738:    function _getTotalPerpPosition() internal view returns (int256 positionSize) {
746:    function _getExposure() internal view returns (int256 exposure) {
754:    function _calculateExposure(int256 currentPosition) internal view returns (int256 exposure) {
764:    function _calculateMargin(int256 size) internal view returns (uint256 margin) {
774:    function _getTotalMargin() internal view returns (uint256) {
798:    function _hedge(int256 size, bool isLiquidation) internal returns (uint256 hedgingFees) {

367:    function getSlippageFee(int256 sizeDelta) public view returns (uint256) {

704:    function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {

708:    receive() external payable {

780:    function _placeDelayedOrder(int256 sizeDelta, bool isLiquidation) internal {

```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol

```solidity
21:   contract KangarooVault is Auth, ReentrancyGuard, PauseModifier {

25:    struct PositionData {

37:    struct LastTradeData {

43:    struct QueuedDeposit {

51:    struct QueuedWithdraw {

156:    constructor(

// @return missing
341:    function getTokenPrice() public view returns (uint256) {
365:    function getTotalSupply() public view returns (uint256) {

454:    receive() external payable {

492:    function setReferralCode(bytes32 _code) external requiresAuth {

556:    function _openPosition(uint256 amt, uint256 minCost) internal {

613:    function _clearPendingOpenOrders(uint256 maxCost) internal {

670:    function _closePosition(uint256 amt, uint256 maxCost) internal {

730:    function _clearPendingCloseOrders(uint256 minCost) internal {

782:    function _resetTrade() internal {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol

```solidity
21:   contract Exchange is IExchange, Auth, ReentrancyGuard, PauseModifier {

67:    constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)

74:    function refresh() public {

// @return missing
86:    function openTrade(TradeParams memory tradeParams)
100:    function closeTrade(TradeParams memory tradeParams)
155:    function getIndexPrice() public view override returns (uint256 indexPrice, bool isInvalid) {
167:    function getFundingRate() public view override returns (int256 fundingRate, bool isInvalid) {
186:    function getMarkPrice() public view override returns (uint256 markPrice, bool isInvalid) {
206:    function orderFee(int256 sizeDelta) public view returns (uint256 fees) {

// @param and @return missing
233:    function _openTrade(TradeParams memory params) internal returns (uint256, uint256) {
288:    function _closeTrade(TradeParams memory params) internal returns (uint256) {

@param missing
333:    function _liquidate(uint256 positionId, uint256 debtRepaying) internal {
356:    function _addCollateral(uint256 positionId, uint256 amount) internal {
382:    function _removeCollateral(uint256 positionId, uint256 amount) internal {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/utils/PauseModifier.sol

```solidity
7:    abstract contract PauseModifier {

10:    modifier whenNotPaused(bytes32 key) {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol

```solidity
4:    library SignedMath {

5:    function signedAbs(int256 x) internal pure returns (int256) {

9:    function abs(int256 x) internal pure returns (uint256) {

13:    function max(int256 x, int256 y) internal pure returns (int256) {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IVaultToken.sol

```solidity
4:    interface IVaultToken {

5:    function mint(address _user, uint256 _amt) external;

6:    function burn(address _user, uint256 _amt) external;

8:    function totalSupply() external view returns (uint256 totalSupply);

9:    function balanceOf(address user) external view returns (uint256 balance);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ISystemManager.sol

```solidity
15:   interface ISystemManager {

16:    function refreshSynthetixAddresses() external;

18:    function isPaused(bytes32) external view returns (bool);

20:    function pool() external view returns (ILiquidityPool);

22:    function exchange() external view returns (IExchange);

24:    function powerPerp() external view returns (IPowerPerp);

26:    function shortCollateral() external view returns (IShortCollateral);

28:    function shortToken() external view returns (IShortToken);

30:    function liquidityToken() external view returns (ILiquidityToken);

32:    function synthetixAdapter() external view returns (ISynthetixAdapter);

34:    function addressResolver() external view returns (IAddressResolver);

36:    function perpMarket() external view returns (IPerpsV2Market);

38:    function PERP_MARKET_CONTRACT() external view returns (bytes32);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ISynthetixAdapter.sol

```solidity
4:    interface ISynthetixAdapter {

5:    function getSynth(bytes32 key) external view returns (address);

7:    function getCurrencyKey(address synth) external view returns (bytes32);

9:    function getAssetPrice(bytes32 key) external view returns (uint256, bool);

11:    function getAssetPrice(address synth) external view returns (uint256, bool);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IShortToken.sol

```solidity
4:    interface IShortToken {

5:    struct ShortPosition {

12:    function balanceOf(address owner) external view returns (uint256 balance);

14:    function ownerOf(uint256 tokenId) external view returns (address owner);

16:    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;

18:    function safeTransferFrom(address from, address to, uint256 tokenId) external;

20:    function transferFrom(address from, address to, uint256 tokenId) external;

22:    function approve(address to, uint256 tokenId) external;

24:    function setApprovalForAll(address operator, bool _approved) external;

26:    function getApproved(uint256 tokenId) external view returns (address operator);

28:    function isApprovedForAll(address owner, address operator) external view returns (bool);

30:    function shortPositions(uint256 positionId) external view returns (ShortPosition memory);

32:    function totalShorts() external view returns (uint256);

34:    function adjustPosition(
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IShortCollateral.sol

```solidity
4:    interface IShortCollateral {

5:    struct Collateral {

14:    struct UserCollateral {

23:    function sendCollateral(uint256 positionId, uint256 amount) external;

25:    function collectCollateral(address collateral, uint256 positionId, uint256 amount) external;

27:    function liquidate(uint256 positionId, uint256 debt, address user)

35:    function getLiquidationBonus(address collateral, uint256 collateralAmount)

40:    function getMinCollateral(uint256 shortAmount, address collateral) external view returns (uint256 minCollateral);

42:    function canLiquidate(uint256 positionId) external view returns (bool);

44:    function maxLiquidatableDebt(uint256 positionId) external view returns (uint256 maxDebt);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IPowerPerp.sol

```solidity
4:    interface IPowerPerp {

8:    function totalSupply() external view returns (uint256);

10:    function balanceOf(address account) external view returns (uint256);

12:    function transfer(address recipient, uint256 amount) external returns (bool);

14:    function allowance(address owner, address spender) external view returns (uint256);

16:    function approve(address spender, uint256 amount) external returns (bool);

18:    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

23:    function mint(address _user, uint256 _amt) external;

25:    function burn(address _user, uint256 _amt) external;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityToken.sol

```solidity
4:    interface ILiquidityToken {

8:    function totalSupply() external view returns (uint256);

10:    function balanceOf(address account) external view returns (uint256);

12:    function transfer(address recipient, uint256 amount) external returns (bool);

14:    function allowance(address owner, address spender) external view returns (uint256);

16:    function approve(address spender, uint256 amount) external returns (bool);

18:    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

23:    function mint(address _user, uint256 _amt) external;

25:    function burn(address _user, uint256 _amt) external;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityPool.sol

```solidity
4:    interface ILiquidityPool {

9:    function openLong(uint256 amount, address user, bytes32 referralCode) external returns (uint256 totalCost);

11:    function closeLong(uint256 amount, address user, bytes32 referralCode) external returns (uint256 totalCost);

13:    function openShort(uint256 amount, address user, bytes32 referralCode) external returns (uint256 totalCost);

15:    function closeShort(uint256 amount, address user, bytes32 referralCode) external returns (uint256 totalCost);

17:    function liquidate(uint256 amount) external;

19:    function deposit(uint256 amount, address recipient) external;

21:    function queueDeposit(uint256 amount, address recipient) external;

23:    function processDeposits(uint256 count) external;

25:    function withdraw(uint256 amount, address recipient) external;

27:    function queueWithdraw(uint256 amount, address recipient) external;

29:    function processWithdraws(uint256 count) external;

31:    function hedgePositions() external;

37:    function baseAsset() external view returns (bytes32);

39:    function minDepositDelay() external view returns (uint256);

41:    function minWithdrawDelay() external view returns (uint256);

43:    function baseTradingFee() external view returns (uint256);

45:    function depositFee() external view returns (uint256);

47:    function withdrawalFee() external view returns (uint256);

49:    function devFee() external view returns (uint256);

51:    function feeReceipient() external view returns (address);

53:    function totalQueuedDeposits() external view returns (uint256);

55:    function totalQueuedWithdrawals() external view returns (uint256);

57:    function queuedDepositHead() external view returns (uint256);

59:    function nextQueuedDepositId() external view returns (uint256);

61:    function queuedWithdrawalHead() external view returns (uint256);

63:    function nextQueuedWithdrawalId() external view returns (uint256);

65:    function totalFunds() external view returns (uint256);

67:    function usedFunds() external view returns (int256);

69:    function queuedPerpSize() external view returns (int256);

71:    function futuresLeverage() external view returns (uint256);

73:    function perpPriceImpactDelta() external view returns (uint256);

75:    function standardSize() external view returns (uint256);

77:    function baseAssetPrice() external view returns (uint256, bool);

79:    function getSkew() external view returns (int256 skew);

81:    function getTokenPrice() external view returns (uint256 tokenPrice);

83:    function getMarkPrice() external view returns (uint256 markPrice, bool isInvalid);

85:    function getDelta() external view returns (uint256 delta);

87:    function orderFee(int256 sizeDelta) external view returns (uint256);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IExchange.sol

```solidity
29:    function openTrade(TradeParams memory tradeParams) external returns (uint256 positionId, uint256 totalCost);

31:    function closeTrade(TradeParams memory tradeParams) external returns (uint256 totalCost);

33:    function addCollateral(uint256 positionId, uint256 amount) external;

35:    function removeCollateral(uint256 positionId, uint256 amount) external;

37:    function liquidate(uint256 positionId, uint256 debtRepaying) external;

43:    function PRICING_CONSTANT() external view returns (uint256);

45:    function getIndexPrice() external view returns (uint256 indexPrice, bool isInvalid);

47:    function getFundingRate() external view returns (int256 fundingRate, bool isInvalid);

49:    function getMarkPrice() external view returns (uint256 markPrice, bool isInvalid);

51:    function normalizationFactor() external view returns (uint256);

53:    function skewNormalizationFactor() external view returns (uint256);

55:    function maxFundingRate() external view returns (uint256);

57:    function fundingLastUpdated() external view returns (uint256);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/ISynthetix.sol

```solidity
4:   interface ISynthetix {

5:    function exchange(bytes32 sourceCurrencyKey, uint256 sourceAmount, bytes32 destinationCurrencyKey)

9:    function exchangeOnBehalf(

16:    function exchangeWithTracking(

24:    function exchangeWithTrackingForInitiator(

32:    function exchangeOnBehalfWithTracking(

41:    function exchangeAtomically(

49:    function synths(bytes32 currencyKey) external view returns (address);

51:    function synthsByAddress(address synthAddress) external view returns (bytes32);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IPerpsV2MarketBaseTypes.sol

```solidity
31:    struct Position {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IPerpsV2Market.sol

```solidity
6:    interface IPerpsV2Market {

7:    function transferMargin(int256 marginDelta) external;

9:    function withdrawAllMargin() external;

11:    function modifyPosition(int256 sizeDelta, uint256 priceImpactDelta) external;

13:    function modifyPositionWithTracking(int256 sizeDelta, uint256 priceImpactDelta, bytes32 trackingCode) external;

15:    function closePosition(uint256 priceImpactDelta) external;

17:    function closePositionWithTracking(uint256 priceImpactDelta, bytes32 trackingCode) external;

19:    function submitDelayedOrder(int256 sizeDelta, uint256 priceImpactDelta, uint256 desiredTimeDelta) external;

21:    function submitDelayedOrderWithTracking(

28:    function cancelDelayedOrder(address account) external;

30:    function executeDelayedOrder(address account) external;

32:    function submitOffchainDelayedOrder(int256 sizeDelta, uint256 priceImpactDelta) external;

34:    function submitOffchainDelayedOrderWithTracking(int256 sizeDelta, uint256 priceImpactDelta, bytes32 trackingCode)

37:    function cancelOffchainDelayedOrder(address account) external;

39:    function executeOffchainDelayedOrder(address account, bytes[] calldata priceUpdateData) external payable;

41:    function orderFee(int256 sizeDelta, IPerpsV2MarketBaseTypes.OrderType orderType)

46:    function assetPrice() external view returns (uint256 price, bool invalid);

48:    function baseAsset() external view returns (bytes32 key);

50:    function remainingMargin(address account) external view returns (uint256 marginRemaining, bool invalid);

52:    function positions(address account) external view returns (IPerpsV2MarketBaseTypes.Position memory);

54:    function delayedOrders(address) external view returns (IPerpsV2MarketBaseTypes.DelayedOrder memory);

56:    function postTradeDetails(
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarketManager.sol

```solidity
4:    interface IFuturesMarketManager {

5:    function marketForKey(bytes32 marketKey) external view returns (address);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarketBaseTypes.sol

```solidity
23:    struct Position {
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarket.sol

```solidity
6:    interface IFuturesMarket {

11:    function marketKey() external view returns (bytes32 key);

13:    function baseAsset() external view returns (bytes32 key);

15:    function marketSize() external view returns (uint128 size);

17:    function marketSkew() external view returns (int128 skew);

19:    function fundingLastRecomputed() external view returns (uint32 timestamp);

21:    function fundingSequence(uint256 index) external view returns (int128 netFunding);

23:    function positions(address account)

28:    function assetPrice() external view returns (uint256 price, bool invalid);

30:    function marketSizes() external view returns (uint256 long, uint256 short);

32:    function marketDebt() external view returns (uint256 debt, bool isInvalid);

34:    function currentFundingRate() external view returns (int256 fundingRate);

36:    function unrecordedFunding() external view returns (int256 funding, bool invalid);

38:    function fundingSequenceLength() external view returns (uint256 length);

42:    function notionalValue(address account) external view returns (int256 value, bool invalid);

44:    function profitLoss(address account) external view returns (int256 pnl, bool invalid);

46:    function accruedFunding(address account) external view returns (int256 funding, bool invalid);

48:    function remainingMargin(address account) external view returns (uint256 marginRemaining, bool invalid);

50:    function accessibleMargin(address account) external view returns (uint256 marginAccessible, bool invalid);

52:    function liquidationPrice(address account) external view returns (uint256 price, bool invalid);

54:    function liquidationFee(address account) external view returns (uint256);

56:    function canLiquidate(address account) external view returns (bool);

58:    function orderFee(int256 sizeDelta) external view returns (uint256 fee, bool invalid);

60:    function postTradeDetails(int256 sizeDelta, address sender)

74:    function recomputeFunding() external returns (uint256 lastIndex);

76:    function transferMargin(int256 marginDelta) external;

78:    function withdrawAllMargin() external;

80:    function modifyPosition(int256 sizeDelta) external;

82:    function modifyPositionWithTracking(int256 sizeDelta, bytes32 trackingCode) external;

84:    function submitNextPriceOrder(int256 sizeDelta) external;

86:    function submitNextPriceOrderWithTracking(int256 sizeDelta, bytes32 trackingCode) external;

88:    function cancelNextPriceOrder(address account) external;

90:    function executeNextPriceOrder(address account) external;

92:    function closePosition() external;

94:    function closePositionWithTracking(bytes32 trackingCode) external;

96:    function liquidatePosition(address account) external;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IExchangeRates.sol

```solidity
6:    function rateAndInvalid(bytes32 currencyKey) external view returns (uint256 rate, bool isInvalid);
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IAddressResolver.sol

```solidity
6:    function getAddress(bytes32 name) external view returns (address);

8:    function getSynth(bytes32 key) external view returns (address);

10:    function requireAndGetAddress(bytes32 name, string calldata reason) external view returns (address);
```

---

### State variable and function names [4]

- Variables should be named according to their specifications
- private and internal `variables` should preppend with `underline`
- private and internal `functions` should preppend with `underline`
- constant state variables should be UPPER_CASE

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol

```solidity
//  private and internal `functions` should preppend with `underline`
5:    function signedAbs(int256 x) internal pure returns (int256) {
9:    function abs(int256 x) internal pure returns (uint256) {
13:    function max(int256 x, int256 y) internal pure returns (int256) {
```

---

### Version [5]

- Pragma versions should be standardized and avoid floating pragma `( ^ )`.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/utils/PauseModifier.sol

```soldity
// avoid the pragma ^
3:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ISystemManager.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ISynthetixAdapter.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IShortToken.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IShortCollateral.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IPowerPerp.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityToken.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IExchange.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/ISynthetix.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IPerpsV2MarketBaseTypes.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IPerpsV2Market.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarketManager.sol

```solidity
// avoid the pragma ^
2:    pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarketBaseTypes.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarket.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IExchangeRates.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IAddressResolver.sol

```solidity
// avoid the pragma ^
2:  pragma solidity ^0.8.9;
```


---

### Observations [6]

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol

```solidity
// not clear enough what this constant does
27:    uint256 public constant WIPEOUT_CUTOFF = 0.95e18;
```
