# Smart contract best practices

**Notice: this translation was generously provided by a contributor. The maintainers are not able to verify the content. Any issues or PRs to help improve it are welcome.**

Bài viết này được dịch nguyên văn từ https://consensys.github.io/smart-contract-best-practices. Để phù hợp với các diễn đạt văn phong tiếng việt, chúng tôi cố gắng diễn đạt tư tưởng của tài liệu chứ không dịch theo từng chữ một.

- [**Những lời khuyến nghị cho việc phát triển hợp đồng thông minh bằng Solidity**](#solidity-tips)
- [**Hiểu biết về các phương thức tấn công phổ biến**](#known-attacks)
- [**Áp dụng các nguyên tắc trong phát triển phần mềm để viết hợp đồng thông minh**](#eng-techniques)
- [**Lời khuyên cho việc implement mã token**](#token)
- [**Tài liệu và thủ tục**](#document)
- [**Các công cụ bảo mật**](#tools)
- [**EIPs**](#eip)
- [**Các tài nguyên tham khảo**](#resource)

<a name="solidity-tips"></a>

# Những khuyến nghị cho việc phát triển hợp đồng thông minh an toàn bằng solidity

### Lời gọi ngoài (External Calls)

#### Hãy thật cẩn trọng khi sử dụng external calls

Các message gọi đến những hợp đồng không đáng tin cậy có thể gây ra một số rủi ro hoặc lỗi không mong muốn. Các lời gọi ngoài có thể thực thi mã độc trong hợp đồng đó hoặc bất kỳ hợp đồng nào khác mà nó phụ thuộc vào. Như vậy, mọi lời gọi ngoài nên được xem là ẩn chứa rủi ro bảo mật. Trong trường hợp bất khả kháng, hãy sử dụng các đề xuất dưới đây để giảm thiểu rủi ro có thể xảy ra.

#### Đánh dấu các hợp đồng không đáng tin cậy

Khi tương tác với các lời gọi ngoài, tên các biến, phương thức và các interface nên được đặt sao cho nó thể hiện được việc tương tác với các lời gọi từ bên ngoài có an toàn hay là không ? Điều này áp dụng cho các hàm mà nó có thể được gọi từ các hợp đồng bên ngoài.

```javascript
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

function makeWithdrawal(uint amount) { // Isn't clear that this function is potentially unsafe
    Bank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // untrusted external call
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

#### Tránh sử dụng `transfer()` và `send()`

`.transfer()` và `.send()` giới hạn chính xác 2.300 gas lời gọi. Mục tiêu của quy định nhằm để ngăn ngừa các lỗ hổng reentrancy, nhưng điều này chỉ có ý nghĩa theo giả định rằng chi phí gas là không đổi. Gần đây EIP 1283 (được hỗ trợ từ hard fork Constantinople vào phút cuối) và EIP 1884 (dự kiến ​​sẽ đến hard fork Istanbul) sẽ làm cho việc gửi tiền bằng hai hàm này không thực sự an toàn nữa.

Để tránh mọi thứ bị đổ bể khi chi phí gas thay đổi trong tương lai, tốt nhất nên sử dụng `.call.value (số tiền) ("")` để thay thế. Lưu ý rằng điều này không đảm bảo giảm thiểu các cuộc tấn công reentrancy, mà cần kết hợp với các biện pháp khác.

#### Sự khác nhau giữa send(), transfer() và call.value()

Khi thực hiện một giao dịch từ hợp đồng thông minh, cần phân biệt sự giống và khác giữa `someAddress.send()`, `someAddress.transfer()`, `someAddress.call().value()`.

- `someAddress.send()` và `someAddress.transfer()` được coi là an toàn để chống lại reentrancy. Chúng giới hạn 2.300 gas, chỉ đủ để ghi lại một sự kiện thay vì chạy một đoạn mã khai thác.

- x.transfer(y) tương đương với lệnh x.send (y), nó sẽ tự động revert nếu giao dịch thất bại.

- Khác với `someAddress.send()` và `someAddress.transfer()`, `someAddress.call.value(y) `không giới hạn gas cho lời gọi và do đó hacker có thể thực thi lời gọi đến một đoạn mã độc nhằm mục đích xấu. Do đó, nó không an toàn để chống lại reentrancy.

Sử dụng send() hoặc transfer() sẽ ngăn chặn reentrancy nhưng nó sẽ không thích hợp với các hợp đồng mà fallback function yêu cầu hơn 2.300 gas. Chúng ta cũng có thể sử dụng someAddress.call.value(ethAmount) .gas(gasAmount) để giới hạn lượng gas cho lời gọi một cách tùy ý.

#### Xử lý lỗi từ các lời gọi ngoài

Solidity cung cấp các phương thức gọi mức thấp (low level) : `address.call()`, `address.callcode()`, `address.delegatecall()` và `address.send()`. Các phương thức ở mức thấp này không bao giờ ném ra ngoại lệ (throw an exception), nhưng sẽ trả về false nếu lời gọi gặp phải ngoại lệ. Mặt khác, các lời gọi hợp đồng (contract calls) (ví dụ như `ExternalContract.doSomething()`) sẽ tự động ném ra một ngoại lệ và báo lỗi.

Nếu bạn lựa chọn sử dụng các phương thức gọi ở mức thấp, hãy kiểm tra xem lời gọi sẽ thất bại hay thành công, bằng cách kiểm tra giá trị trả về là `true` hay` false`.

```javascript
// bad
someAddress.send(55);
someAddress.call.value(55)(""); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3('deposit()'))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted

(bool success, ) = someAddress.call.value(55)("");
if(!success) {
    // handle failure code
}

ExternalContract(someAddress).deposit.value(100)();
```

#### Ưu tiên pull hơn là push cho các lời gọi ngoài

Các lời gọi từ bên ngoài có thể thất bại một cách vô tình hoặc cố ý. Để giảm thiểu rủi ro từ các lỗi đó gây ra, tốt hơn hết là chia từng lời gọi thành các lời gọi nhỏ hơn. Điều này đặc biệt phù hợp với các giao dịch thanh toán, trong đó cho phép người dùng rút tiền sẽ tốt hơn là tự động chuyển tiền cho họ. (Điều này cũng làm giảm khả năng xảy ra sự cố với gasLimit.) và tránh việc thực hiện cùng một lúc nhiều hàm transfer() trong một giao dịch.

```javascript
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            (bool success, ) = highestBidder.call.value(highestBid)("");
            require(success); // if this call consistently fails, no one else can bid
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        (bool success, ) = msg.sender.call.value(refund)("");
        require(success);
    }
}
```

#### Không nên dùng delegatecall với đoạn mã không được tin cậy

Hàm `delegatecall` được sử dụng để gọi các hàm từ các hợp đồng khác như thể chúng thuộc về hợp đồng của người gọi. Do đó, người gọi có thể thay đổi trạng thái của hợp đồng được gọi đến. Điều này có thể không an toàn. Ví dụ dưới đây cho thấy cách sử dụng `delegatecall` có thể dẫn đến việc hợp đồng bị phá hủy và mất hết số dư.

```javascript
contract Destructor
{
    function doWork() external
    {
        selfdestruct(0);
    }
}

contract Worker
{
    function doWork(address _internalWorker) public
    {
        // unsafe
        _internalWorker.delegatecall(bytes4(keccak256("doWork()")));
    }
}
```

Nếu `worker.doWork()` được gọi với tham số là địa chỉ của hợp đồng Destructor, hợp đồng Worker sẽ tự hủy. Bạn chỉ nên thực hiện delegate call cho các hợp đồng đáng tin cậy.

**Lưu ý**: Đừng cho rằng các hợp đồng khi được khởi tạo có số dư bằng 0. Một kẻ tấn công có thể gửi ether đến địa chỉ của hợp đồng trước khi nó được khởi tạo. [Xem vấn đề 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61) để biết thêm chi tiết.

### Ether có thể được gửi đến bất kỳ hợp đồng nào

Kẻ tấn công có thể gửi ether đến bất kỳ tài khoản nào và điều này không thể ngăn chặn được (ngay cả với fallback function với câu lệnh revert).

Kẻ tấn công có thể làm điều này bằng cách tạo ra một hợp đồng, gửi cho nó 1 wei và hàm `selfdestruct(victimAddress`), ở đây `victimAddress` là địa chỉ hợp đồng cần gửi ether vào.

### Hãy nhớ rằng Ethereum là mạng public blockchain, mọi dữ liệu trên các block đều được công khai

Nhiều ứng dụng yêu cầu dữ liệu được gửi phải ở chế độ riêng tư cho đến một lúc nào đó. Các trò chơi (ví dụ: oản tù tì) và việc đấu giá kín là hai ví dụ chính. Nếu bạn đang xây dựng một ứng dụng mà sự riêng tư là một vấn đề, hãy đảm bảo bạn tránh yêu cầu người dùng công khai thông tin quá sớm. Chiến lược tốt nhất là chia thành các giai đoạn riêng biệt: đầu tiên thì sử dụng hàm băm của các giá trị và trong giai đoạn tiếp theo thì tiết lộ các giá trị.

Ví dụ:

- Trong trò chơi oản tù tì, yêu cầu cả hai người chơi gửi giá trị băm của kéo, đá hay giấy (do người chơi quyết định), sau đó trò chơi yêu cầu cả hai người chơi gửi kết quả mình lựa chọn. Tiếp đó so sánh giá trị băm, nếu khớp thì hợp lệ, trò chơi sẽ phân thắng hòa hay thua dựa trên kết quả chọn của 2 người chơi.

- Trong phiên đấu giá kín, yêu cầu người đấu giá gửi giá trị băm mức giá mà họ chọn trong giai đoạn ban đầu (cùng với khoản tiền gửi lớn hơn giá trị giá thầu của họ), sau đó gửi giá trị đấu giá của họ trong giai đoạn thứ hai.

- Khi phát triển một ứng dụng mang tính ngẫu nhiên, thứ tự phải luôn là: (1) người chơi submit, (2) số ngẫu nhiên được tạo, (3) người chơi hoàn thành giao dịch. Phương thức mà các số ngẫu nhiên được tạo ra là cả một lĩnh vực nghiên cứu, các giải pháp tốt nhất hiện tại bao gồm việc sử dụng block header Bitcoin (được xác minh thông qua http://btcrelay.org), các cơ chế hash-commit-reveal (tức là một bên tạo ra một số, xuất bản hàm băm của nó để "cam kết" và sau đó tiết lộ giá trị sau), cùng với đó là [RANDAO](https://github.com/randao/randao). Vì Ethereum là một giao thức xác định, không có biến nào trong giao thức có thể được sử dụng như một số ngẫu nhiên không thể đoán trước. Ngoài ra, hãy lưu ý rằng thợ đào (miner) trong một chừng mực nào đó kiểm soát giá trị block.blockhash().

### Cảnh giác với khả năng một số người tham gia có thể "drop offline" và không quay lại

Ví dụ, trong trò chơi oản tù tì, một ván đấu được tiếp tục cho đến khi cả hai người chơi gửi lựa chọn của họ. Tuy nhiên, một người chơi có thể không bao giờ gửi lựa chọn của họ - thực tế, nếu một người chơi thấy động thái được tiết lộ từ người chơi khác và xác định rằng họ đã thua, họ không có lý do gì để tự gửi kết quả. Khi gặp các tình huống như vậy thì , (1) cung cấp một cách để tránh những người chơi không tham gia, có thể giới hạn thời gian và (2) xem xét thêm lợi ích bổ sung cho những người tham gia khi gửi kết quả trong tất cả các tình huống.

### Trường hợp đổi dấu số âm bé nhất

Solidity cung cấp một số kiểu dữ liệu số nguyên. Giống như trong hầu hết các ngôn ngữ lập trình khác, trong Solidity, một số nguyên N bit có thể biểu thị các giá trị từ `-2 ^ (N-1) đến 2 ^ (N-1) - 1`. Điều này có nghĩa là không có giá trị dương mà có trị tuyệt đối bằng `MIN_INT`. `- MIN_INT` sẽ được gắn bằng `MIN_INT`.

Điều này đúng với tất cả các kiểu số nguyên trong Solidity (int8, int16, ..., int256).

```javascript
contract Negation {
    function negate8(int8 _i) public pure returns(int8) {
        return -_i;
    }

    function negate16(int16 _i) public pure returns(int16) {
        return -_i;
    }

    int8 public a = negate8(-128); // -128
    int16 public b = negate16(-128); // 128
    int16 public c = negate16(-32768); // -32768
}
}

```

Một cách để xử lý điều này là kiểm tra giá trị của biến trước khi đảo dấu và ném ra ngoại lệ nếu nó bằng `MIN_INT`. Một tùy chọn khác là đảm bảo rằng số âm nhất bé nhất sẽ không bao giờ đạt được bằng cách sử kiểu biến có khoảng giá trị lớn (ví dụ: int32 thay vì int16).

### Sử dụng assert(), revert() và require() đúng cách

Các hàm assert và require được sử dụng để kiểm tra các điều kiện và ném ra một ngoại lệ nếu điều kiện không được đáp ứng.

Hàm `assert` chỉ nên được sử dụng để kiểm tra các lỗi bên trong (internal error) và các biến hằng.

Hàm `require` nên được dùng để đảm bảo các điều kiện hợp lệ, chẳng hạn như biến đầu vào, biến trạng thái của hợp đồng hoặc để xác thực giá trị trả về từ các lời gọi đến hợp đồng bên ngoài.

Ví dụ dưới đây cho thấy rằng các opcode không hợp lệ không có cơ hội để thực thi: các biến đều được xác minh và nếu có sai số thì đoạn mã sẽ ném ra lỗi.

```javascript
pragma solidity ^0.5.0;

contract Sharer {
    function sendHalf(address payable addr) public payable returns (uint balance) {
        require(msg.value % 2 == 0, "Even value required."); //Require() can have an optional message string
        uint balanceBeforeTransfer = address(this).balance;
        addr.transfer(msg.value / 2);
        // Since transfer throws an exception on failure and
        // cannot call back here, there should be no way for us to
        // still have half of the money.
        assert(address(this).balance == balanceBeforeTransfer - msg.value / 2); // used for internal error checking
        return address(this).balance;
    }
}
```

### Chỉ sử modifier khi cần kiểm tra dữ liệu

Mã bên trong modifier được thực thi trước khi chạy mã bên trong hàm. Do đó, bất kỳ thay đổi trạng thái hoặc lời gọi ngoài nào được tạo ra bởi đoạn mã trong modifier cũng sẽ vi phạm thiết kế [Checks-Effects-Interactions](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) mà chúng tôi đã đề ra. Ví dụ dưới đây, một lời gọi ngoài hợp đồng được chèn trong modifier có thể dẫn đến lỗ hổng reentrancy.

```javascript
contract Registry {
    address owner;

    function isVoter(address _addr) external returns(bool) {
        // Code
    }
}

contract Election {
    Registry registry;

    modifier isEligible(address _addr) {
        require(registry.isVoter(_addr));
        _;
    }

    function vote() isEligible(msg.sender) public {
        // Code
    }
}
```

Trong trường hợp này, hợp đồng Registry có thể bị tấn công reentracy bằng cách gọi đến Election.vote()

**Lưu ý**: Sử dụng modifier để thay các câu lệnh kiểm tra điều kiện bên trong thân của hàm. Điều này làm cho mã nguồn hợp đồng thông minh của bạn gọn nhẹ và dễ đọc hơn.

### Hãy cẩn thận với việc làm tròn kết quả trong phép chia

Tất cả các phép chia số nguyên được làm tròn bằng cách lấy số nguyên gần nhất. Nếu bạn cần độ chính xác cao hơn, hãy cân nhắc lưu trữ cả tử và mẫu số, hoặc số nhân vào một biến trung gian nào đó.

(Trong tương lai, Solidity sẽ có [fixed_point type](https://solidity.readthedocs.io/en/develop/types.html#fixed-point-numbers), điều này sẽ khiến vấn đề dễ dàng hơn.)

```javascript
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer
```

Sử dụng số nhân sẽ ngăn việc làm tròn xuống, số nhân này cần được tính toán khi làm việc với x trong tương lai:

```javascript
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

Lưu trữ tử số và mẫu số có nghĩa là bạn có thể tính kết quả của tử số/mẫu số ngoài chuỗi:

```javascript
// good
uint numerator = 5;
uint denominator = 2;
```

### Abstract contract và interfaces

Cả interface và hợp đồng trừu tượng (abstract contract) đều có chung một tư tưởng là cho phép tùy chỉnh mã nguồn của các function dựa trên prototype có sẵn. Interface, được giới thiệu trong phiên bản Solidity 0.4.11, tương tự như các abstract contract nhưng interface chỉ có các prototype mà không có chứa thân hàm. Interface cũng có những hạn chế như không thể truy cập vào storage hoặc kế thừa từ các interface khác, điều này làm cho các abstract contract có ưu thế hơn một chút. Ngoài ra, điều quan trọng cần lưu ý là nếu một hợp đồng kế thừa từ một abstract contract thì các hàm sẽ được thực thi bằng cách ghi đè (overriding).

### Fallback functions

#### Giữ cho Fallback function đơn giản

[Fallback function](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) được thực thi khi hợp đồng được gọi bởi một message không có tham số (hoặc message đó gọi đến một hàm không tồn tại trong hợp đồng). Nếu bạn chỉ muốn nhận Ether từ fallback function bằng cách gọi `.send()` hoặc `.transfer()`, thì 2300 gas đủ để cho bạn kích hoạt một event. Nếu cần sử nhiều tính toán hơn thì có thể cấu hình lượng gas tối đa mà fallback function có thể sử dụng.

```javascript
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### Kiểm tra độ dài dữ liệu trong fallback function

Fallback function không chỉ được dùng để nhận ether gửi vào hợp đồng (không có dữ liệu trong message) mà còn dùng kh gọi hàm không có trong hợp đồng hoặc tham số không đúng. Do đó, kiểm tra độ dài data trước khi thực thi các mã trong fallback function nhằm tránh việc bị thực thi mã độc.

```javascript
// bad
function() payable { emit LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### Định nghĩa rõ ràng các hàm  và các biến có thể nhận ether

Bắt đầu từ phiên bản Solidity 0.4.0, mọi hàm nhận ether phải có modifier `payable`, mặt khác, nếu lời gọi đến hàm payable có msg.value = 0 thì giao dịch sẽ bị revert ([trừ khi bị bắt buộc](https://consensys.github.io/smart-contract-best-practices/recommendations/#remember-that-ether-can-be-forcibly-sent-to-an-account)).

Nếu bạn muốn dùng chức năng chuyển tiền, hãy khai báo các biến và các tham số của hàm có dạng `address payable`. Bạn chỉ có thể sử dụng .transfer (..) và .send (..) trên `address payable` thay vì `address`. Bạn có thể sử dụng .call (..) cho cả `address payable` và `address`. Nhưng điều này không được khuyến khích.

**Lưu ý**: Modifier payable chỉ áp dụng cho các lời gọi từ bên ngoài. Nếu một hàm non-payable gọi hàm payable trên cùng một hợp đồng, thì hàm non-payable sẽ không thành công, mặc dù msg.value vẫn được đặt.

### Định nghĩa rõ ràng phạm vi truy cập của các hàm, các biến

Các hàm có bốn loại phạm vi truy cập là external, public, private, internal. . Đối với các biến, không thể định nghĩa phạm vi external. Định nghĩa đầy đủ, rõ ràng phạm vi truy cập của các biến, các hàm giúp dễ dàng nắm được được phạm vi của từng thành phần trong hợp đồng, tránh các lỗi không đáng có.

- Các hàm `external` là một phần chức năng của contract interface. Các hàm external hiểu quả hơn các hàm public khi tham số là các mảng dữ liệu lớn do hàm external sẽ tốn ít gas hơn.
- Các hàm `public` có thể được gọi từ bất cứ đâu, trong hợp đồng, hoặc từ một hợp đồng khác.
- Các hàm `internal` chỉ có thể được gọi từ bên trong hợp đồng hoặc các hợp đồng kế thừa.
- Các hàm `private` chỉ có thể được gọi từ bên trong hợp đồng.

```javascript
// bad
uint x; // the default is internal for state variables, but it should be made explicit
function buy() { // the default is public
    // public code
}

// good
uint private y;
function buy() external {
    // only callable externally or using this.buy()
}

function utility() public {
    // callable externally, as well as internally: changing this code requires thinking about both cases.
}

function internalAction() internal {
    // internal code
}
```

### Fix cứng phiên bản trình biên dịch của Solidity

Các hợp đồng nên được triển khai với cùng phiên bản trình biên dịch mà chúng đã được kiểm thử nhiều nhất. Fix cứng phiên bản trình biên dịch để tránh trường hợp được triển khai bởi một phiên bản mới hơn (vốn có thể tiềm ẩn lỗi).

```
// bad
pragma solidity ^0.4.4;

// good
pragma solidity 0.4.4;
```

### Sử dụng các sự kiện (event) để theo dõi hoạt động của hợp đồng

Cần có cách giám sát hoạt động của hợp đồng sau khi nó được triển khai. Một cách để thực hiện điều này là xem xét tất cả các giao dịch của hợp đồng, tuy nhiên điều đó là chưa đủ, vì các lời gọi (message call) giữa các hợp đồng không được ghi lại trên blockchain. Hơn nữa, nó cũng chỉ hiển thị các tham số đầu vào, không phải là những thay đổi trạng thái. Ngoài việc theo dõi hoạt động của hợp đồng ra, các sự kiện có thể được sử dụng để tương tác với giao diện người dùng.

```javascript
contract Charity {
    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

Ở trên, hợp đồng Game sẽ thực hiện lời gọi (internal call) đến Charity.donate(). Giao dịch này sẽ không xuất hiện trong danh sách các giao dịch bên ngoài (external transaction) của hợp đồng Charity mà có trong danh sách các giao dịch nội bộ (internal transaction).

Sự kiện là cách thuận tiện để ghi lại một điều gì đó đã xảy ra trong hợp đồng. Các sự kiện được phát ra (emit) được lưu lại trong blockchain cùng với dữ liệu khác của hợp đồng. Đây là một cải tiến cho ví dụ ở trên, chúng ta sử dụng sự kiện để ghi lại lịch sử quyên góp của Hội từ thiện.

```javascript
contract Charity {
    // define event
    event LogDonate(uint _amount);

    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
        // emit event
        emit LogDonate(msg.value);
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

Tất cả các giao dịch gọi hàm donate của hợp đồng Charity, dù trực tiếp hay không, sẽ hiển thị trong danh sách sự kiện của hợp đồng đó cùng với số tiền quyên góp.

### Sự phức tạp của ngôn từ (Lườm rau gắp thịt)

Khi bạn định nghĩa tên một hàm, nếu nó trùng với tên các hàm có sẵn của Solidity. Nó sẽ ghi đè (override) hàm mặc định nhưng nó sẽ gây hiểu nhầm cho người đọc với ý nghĩa của đoạn mã.

```javascript
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

```javascript
contract FakingItsOwnDeath {
    function selfdestruct(address x) internal constant {}
}

contract SelfDestructive is FakingItsOwnDeath {
    function die() public {
        selfdestruct(address(0x0));
    }
}
```

Ở ví dụ thứ nhất, hàm `revert()` được gọi là hàm revert của hợp đồng `PretendingToRevert` thay vì hàm `revert()` mặc định. Do đó, khi gọi hàm `someThingBad()` của Example thì nó vẫn hoạt động bình thường.

Tương tự ở ví dụ thứ hai, không có hợp đồng nào bị hủy cả khi gọi đến hàm `die()` của SelfDestructive.

### Tránh sử dụng tx.origin

tx.origin chỉ có thể là tài khoản người dùng, không thể là tài khoản hợp đồng.
msg.sender có thể là tài khoản người dùng hoặc tài khoản hợp đồng.

Ví dụ các lời gọi theo chuỗi như sau: A->B->C->D, một hàm của hợp đồng D được gọi thì msg.sender là địa chỉ của C còn tx.origin là địa chỉ của A.

```javascript
contract MyContract {

    address owner;

    function MyContract() public {
        owner = msg.sender;
    }

    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        receiver.transfer(amount);
    }

}

contract AttackingContract {

    MyContract myContract;
    address attacker;

    function AttackingContract(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }

    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }

}
```

Đoạn mã ở trên tận dụng đặc điểm của `tx.orign` để chuyển tiền từ hợp đồng `MyContract` vào tài khoản của kẻ xấu bằng cách viết hợp đồng `AttackingContract`và gọi đến hàm `sendTo` trong `MyContract`.

Khả năng trong tương lai, `tx.origin` sẽ bị loại bỏ khỏi nền tảng Ethereum. Chính nhà đồng sáng lập Ethereum Vatalik Buterin cho rằng tx.orgin không có ý nghĩa để có thể sử dụng trong hợp đồng thông minh.

Điều đáng nói là bằng cách sử dụng tx.origin, bạn sẽ hạn chế khả năng tương tác giữa các hợp đồng vì hợp đồng sử dụng tx.origin không thể được sử dụng bởi một hợp đồng khác vì tài khoản hợp đồng không thể là tx.origin.

### Nhãn thời gian (timestamp)

Có ba điều cần lưu ý khi sử dụng nhãn thời gian để thực hiện các chức năng quan trọng trong hợp đồng thông minh, đặc biệt là khi các hành động liên có quan đến việc chuyển tiền.

#### Thao tác với nhãn thời gian

Cần lưu ý rằng nhãn thời gian của một block có thể được tác động bởi thợ đào. Chúng ta cùng xem xét ví dụ sau đây:

```javascript
uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```

Khi hợp đồng sử dụng nhãn thời gian để tạo số ngẫu nhiên, người thợ đào thực sự có thể đóng nhãn thời gian trong vòng 15 giây khi block đang được xác thực, cho phép người thợ đào có thể tính toán trước các tùy chọn. Nhãn thời gian không phải là ngẫu nhiên và không nên được sử dụng trong bối cảnh đó.

#### Quy tắc 15 giây

Trong Yellow Paper của Ethereum không mô tả về số lượng block có thể tạo ra trong một khoảng thời gian nhất định, nhưng nó đề cập rằng mỗi nhãn thời gian của block con phải lớn hơn nhãn thời gian của block cha mẹ. Các giao thức Ethereum trên Geth và Parity đều từ chối các block với nhãn thời gian lớn hơn 15 giây so với cha mẹ nó. Do đó, một nguyên tắc nhỏ trong việc đánh giá việc sử dụng nhãn thời gian là:

**Nếu sự kiện hợp đồng bạn triển khai có thể thay đổi trong 15 giây và duy trì tính toàn vẹn, thì việc sử dụng block.timestamp là an toàn.**

#### Tránh sử dụng block.number như là nhãn thời gian

Có thể ước tính một khoảng thời gian bằng cách sử dụng thuộc tính block.number và [thời gian khối trung bình](https://etherscan.io/chart/blocktime), tuy nhiên đây không phải là cách hay vì thời gian một block mới được tạo mới (block times) có thể thay đổi (ví dụ như việc xảy ra [fork reorganisations](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/) hoặc thay đổi [difficulty bomb](https://github.com/ethereum/EIPs/issues/649)).

### Hãy thận trọng khi sử dụng tính năng đa kế thừa

Khi sử dụng đa kế thừa trong Solidity, điều quan trọng là phải hiểu cách nó hoạt động như thế nào.

```javascript
contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;

    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;

    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```

Khi một hợp đồng được triển khai, trình biên dịch sẽ tuyến tính hóa sự kế thừa từ phải sang trái (sau từ khóa `is` là danh sách các hợp đồng cha mẹ được liệt kê).

#### Final \<- B \<- C \<- A

Hàm khởi tạo của hợp đồng A sẽ trả về 5, vì C là gần A nhất theo sự tuyết tính hóa từ phải qua trái.

Để biết thêm về bảo mật và kế thừa, hãy xem [bài viết này](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/)

Để giúp đóng góp, Github của Solidity có một [dự án](https://github.com/ethereum/solidity/projects/9#card-8027020) với tất cả các vấn đề liên quan đến thừa kế.

### Sử dụng interface thay vì address

Khi một hàm có tham số đầu vào là địa chỉ của một hợp đồng, tốt hơn là nên truyền vào interface hoặc một tham chiếu đến hợp đồng đó thay vì truyền vào địa chỉ của hợp đồng.

```javascript
contract Validator {
    function validate(uint) external returns(bool);
}

contract TypeSafeAuction {
    // good
    function validateBet(Validator _validator, uint _value) internal returns(bool) {
        bool valid = _validator.validate(_value);
        return valid;
    }
}

contract TypeUnsafeAuction {
    // bad
    function validateBet(address _addr, uint _value) internal returns(bool) {
        Validator validator = Validator(_addr);
        bool valid = validator.validate(_value);
        return valid;
    }
}
```

Những lợi ích của việc sử dụng hợp đồng `TypeSafeAuction` ở trên có thể được nhìn thấy từ ví dụ dưới đây. Nếu hàm `validateBet()` có tham số đầu vào là địa chỉ của hợp đồng, hoặc tham chiếu của một hợp đồng không phải là `TypeSafeAuction` thì trình biên dịch ném ra lỗi.

```javascript
contract NonValidator{}

contract Auction is TypeSafeAuction {
    NonValidator nonValidator;

    function bet(uint _value) {
        bool valid = validateBet(nonValidator, _value); // TypeError: Invalid type for argument in function call.
                                                        // Invalid implicit conversion from contract NonValidator
                                                        // to contract Validator requested.
    }
}
```

### Tránh sử dụng extcodesize để kiểm tra tài khoản người dùng (Externally Owned Accounts)

Modifier dưới đây có chức năng kiểm tra xem message gọi đến là tài khoản hợp đồng hay tài khoản người dùng.

```javascript
// bad
modifier isNotContract(address _a) {
  uint size;
  assembly {
    size := extcodesize(_a)
  }
    require(size == 0);
     _;
}
```

Ý tưởng rất đơn giản: nếu một địa chỉ có chứa mã nguồn, đó không phải là tài khoản người dùng mà là tài khoản hợp đồng. Tuy nhiên, một hợp đồng sẽ chưa bao gồm mã nguồn trong quá trình khởi tạo. Điều này có nghĩa là trong khi hàm contructor của hợp đồng đang được thực hiện, nó có thể thực hiện các lời gọi đến các hợp đồng khác với `extcodesize` trả về 0. Dưới đây là một ví dụ để làm rõ hơn.

```javascript
contract OnlyForEOA {    
    uint public flag;

    // bad
    modifier isNotContract(address _a){
        uint len;
        assembly { len := extcodesize(_a) }
        require(len == 0);
        _;
    }

    function setFlag(uint i) public isNotContract(msg.sender){
        flag = i;
    }
}

contract FakeEOA {
    constructor(address _a) public {
        OnlyForEOA c = OnlyForEOA(_a);
        c.setFlag(1);
    }
}
```

Đoạn mã trong hàm contructor của `FakeEOA` gọi đến hàm `setFlag` của `OnlyForEOA`, do hàm constructor của hợp đồng `FakeEOA` chưa được thực hiện xong, nên extcodesize của nó sẽ trả về 0 và vượt qua được bộ lọc của modifier `isNotContract` từ đó thay đổi giá trị flag trong` OnlyForEOA` một cách trái phép.

**Cảnh báo**\*:
Nếu mục tiêu của bạn là ngăn chặn các hợp đồng khác có thể gọi đến hợp đồng của bạn, thì việc dùng `extcodesize` có lẽ cũng là tương đối tốt. Một cách tiếp cận khác là kiểm tra giá trị của `(tx.origin == msg.sender)`, mặc dù điều này cũng có nhược điểm.

### Cẩn thận với phép chia cho 0 (Sodility \<0,4)

Trước phiên bản 0.4, Solidity [trả về 0](https://github.com/ethereum/solidity/issues/670) và không ném ngoại lệ khi một số được chia cho 0. Đảm bảo bạn đang chạy phiên bản solidity từ 0.4 trở lên.

### Cách đặt tên function và event (Solidity \<0.4.21)

Viết in hoa chữ đầu tiên tên của event, để ngăn ngừa rủi ro nhầm lẫn giữa các function và event. Đối với các function, luôn luôn bắt đầu bằng một chữ cái viết thường, ngoại trừ hàm khởi tạo (constructor).

<a name="known-attacks"></a>

# Hiểu về các phương thức tấn công phổ biến

Biết tấn công để biết cách phòng thủ là một trong những nguyên tắc cơ bản trong an toàn thông tin. Nội dung phần này đề cập đến các lỗ hổng, phương thức tấn công đã biết để có thể khai thác hợp đồng thông minh.

## Reentrancy

Một trong những mối nguy hiểm lớn khi gọi đến các hợp đồng bên ngoài là chúng có thể chiếm quyền điều khiển và thực hiện các thay đổi không mong muốn. Loại lỗi này có thể có nhiều biến thể và cả hai lỗi lớn dẫn vụ DAO hack nổi tiếng đều là các lỗi thuộc loại này.

### Reentrancy trên một hàm (Reentrancy on single function)

```javascript
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // At this point, the caller's code is executed, and can call withdrawBalance again
    require(success);
    userBalances[msg.sender] = 0;
}
```

Vì số dư tài khoản của người dùng không gắn bằng 0 cho đến khi kết thúc hàm, hacker có thể lợi dụng điều này để rút tiền nhiều lần bằng cách gọi liên tục hàm withdrawBalance để ngăn cản hàm đó chạy đến câu lệnh `userBalances[msg.sender] = 0; `

> DAO (Decentralized Autonomous Organization). Mục đích mà nó hướng đến là tự động hóa các quy tắc, bộ máy điều hành của một các tổ chức, từ đó loại bỏ vai trò của tài liệu và con người trong quá trình quản lý, tạo ra một cấu trúc với sự kiểm soát phi tập trung.
>
> Vào ngày 17 tháng 6 năm 2016, DAO đã bị hack và 3,6 triệu Ether (50 triệu đô la) đã bị đánh cắp bằng cách khai thác lỗ hổng Reetrancy.
>
> Ethereum Foundation đã phát hành một bản cập nhật quan trọng để khôi phục lại trạng thái trước vụ hack. Điều này dẫn đến việc Ethereum được chia thành Ethereum Classic và Ethereum.

Trong ví dụ trên, cách giảm thiểu tối đa rủi ro là sử dụng  hàm `send()` thay vì hàm `call.value ()`. Điều này sẽ hạn chế bất kỳ mã bên ngoài nào được thực thi.

Tuy nhiên, nếu bạn không thể tránh các lời gọi ngoài, thì cách đơn giản để ngăn chặn cuộc tấn công này là đảm bảo bạn không gọi thực hiện lời gọingoài trước khi các đoạn mã internal được thực hiện xong.

```javascript
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // The user's balance is already 0, so future invocations won't withdraw anything
    require(success);
}
```

### Reentrancy liên hàm (Cross-function Reentrancy)

Kẻ tấn công cũng có thể thực hiện một cuộc tấn công bằng cách sử dụng hai hay nhiều hàm khác nhau có cùng trạng thái.

```javascript
// INSECURE
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // At this point, the caller's code is executed, and can call transfer()
    require(success);
    userBalances[msg.sender] = 0;
}
```

Trong trường hợp này, kẻ tấn công sẽ gọi hàm transfer() khi chúng đang thực hiện lời gọi ngoài `.call.value(amountToWithdraw)("")` ở hàm `withdrawBalance`. Vì khi đó số dư của kẻ tấn công chưa được gán bằng 0, nên chúng có thể chuyển mã số dư đến một tài khoản khác mặc dù chúng đã nhận được tiền từ hàm withdrawBalance. Lỗ hổng này cũng đa được sử dụng trong cuộc tấn công vào DAO.

Lưu ý rằng trong ví dụ này, cả hai hàm đều cùng thuộc một hợp đồng. Tuy nhiên, lỗi có thể xảy ra trên nhiều hợp đồng, nếu các hợp đồng đó chia sẻ trạng thái với nhau.

### Pitfalls in Reentrancy Solutions

Reentrancy có thể xảy ra trên một chuỗi các hàm, thậm chí là chuỗi các hợp đồng. Cho nên, bất kỳ giải pháp nào nhằm ngăn chặn reentrancy chỉ với một hàm là không đủ.

Thay vào đó, chúng tôi khuyến nghị nên chạy hết các câu lệnh trong hợp đồng (internal work)  (ví dụ: thay đổi trạng thái các biến, mapping) trước việc thực hiện các lời gọi hàm bên ngoài (external call). Quy tắc này, nếu được tuân thủ cẩn thận, sẽ phòng chống được các lỗ hổng do reentrancy có thể gây ra. Tuy nhiên, bạn không chỉ cần tránh các lời gọi ngoài được thực thi quá sớm mà còn nên tránh việc các hàm bên trong hợp đồng gọi đến các hàm khác quá sớm (nên để các lời gọi hàm thực thi sau khi các đoạn mã logic khác trong thân hàm đã chạy qua). Ví dụ sau đây minh chứng cho việc viết mã không an toàn.

```javascript
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    (bool success, ) = recipient.call.value(amountToWithdraw)("");
    require(success);
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdrawReward(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

Mặc dù hàm `getFirstWithdrawalBonus()` không gọi thực hiện lời gọi ngoài, nhưng đoạn mã `withdrawReward(recipient)` gọi đến hàm `withdrawReward()` đã tạo ra lỗ hổng mà kẻ xấu có thể khai thác. Do đó, phải xem lời gọi đến hàm `withdrawReward()` trong hàm `getFirtWithdrawBonus()` là không đáng tin cậy và nên để nó được thực thi ở sau cùng trong hàm `getFirtWithdrawBonus()`.

```javascript
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    (bool success, ) = recipient.call.value(amountToWithdraw)("");
    require(success);
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdrawReward(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

Ngoài việc chuyển dòng code gọi đến hàm withdrawReward xuống cuối cùng trong hàm GetFirstWithdrawalBonus, chúng ta còn nên thay đổi tên hàm để đánh dấu nó là không đáng tin cậy.

Một giải pháp khác được đề xuất là [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion). Điều này cho phép bạn "khóa" một số trạng thái để chủ sở hữu khóa mới có thể thay đổi trạng thái. Một ví dụ đơn giản có thể trông như thế này:

```javascript
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    require(!lockBalances);
    lockBalances = true;
    balances[msg.sender] += msg.value;
    lockBalances = false;
    return true;
}

function withdraw(uint amount) payable public returns (bool) {
    require(!lockBalances && amount > 0 && balances[msg.sender] >= amount);
    lockBalances = true;

    (bool success, ) = msg.sender.call(amount)("");

    if (success) { // Normally insecure, but the mutex saves it
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

Nếu kẻ xấu cố gắng gọi hàm withdraw() một lần nữa trước khi câu lệnh `msg.sender.call(amount)()` kết thúc, khóa sẽ ngăn cản điều đó. Đây có thể là một giải pháp hiệu quả, nhưng nó trở nên khó khăn khi bạn có nhiều hợp đồng liên kết với nhau. Sau đây ví dụ minh chứng cho nhận định trên:

```javascript
// INSECURE
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == address(0));
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = address(0);
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

Kẻ tấn công có thể gọi hàm getLock(), và sẽ không bao giờ gọi đến releaseLock(). Nếu điều này xảy ra, thì hợp đồng sẽ bị khóa vĩnh viễn và không thể thay đổi trạng thái của lockHolder. Nếu bạn sử dụng mutexes để phòng chống reentrancy, bạn sẽ cần chắc chắn rằng sẽ không có trường hợp bị khóa cứng như đoạn mã ở trên. (Có những nguy cơ tiềm tàng khác khi lập trình với mutexes, chẳng hạn như deadlocks và livelocks).

## Front-Running (AKA Transaction-Ordering Dependence)

Ở phần trên chúng ta đã đề cập đến thức khai thác và phòng chống kiểu tấn công reentrancy. Bây giờ chúng ta sẽ cùng tìm hiểu một kiểu tấn công khác trong Blockchain dựa vào đặc điểm rằng: thứ tự của các giao dịch (ví dụ: trong một block) có thể bị kiểm soát.

Một giao dịch trước khi được xác minh được nằm trong mempool một thời gian ngắn, người ta có thể biết được những gì xảy ra với giao dịch trước khi nó được đưa vào một block. Điều này có thể gây rắc rối cho những sàn giao dịch phi tập trung, nơi mọi người có thể nhìn thấy tất cả các giao dịch. Để ngăn cản điều này thực sự rất khó. Ví dụ, tại các sàn giao dịch, sẽ tốt hơn nếu thực hiện đấu giá hàng loạt (điều này cũng bảo vệ chống lại việc phân tích giao dịch). Một cách khác là sử dụng sơ đồ pre-commit (Chúng tôi sẽ đi vào chi tiết ở phần sau).

## Tràn số nguyên (underfow và overflow)

Chúng ta hãy còn xem đoạn mã dưới đây:

```javascript
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] >= _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

Nếu giá trị `balanceOf[_to]` đạt đến giá trị lớn nhất có thể (2 ^ 256 - 1), chúng ta tăng `balanceOf[_to]` lên, nó sẽ quay vòng về giá trị 0. Khi thực hiện phép tính với các biến, hãy xem xem việc giá trị uint có cơ hội tiếp cận một số lớn như vậy hay không. Hãy xem rằng các biến uint có thể thay đổi giá trị bởi những ai. Nếu bất kỳ người dùng nào có thể gọi các hàm cập nhật giá trị uint, thì rất là nguy hiểm. Sẽ an toàn hơn nếu chỉ có quản trị viên có quyền truy cập để thay đổi trạng thái của biến. Nếu người dùng chỉ có thể tăng thêm 1 lần một, có lẽ bạn cũng an toàn vì khó có cách nào khả thi để đạt đến giới hạn số.

Tương tự với underflow. Nếu một uint bị giảm giá trị đến một số âm nhỏ hơn 0, nó sẽ quay vòng lại số lớn nhất của kiểu dữ liệu.

Một giải pháp đơn giản để giảm thiểu lỗi tràn số là sử dụng thư viện [SafeMath.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/math/SafeMath.sol) cho các hàm số học.

### Tìm hiểu chi tiết hơn về underflow

Bài dự thi của Doug Hoyte trong Solidity contest năm 2017 đã đề cập đến một vấn đề quan trọng. Đây là một phiên bản đơn giản hóa bài dự thi của Doug Hoyte:

```javascript
contract UnderflowManipulation {
    address public owner;
    uint256 public manipulateMe = 10;
    function UnderflowManipulation() {
        owner = msg.sender;
    }

    uint[] public bonusCodes;

    function pushBonusCode(uint code) {
        bonusCodes.push(code);
    }

    function popBonusCode()  {
        require(bonusCodes.length >=0);  // this is a tautology
        bonusCodes.length--; // an underflow can be caused here
    }

    function modifyBonusCode(uint index, uint update)  {
        require(index < bonusCodes.length);
        bonusCodes[index] = update; // write to any index less than bonusCodes.length
    }

}
```

Nhìn vào đoạn mã trên, thật khó để có thể gây tràn số dưới với biến `manipulateMe`. Tuy nhiên, với các mảng động được lưu trữ tuần tự, nên nếu một kẻ xấu muốn thay đổi thao tác, tất cả những gì họ cần làm là:

- Gọi `popBonusCode` để underflow (Lưu ý: phương thức `Array.pop()` đã được thêm vào trong phiên bản Solidity 0.5.0)
- Tính toán vị trí lưu trữ của biến  `manipulateMe`
- Sửa đổi và cập nhật giá trị của mảng bằng cách sử dụng `notifyBonusCode`

### Tấn công từ chối dịch vụ với revert

Chúng ta cùng xem một hợp đồng đấu giá đơn giản dưới đây

```javascript
// INSECURE
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // Refund the old leader, if it fails then revert

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
```

Nếu kẻ tấn công sử dụng một hợp đồng thông minh với hàm fallback có chức năng revert mọi giao dịch đến, kẻ tấn công có thể giành chiến thắng trong bất kỳ cuộc đấu giá nào. Có nghĩa là hacker sẽ gửi số tiền cao hơn số tiền hiện tại vào hàm `bid()` và trở thành leader, sau đó hắn đảm bảo rằng khi ai đó gửi số tiền lớn hơn, thì khi xảy ra giao dịch hoàn lại tiền cho hacker, nó đều sẽ không thành công.. Bằng cách này, chúng có thể ngăn bất kỳ ai khác gọi hàm `bid()` và chúng sẽ là leader mãi mãi. Lời khuyên ở đây là chúng ta sẽ chia thành 2 hàm gửi tiền và rút tiền, người dùng khi không là leader nữa thì sẽ gọi hàm rút tiền để thu lại số tiền đã gửi vào hàm gửi tiền.

Một ví dụ khác là khi  hợp đồng sử dụng vòng lặp để duyệt qua mảng nhằm trả tiền cho các người dùng (ví dụ: những người ủng hộ trong hợp đồng gây quỹ cộng đồng). Điều thông thường là muốn đảm bảo rằng mỗi khoản thanh toán thành công. Nếu không, giao dịch sẽ bị revert. Vấn đề là nếu giao dịch thất bại, bạn đang revert toàn bộ hệ thống thanh toán, nghĩa là vòng lặp sẽ không bao giờ được hoàn thành. Không ai được trả tiền vì một địa chỉ giao dịch bị lỗi.

```javascript
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // doubly bad, now a single failure on send will hold up all funds
    }
}
```

### Tấn công từ chối dịch vụ dựa với GasLimit

Mỗi block có giới hạn trên về lượng gas có thể được sử dụng, và do đó suy ra khối lượng tính toán có thể được thực hiện. Nếu lượng gas chi vượt quá giới hạn này, giao dịch sẽ thất bại. Điều này dẫn đến một vài nguy cơ có thể bị hacker tấn công từ chối dịch vụ.

### Gas Limit DoS on a Contract via Unbounded Operations

Một vấn đề khác với ví dụ trước: bằng cách thanh toán cho mọi người cùng một lúc, bạn có nguy cơ sử dụng quá lượng gas giới hạn.

Điều này có thể dẫn đến các vấn đề ngay cả khi không có một cuộc tấn công có chủ đích. Tuy nhiên, thật tệ nếu kẻ tấn công có thể thao túng lượng gas cần thiết. Trong trường hợp của ví dụ trước, kẻ tấn công có thể tạo một loạt địa chỉ, mỗi địa chỉ cần được hoàn lại một số tiền rất nhỏ. Do đó, chi phí gas cho việc hoàn trả tiền cho từng địa chỉ của kẻ tấn công có thể vượt lượng gas, ngăn chặn giao dịch hoàn tiền có thể xảy ra cho những người dùng khác.

Nếu bạn phải sử dụng vòng lặp để duyệt qua một mảng có kích thước không xác định, bạn nên để chúng diễn ra dàn trải trên nhiều block. Như trong ví dụ sau:

```javascript
struct Payee {
    address addr;
    uint256 value;
}

Payee[] payees;
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
      payees[i].addr.send(payees[i].value);
      i++;
    }
    nextPayeeIndex = i;
}
```

Bạn sẽ cần đảm bảo rằng sẽ không có gì xấu xảy ra nếu các giao dịch khác được xử lý trong khi chờ lần lặp tiếp theo của hàm `payOut()`. Vì vậy, chỉ sử dụng mô hình này nếu thực sự cần thiết.

### Gas Limit DoS on the Network via Block Stuffing

Ngay cả khi hợp đồng của bạn không chứa vòng lặp, kẻ tấn công có thể ngăn các giao dịch khác được đưa vào blockchain trong một vài block bằng cách đặt một số giao dịch với lượng gas cao.

Để làm được điều này, kẻ tấn công sẽ phát hành một số giao dịch sẽ với lượng tổng lượng gas tiêu thụ tương đương với gasLimit, với phí gas đủ cao thì những giao dịch đấy sẽ được đưa vào block mới nhất. Không có gì đảm bảo một giao dịch có phí gas cao sẽ được đưa vào block, nhưng giá càng cao, cơ hội càng cao.

Nếu cuộc tấn công thành công, sẽ không có giao dịch nào khác được đưa vào block. Đôi khi, mục tiêu của kẻ tấn công là ngăn chặn các giao dịch gọi đến một hợp đồng cụ thể trong một khoảng thời gian.

Cuộc tấn công này đã xảy ra trên [Fomo3D](https://solmaz.io/2018/10/18/anatomy-block-stuffing/), một ứng dụng đánh bạc. Trò chơi sẽ có một bộ đếm ngược thời gia, khi bộ đếm ngược chạy về 0, ai là người mua "chìa khóa" cuối cùng sẽ là người lãnh thưởng. Kẻ tấn công đã mua một khóa và sau đó chúng đã nhồi các giao dịch có phí gas cao trong 13 block liên tiếp cho đến khi bộ đếm thời gian hết giờ và khoản tiền thưởng được giải phóng. Các giao dịch được gửi bởi kẻ tấn công đã tốn mất 7,9 triệu gas trên mỗi block, do đó lượng gas còn lại chỉ cho phép một vài giao dịch nhỏ (mất 21.000 gas mỗi lần), nhưng không cho phép bất kỳ giao dịch nào gọi đến hàm buyKey() (tốn 300.000+ gas).

Một cuộc tấn công Block Stuffing có thể được sử dụng trên bất kỳ hợp đồng nào yêu cầu một hành động trong một khoảng thời gian nhất định. Tuy nhiên, như với bất kỳ cuộc tấn công nào, nó chỉ có lợi khi phần thưởng đạt được lớn hơn chi phí mà những kẻ tấn công phải bỏ ra. Chi phí của cuộc tấn công này tỷ lệ thuận với số block cần tấn công. Nếu hợp đồng của bạn gồm một khoản thanh toán lớn có thể có được bằng cách ngăn chặn các hành động từ những người dùng khác, thì nó có thể sẽ là mục tiêu của một cuộc tấn công như vậy.

## Insufficient gas griefing

Phương thức tấn công này có thể xảy ra đối với một hợp đồng chấp nhận sử dụng dữ liệu chung và nó thực hiện lời gọi một hợp trung gian (sub call) thông qua phương thức ở mức thấp `address.call()` .

Nếu một lời gọi thất bại, hợp đồng có 2 lựa chọn

- Revert toàn bộ giao dịch
- Tiếp tục thực thi

Ví dụ sau đây về lời gọi của hợp đồng Relayer sẽ tiếp tục thực thi bất kể kết quả của subcall:

```javascript
contract Relayer {
    mapping (bytes => bool) executed;

    function relay(bytes _data) public {
        // replay protection; do not call the same transaction twice
        require(executed[_data] == 0, "Duplicate call");
        executed[_data] = true;
        innerContract.call(bytes4(keccak256("execute(bytes)")), _data);
    }
}
```

Hợp đồng này cho phép chuyển tiếp giao dịch. Ai đó muốn thực hiện giao dịch nhưng không thể tự thực hiện giao dịch (ví dụ do thiếu ether để thanh toán tiền gas) có thể ký dữ liệu mà anh ta muốn chuyển và chuyển dữ liệu bằng chữ ký của mình. Sau đó, một "người chuyển tiếp" bên thứ ba có thể gửi giao dịch này tới mạng thay mặt cho người dùng.

Nếu chỉ được cung cấp đủ lượng gas, Relayer sẽ hoàn thành việc ghi tham số `_data` vào trong mapping \`\`excuted\`\`\`, nhưng subcall sẽ thất bại vì không nhận đủ gas để thực hiện xong.

Kẻ tấn công có thể sử dụng điều này để kiểm duyệt các giao dịch, khiến chúng thất bại bằng cách gửi chúng đi với một lượng gas thấp. Cuộc tấn công này không trực tiếp mang lại lợi ích cho kẻ tấn công, nhưng gây ra thiệt hại cho nạn nhân. Kẻ tấn công, sẵn sàng tiêu tốn một lượng khí nhỏ về mặt lý thuyết có thể kiểm duyệt tất cả các giao dịch theo cách này, nếu chúng là người đầu tiên gửi chúng cho Relayer.

Một cách để giải quyết vấn đề này là triển khai logic mã nguồn yêu cầu các hợp đồng cung cấp đủ gas để hoàn thành subcall. Nếu một thợ đào đã cố gắng tiến hành cuộc tấn công theo kịch bản này, câu lệnh `require` sẽ thất bại và giao dịch sẽ bị revert. Người dùng có thể chỉ định gasLimit tối thiểu cùng với dữ liệu khác (trong ví dụ này, thông thường, giá trị \_gasLimit sẽ được xác minh bằng chữ ký, nhưng điều đó được khuyến nghị vì đơn giản trong trường hợp này).

```javascript
// contract called by Relayer
contract Executor {
    function execute(bytes _data, uint _gasLimit) {
        require(gasleft() >= _gasLimit);
        ...
    }
}
```

Một giải pháp khác là chỉ cho phép các tài khoản đáng tin cậy chuyển tiếp giao dịch.

## Bắt buộc một hợp đồng phải nhận ether

Có thể gửi Ether vào một hợp đồng mà không cần kích hoạt fallback function của hợp đồng đó. Đây là một cân nhắc quan trọng khi viết mã cho fallback function hoặc thực hiện các tính toán dựa trên số dư của hợp đồng. Lấy ví dụ sau:

```javascript
contract Vulnerable {
    function () payable {
        revert();
    }

    function somethingBad() {
        require(this.balance > 0);
        // Do something bad
    }
}
```

Logic hợp đồng dường như không cho phép gửi ether vào hợp đồng. Tuy nhiên, một vài phương thức tồn tại để buộc hợp đồng nhận ether và làm cho số dư của nó lớn hơn 0.

Phương thức `sefldestruct`  cho phép chỉ định tài khoản nhận ether và không có cách gì để ngăn cản điều đó. `sefldestruct` không kích hoạt fallback function của hợp đồng.

## Các vụ hack lịch sử

Đây là những cuộc tấn công không còn có thể thực hiện do những thay đổi trong giao thức hoặc do các bản cập nhật solidity.

### Các lỗ hổng khác

[Smart Contract Weakness Classification Registry](https://smartcontractsecurity.github.io/SWC-registry/) cơ cung cấp một danh mục đầy đủ cập nhật về các lỗ hổng hợp đồng thông minh đã biết và các cách chống lại chúng với các ví dụ trong thế giới thực. Hãy thường xuyên xem qua là một cách tốt để cập nhật các cuộc tấn công mới nhất.

<a name="eng-techniques"></a>

# Áp dụng các nguyên tắc trong phát triển phần mềm để viết hợp đồng thông minh

Việc bảo vệ trước các cuộc tấn công đã biết là không đủ. Vì chi phí thiệt hại trên blockchain có thể rất cao, bạn cũng phải điều chỉnh cách bạn viết phần mềm, để phòng tránh những rủi ro đó.

Cách tiếp cận chúng tôi  là "chuẩn bị cho thất bại". Không thể biết trước được liệu mã nguồn của bạn có an toàn hay không ? Tuy nhiên, bạn có thể thiết kế các hợp đồng của mình theo cách cho phép chúng thất bại với thiệt hại tối thiểu. Phần này trình bày một loạt các kỹ thuật sẽ giúp bạn chuẩn bị cho thất bại.

*Lưu ý*: Luôn có rủi ro khi bạn thêm một thành phần mới vào hệ thống của mình. Hãy suy nghĩ kỹ về từng kỹ thuật bạn sử dụng trong các hợp đồng của mình và xem xét cẩn thận cách chúng phối hợp với nhau để tạo ra một hệ thống an toàn.

## Nâng cấp hợp đồng bị lỗi

Mã nguồn sẽ cần phải được thay đổi nếu phát hiện lỗi hoặc vì lý do cần cải thiện.

Thiết kế một hệ thống nâng cấp hiệu quả cho các hợp đồng thông minh là một lĩnh vực nghiên cứu và chúng tôi sẽ không thể đề cập đến tất cả các vấn đề trong tài liệu này được. Tuy nhiên, có hai cách tiếp cận cơ bản được sử dụng phổ biến nhất. Cách thứ nhất là tạo một hợp đồng giữ địa chỉ phiên bản mới nhất của các hợp đồng khác. Một cách tiếp cận khác là có một hợp đồng chuyển tiếp các lời gọi và dữ liệu đến phiên bản mới nhất của hợp đồng.

Dù là kỹ thuật nào, điều quan trọng là mã nguồn được mô đun hóa và phân tách tốt giữa các thành phần, để nếu một thay đổi xảy ra sẽ không phá vỡ cấu trúc của toàn bộ hệ thống.

Điều quan trọng là có một cách an toàn để các bên quyết định nâng cấp mã. Tùy thuộc vào hợp đồng của bạn, các thay đổi mã nguồn có thể cần được chấp thuận bởi một bên đáng tin cậy, một nhóm thành viên hoặc bỏ phiếu của toàn bộ các bên liên quan. Nếu quá trình này có thể mất một chút thời gian, bạn sẽ muốn xem xét liệu có cách nào khác để phản ứng nhanh hơn trong trường hợp bị tấn công, chẳng hạn như dừng khẩn cấp hoặc ngắt mạch.

### Ví dụ 1: Sử dụng hợp đồng để lưu trữ địa chỉ phiên bản mới nhất của các hợp đồng khác

```javascript
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner)
        _;
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

Có hai nhược điểm chính trong cách tiếp cận này:

1. Người dùng phải luôn luôn kiểm tra địa chỉ mới nhất của hợp đồng cần gọi đến và bất kỳ ai không thực hiện điều này có thể gặp rủi ro khi tương tác với các phiên bản cũ của hợp đồng.
1. Bạn sẽ cần suy nghĩ cẩn thận về cách xử lý dữ liệu hợp đồng khi bạn thay thế hợp đồng mới.

### Ví dụ 2: Sử dụng \`\`\`delegatecall\`\` cho việc chuyển tiếp lời gọi và dữ liệu đến hợp đồng.

```javascript
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        require(currentVersion.delegatecall(msg.data));
    }
}
```

Cách tiếp cận này tránh được các vấn đề ở ví dụ 1 ở trên nhưng cũng có vấn đề của riêng nó. Bạn phải cực kỳ cẩn thận với cách bạn lưu trữ dữ liệu trong hợp đồng này. Nếu hợp đồng mới của bạn có bố cục lưu trữ khác với hợp đồng đầu tiên, dữ liệu của bạn có thể bị hỏng. Ngoài ra, phiên bản đơn giản này của không thể trả về giá trị từ các hàm, chỉ chuyển tiếp chúng, làm hạn chế khả năng ứng dụng của nó.

## Bộ ngắt  (Tạm dừng chức năng của hợp đồng)

Bộ ngắt sẽ dừng việc thực thi nếu thỏa mãn một số điều kiện nhất định và nó có thể hữu ích khi phát hiện ra lỗi mới. Ví dụ: hầu hết các hành động có thể bị đình chỉ trong hợp đồng nếu phát hiện ra lỗi và hành động duy nhất hiện đang hoạt động là rút tiền. Bạn có thể cung cấp cho các bên đáng tin cậy khả năng kích hoạt bộ ngắt hoặc có thể lập trình để tự động hóa kích hoạt bộ ngắt khi gặp một số trường hợp nhất định.

```javascript
bool private stopped = false;
address private owner;

modifier isAdmin() {
    require(msg.sender == owner);
    _;
}

function toggleContractActive() isAdmin public {
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _; }
modifier onlyInEmergency { if (stopped) _; }

function deposit() stopInEmergency public {
    // some code
}

function withdraw() onlyInEmergency public {
    // some code
}
```

## Trì hoãn hành động của hợp đồng

Với việc trì hoãn các hành đồng từ hợp đồng, chúng ta sẽ có thêm thời gian để phục hồi hệ thống nếu bị tấn công. Ví dụ, đoạn mã ở dưới chỉ cho phép người dùng rút tiền sau 28 ngày kể từ lúc yêu câu rút tiền được gửi đến hợp đồng.

```javascript
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4 weeks

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
        uint amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0; // for simplicity, we withdraw everything;
        // presumably, the deposit function prevents new deposits when withdrawals are in progress

        requestedWithdrawals[msg.sender] = RequestedWithdrawal({
            amount: amountToWithdraw,
            time: now
        });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        require(msg.sender.send(amountToWithdraw));
    }
}
```

## Giới hạn tỷ lệ

Ví dụ: người gửi tiền chỉ có thể được phép rút một số tiền hoặc tỷ lệ phần trăm của tổng số tiền gửi trong một khoảng thời gian nhất định (ví dụ: tối đa 100 ether trong 1 ngày) - rút tiền bổ sung trong khoảng thời gian đó có thể thất bại hoặc hợp đồng sẽ yêu cầu một số phê duyệt đặc biệt. Hoặc ta giới hạn tỷ lệ có thể ở hợp đồng, chỉ với một số lượng mã token nhất định do hợp đồng phát hành trong một khoảng thời gian.

```javascript
contract CircuitBreaker {
  struct Transfer { 
      uint amount; 
      address to; 
      uint releaseBlock;
      bool released; 
      bool stopped; 
  }
  
  Transfer[] public transfers;

  address public curator;
  address public authorizedSender;
  uint public period;
  uint public limit;

  uint public currentPeriodEnd;
  uint public currentPeriodAmount;

  event PendingTransfer(uint id, uint amount, address to, uint releaseBlock);

  function CircuitBreaker(address _curator, address _authorizedSender, uint _period, uint _limit) {
    curator = _curator;
    period = _period;
    limit = _limit;
    authorizedSender = _authorizedSender;
    currentPeriodEnd = block.number + period;
  }

  function transfer(uint amount, address to) {
    if (msg.sender == authorizedSender) {
      updatePeriod();

      if (currentPeriodAmount + amount > limit) {
        uint releaseBlock = block.number + period;
        PendingTransfer(transfers.length, amount, to, releaseBlock);
        transfers.push(Transfer(amount, to, releaseBlock, false, false));
      } else {
        currentPeriodAmount += amount;
        transfers.push(Transfer(amount, to, block.number, true, false));
        if(!to.send(amount)) throw;
      }
    }
  } 
  
  function updatePeriod() {
    if (currentPeriodEnd < block.number) {
      currentPeriodEnd = block.number + period;
      currentPeriodAmount = 0;
    }
  }

  function releasePendingTransfer(uint id) {
    Transfer transfer = transfers[id];
    if (transfer.releaseBlock <= block.number && !transfer.released && !transfer.stopped) {
      transfer.released = true;
      if(!transfer.to.send(transfer.amount)) throw;
    }
  }
  
  function stopTransfer(uint id) {
    if (msg.sender == curator) {
      transfers[id].stopped = true;
    }
  }
}
```

## Triển khai hợp đồng

Hợp đồng nên có thời gian thử nghiệm - trước khi triển khai chính thức.

Tối thiểu, bạn nên:

- Kiểm thử đầy đủ với test coverage 100% (hoặc gần bằng)
- Deploy lên mạng thử nghiệm local (local testnet)
- Triển khai trên testnet public
- Triển khai trên mainnet ở phiên bản beta

#### AUTOMATIC DEPRECATION

Trong quá trình thử nghiệm, bạn có thể ngăn chặn mọi hành động, sau một khoảng thời gian nhất định. Ví dụ: hợp đồng alpha có thể hoạt động trong vài tuần và sau đó tự động tắt tất cả các hành động, ngoại trừ lần rút tiền cuối cùng.

```javascript
modifier isActive() {
    require(block.number <= SOME_BLOCK_NUMBER);
    _;
}

function deposit() public isActive {
    // some code
}

function withdraw() public {
    // some code
}
```

#### GIỚI HẠN SỐ TIỀN CỦA MỌI NGƯỜI DÙNG / HỢP ĐỒNG

Trong giai đoạn đầu, bạn có thể hạn chế lượng Ether cho bất kỳ người dùng nào (hoặc cho toàn bộ hợp đồng) - để thiểu giảm rủi ro.

### Trả thưởng cho những người tìm ra lỗi (Bug Bounty Program)

Các tips cho việc áp dụng việc trả thưởng cho người tìm ra lỗi:

- Quyết định xem loại tiền nào sẽ được dùng để trả thưởng (ETH hay BTC ...)
- Quyết định xem tổng ngân sách trả thưởng là bao nhiêu
- Từ ngân sách, xác định ba loại phần thưởng:
  - phần thưởng nhỏ nhất bạn sẵn sàng đưa ra
  - phần thưởng cao nhất được trao là bao nhiêu
  - một số điều khoản bổ sung nếu trong trường hợp lỗ hổng rất nghiêm trọng
- Xác định các chuyên gia để đánh giá mức độ nghiêm trọng của lỗ hổng
- Lead developer có thể là một trong các chuyên gia để đánh giá mức độ nghiêm trọng của lỗ hổng
- Khi nhận được báo cáo lỗ hổng, các chuyên gia sẽ đánh giá xem lỗ hổng có mức độ nghiêm trọng như thế nào
- Hỏi xem người săn lỗi có bản vá hay chưa ? Nếu chưa có thì đội phát triển cần đưa ra bản vá một cách nhanh chóng
- Trả thưởng cho người tìm ra lỗi

Các bạn có thể tham khảo chương trình của Ethereum [Ethereum's Bounty Program](https://bounty.ethereum.org/)

<a name="token"></a>

# Lời khuyên cho việc implement mã Token

## Tuân thủ tiêu chuẩn mới nhất

Nói chung, hợp đồng thông minh của mã token phải tuân theo tiêu chuẩn  đax được cộng đồng chấp nhận và xem là ổn định. Ví dụ về các tiêu chuẩn hiện được chấp nhận là:

- [EIP20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)
- [EIP721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md)

## Lưu ý về các cuộc tấn công với EIP-20

Hàm `approve()` của token EIP-20 có thể dẫn đến trường hợp người dùng có số tiền chi tiêu được phê duyệt chi tiêu nhiều hơn số tiền dự định. Một cuộc tấn công front-running có thể được sử dụng, cho phép một kẻ xấu gọi `transferFrom()` cả trước và sau khi gọi `approve()` được xử lý. Thông tin chi tiết có sẵn trên [EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#approve) và trong [tài liệu này](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit).

## Tránh việc chuyển token đến địa chỉ 0x0

Tại thời điểm viết, địa chỉ "không" [0x0000000000000000000000000000000000000000](https://etherscan.io/address/0x0000000000000000000000000000000000000000) giữ các mã token có giá trị hơn 80 triệu đô la !

## Tránh việc chuyển token đến chính hợp đồng của gửi token

Xem xét cũng ngăn chặn việc chuyển token đến chính địa chỉ của hợp đồng thông minh gửi token.

Một ví dụ về trường hợp này là hợp đồng thông minh [EOS](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0) nơi có hơn 90.000 token bị kẹt tại địa chỉ hợp đồng.

## Ví dụ

Một ví dụ về việc thực hiện cả hai khuyến nghị trên sẽ viết một modifier; xác thực rằng địa chỉ "đến" không phải là 0x0 cũng không phải là địa chỉ của hợp đồng thông minh:

```javascript
 modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }
```

Modifier  được áp dụng cho các phương thức "transfer" và "transferFrom":

```javascript
 function transfer(address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }
```

<a name="document"></a>

# Tài liệu và thủ tục

Khi triển khai một hợp đồng, điều quan trọng là phải bao gồm tài liệu thích hợp cho các biên liên quan có thể tìm hiểu. Một số tài liệu liên quan đến bảo mật bao gồm:

## Thông số kỹ thuật và kế hoạch triển khai

- Thông số kỹ thuật, sơ đồ, trạng thái, mô hình và tài liệu khác giúp người đánh giá và cộng đồng hiểu hệ thống dự định làm gì.
- Nhiều lỗi có thể được tìm thấy chỉ từ các thông số kỹ thuật, và chúng không tốn kém lắm để có thể sửa chữa.

## Trạng thái

- Nơi mã nguồn hiện tại được triển khai
- Phiên bản trình biên dịch, các bước để xác minh bytecode được triển khai khớp với mã nguồn
- Các phiên bản trình biên dịch sẽ được sử dụng cho các giai đoạn khác nhau.
- Trạng thái hiện tại của mã nguồn được triển khai (bao gồm các sự cố còn tồn tại, số liệu thống kê hiệu suất, v.v.)

## Nắm bắt được các vấn đề

- Ước lượng rủi ro chính với hợp đồng
- ví dụ: Bạn có thể mất tất cả tiền của mình, hacker có thể thực hiện một số hành đồng trái phép
- Nắm được tất cả các lỗi/giới hạn
- Các cuộc tấn công và giảm nhẹ thiệt hại

## Lịch sử

- Kiểm thử (bao gồm thống kê sử dụng, phát hiện lỗi, thời gian thử nghiệm)
- Những người đã review mã nguồn (và phản hồi chính của họ)

## Thủ tục

- Kế hoạch hành động trong trường hợp phát hiện ra lỗi (ví dụ: tùy chọn khẩn cấp, quy trình thông báo công khai, v.v.)
- Kết thúc quá trình nếu có sự cố xảy ra (ví dụ: các nhà tài trợ sẽ nhận được phần trăm số dư của bạn trước khi tấn công, từ số tiền còn lại)
- Chính sách công bố có trách nhiệm (ví dụ: nơi báo cáo lỗi được tìm thấy, các quy tắc của bất kỳ chương trình tiền thưởng lỗi nào)
- Phòng ngừa trong trường hợp thất bại (ví dụ: bảo hiểm, ...)

## Thông tin liên lạc

- Ai liên hệ được liên hệ khi xảy ra các vấn đề
- Phòng chat nơi câu hỏi có thể được hỏi

<a name="tools"></a>

# Các công cụ bảo mật

## Công cụ trực quan hóa

- [Sūrya](https://github.com/ConsenSys/surya)  - Công cụ tiện ích cho các hệ thống hợp đồng thông minh, cung cấp một số kết quả đầu ra trực quan và thông tin về cấu trúc của hợp đồng. Nó hỗ trợ tính năng biểu đồ cho các lời gọi hàm.
- [Solgraph](https://github.com/raineorshine/solgraph)  - Tạo biểu đồ DOT trực quan hóa luồng điều khiển chức năng của hợp đồng Solidity và làm nổi bật các lỗ hổng bảo mật tiềm ẩn.
- [EVM Lab](https://github.com/ethereum/evmlab)  - Gói công cụ phong phú để tương tác với EVM. Bao gồm VM, API Etherchain.
- [ethereum-graph-debugger](https://github.com/fergarrui/ethereum-graph-debugger)  - Một trình gỡ lỗi EVM bằng đồ họa. Hiển thị toàn bộ biểu đồ luồng điều khiển chương trình.

## Static and Dynamic Analysis

- [MythX](https://mythx.io/)  - Các công cụ và extension phân tích bảo mật chuyên nghiệp cho Truffle, Embark và các môi trường khác.
- [Mythril](https://github.com/ConsenSys/mythril)  - Một con dao quân đội Thụy Sĩ thực thụ cho bảo mật hợp đồng thông minh.
- [Slither](https://github.com/trailofbits/slither)  - Framework phân tích cho nhiều vấn đề phổ biến với Solidity. Nó viết bằng Python.
- [Echidna](https://github.com/trailofbits/echidna)  - Trong môi trường testing,  tạo đầu vào độc hại nhằm phá vỡ hợp đồng thông minh.
- [Manticore](https://github.com/trailofbits/manticore)  - Công cụ phân tích mã nhị phân với sự hỗ trợ EVM.
- [Oyente](https://github.com/melonproject/oyente)  - Phân tích mã Ethereum để tìm các lỗ hổng phổ biến.
- [Securify](https://github.com/eth-sri/securify2)  -phân tích trực tuyến một cách hoàn toàn tự động cho các hợp đồng thông minh, cung cấp các báo cáo bảo mật dựa trên các lỗ hổng đã biết.
- [SmartCheck](https://tool.smartdec.net/)  - Phân tích tĩnh mã nguồn Solidity với các lỗ hổng bảo mật.
- [Octopus](https://github.com/quoscient/octopus)  - Công cụ phân tích bảo mật cho hợp đồng thông minh Blockchain với sự hỗ trợ của EVM và (e) WASM.

## Weakness OSSClassifcation & Test Cases

- [SWC-registry](https://github.com/SmartContractSecurity/SWC-registry/)  - Các định nghĩa SWC và một repo lớn các mẫu hợp đồng thông minh dễ bị tấn công.
- [SWC Pages](https://smartcontractsecurity.github.io/SWC-registry/)  - Repo đăng ký SWC được xuất bản trên Github Pages.

## Test Coverage

- [solidity-coverage](https://github.com/sc-forks/solidity-coverage)  - Code coverage và Solidity testing.

## Linters

Linters cải thiện chất lượng mã nguồn bằng cách thực thi các quy tắc làm cho mã dễ đọc và xem xét hơn.

- [Solcheck](https://github.com/federicobond/solcheck)  - Một phiên bản eslint cho mã Solidity được viết bằng JS.
- [Solint](https://github.com/weifund/solint)  - Solid linting giúp bạn thực thi các quy ước nhất quán và tránh các lỗi trong hợp đồng thông minh Solidity của bạn.
- [Solium](https://github.com/duaraghav8/Solium)
- [Solhint](https://github.com/protofire/solhint)  - Cung cấp cho người dùng viết Solidity một cách Bảo mật và Phong cách.

<a name="eip"></a>

# EIPS

## Các đề xuất EIP liên quan đến bảo mật

Các đề xuất EIP sau đây rất quan trọng để nhận biết, hiểu cách EVM hoạt động hoặc thông báo các kỹ thuật mới áp dụng khi phát triển hệ thống hợp đồng thông minh.

Dưới đây chưa phải là một danh sách thật sự đầy đủ

### Các đề xuất hoàn chỉnh

- [EIP 155](https://eips.ethereum.org/EIPS/eip-155)  Bảo vệ trước các vụ tấn công replay đơn giản
- [EIP 214](https://eips.ethereum.org/EIPS/eip-214)  Opcode STATICCALL mới - Thêm một opcode mới có thể được sử dụng để gọi một hợp đồng khác (hoặc chính nó) trong khi không cho phép bất kỳ sửa đổi nào đối với trạng thái trong suốt lời gọi.
- [EIP 607](https://eips.ethereum.org/EIPS/eip-607) Triển khai các biện pháp phồng chống tấn công replay đơn giản từ EIP 155.
- [EIP 779](https://eips.ethereum.org/EIPS/eip-779) Tài liệu này ghi lại những thay đổi có trong hard fork có tên là "DAO Fork". Không giống như các hard fork khác, DAO Fork không thay đổi giao thức; tất cả các mã EVM, định dạng giao dịch, cấu trúc block, v.v vẫn giữ nguyên.

### Các đề xuất còn đang trong quá trình sửa đổi, phát triển

- [EIP 1470](https://eips.ethereum.org/EIPS/eip-1470)  Phân loại điểm yếu hợp đồng thông minh - Đề xuất một sơ đồ phân loại cho các điểm yếu bảo mật trong hợp đồng thông minh Ethereum. (SWC)
- [EIP 1051](https://eips.ethereum.org/EIPS/eip-1051) Kiểm tra lỗi Overflow cho EVM - Thêm hai opcode mới cho phép phát hiện và ngăn chặn overflow hiệu quả.
- [EIP 1271](https://eips.ethereum.org/EIPS/eip-1271) Phương thức xác thực chữ ký tiêu chuẩn cho hợp đồng thông minh - Thiết kế hiện tại của nhiều hợp đồng thông minh là không có khóa riêng (private key) và do đó không thể ký trực tiếp message. Đề xuất ở đây phác thảo một cách tiêu chuẩn cho các hợp đồng để xác minh xem chữ ký được cung cấp có hợp lệ khi tài khoản là hợp đồng không ?

<a name="resource"></a>

# Các tài nguyên tham khảo

Dưới đây là danh sách các nguồn tài nguyên hữu ích để tham khải về bảo mật trong Ethereum cũng như Solidity. Nguồn tin chính thống thông báo về các vấn đề bảo mật là Blog Ethereum, nhưng trong nhiều trường hợp, các lỗ hổng sẽ được tiết lộ và thảo luận trước đó ở các nơi khác.

- [Ethereum Blog](https://blog.ethereum.org/): The official Ethereum blog
- [Ethereum Gitter](https://gitter.im/orgs/ethereum/rooms)  chat rooms
  - [Solidity](https://gitter.im/ethereum/solidity)
  - [Go-Ethereum](https://gitter.im/ethereum/go-ethereum)
  - [CPP-Ethereum](https://gitter.im/ethereum/cpp-ethereum)
  - [Research](https://gitter.im/ethereum/research)
- [Smart Contract Security Weekly](https://tinyletter.com/smart-contract-security):Cập nhật hàng tuần về thông Hợp đồng thông minh Ethereum và Bảo mật cơ sở hạ tầng ([Past Articles](https://tinyletter.com/smart-contract-security/archive))
- [Reddit](https://www.reddit.com/r/ethereum)
- [Network Stats](https://ethstats.net/)

Chúng tôi khuyên bạn nên thường xuyên đọc các nguồn này, vì các phương pháp khai thác phát hiện có thể ảnh hưởng đến hợp đồng của bạn.

Ngoài ra, đây là danh sách các nhà phát triển của Ethereum:

- **Vitalik Buterin**:  [Twitter](https://twitter.com/vitalikbuterin),  [Github](https://github.com/vbuterin),  [Reddit](https://www.reddit.com/user/vbuterin),  [Ethereum Blog](https://blog.ethereum.org/author/vitalik-buterin/)
- **Dr. Christian Reitwiessner**:  [Twitter](https://twitter.com/ethchris),  [Github](https://github.com/chriseth),  [Ethereum Blog](https://blog.ethereum.org/author/christian_r/)
- **Dr. Gavin Wood**:  [Twitter](https://twitter.com/gavofyork),  [Blog](http://gavwood.com/),  [Github](https://github.com/gavofyork)
- **Vlad Zamfir**:  [Twitter](https://twitter.com/vladzamfir),  [Github](https://github.com/vladzamfir),  [Ethereum Blog](https://blog.ethereum.org/author/vlad/)

Và cuối cùng, chúng ta có the tham khảo các danh sách các bài viết về bảo mật trong Ethereum được viết từ các nhà phát triển Ethereum đến những người khác trong cộng động.

#### Viết bởi các nhà phát triển Ethereum

- [How to Write Safe Smart Contracts](https://chriseth.github.io/notes/talks/safe_solidity)  (Christian Reitwiessner)
- [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/)  (Christian Reitwiessner)
- [Thinking about Smart Contract Security](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/)  (Vitalik Buterin)
- [Solidity](http://solidity.readthedocs.io/)
- [Solidity Security Considerations](http://solidity.readthedocs.io/en/latest/security-considerations.html)

#### Viết bởi cộng đồng

- [https://blog.sigmaprime.io/solidity-security.html](https://blog.sigmaprime.io/solidity-security.html)
- [http://forum.ethereum.org/discussion/1317/reentrant-contracts](http://forum.ethereum.org/discussion/1317/reentrant-contracts)
- [http://hackingdistributed.com/2016/06/16/scanning-live-ethereum-contracts-for-bugs/](http://hackingdistributed.com/2016/06/16/scanning-live-ethereum-contracts-for-bugs/)
- [http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/)
- [http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/](http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/)
- [http://martin.swende.se/blog/Devcon1-and-contract-security.html](http://martin.swende.se/blog/Devcon1-and-contract-security.html)
- [http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf](http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf)
- [http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour)
- [http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful](http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful)
- [http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal](http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal)
- [https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a](https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a)
- [https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e](https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e)
- [https://blog.vdice.io/wp-content/uploads/2016/11/vsliceaudit_v1.3.pdf](https://blog.vdice.io/wp-content/uploads/2016/11/vsliceaudit_v1.3.pdf)
- [https://eprint.iacr.org/2016/1007.pdf](https://eprint.iacr.org/2016/1007.pdf)
- [https://github.com/Bunjin/Rouleth/blob/master/Security.md](https://github.com/Bunjin/Rouleth/blob/master/Security.md)
- [https://github.com/LeastAuthority/ethereum-analyses](https://github.com/LeastAuthority/ethereum-analyses)
- [https://github.com/bokkypoobah/ParityMultisigRecoveryReconciliation](https://github.com/bokkypoobah/ParityMultisigRecoveryReconciliation)
- [https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c](https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c)
- [https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b](https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b)
- [https://medium.com/@hrishiolickel/why-smart-contracts-fail-undiscovered-bugs-and-what-we-can-do-about-them-119aa2843007](https://medium.com/@hrishiolickel/why-smart-contracts-fail-undiscovered-bugs-and-what-we-can-do-about-them-119aa2843007)
- [https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc)
- [https://medium.com/zeppelin-blog/zeppelin-framework-proposal-and-development-roadmap-fdfa9a3a32ab](https://medium.com/zeppelin-blog/zeppelin-framework-proposal-and-development-roadmap-fdfa9a3a32ab)
- [https://pdaian.com/blog/chasing-the-dao-attackers-wake](https://pdaian.com/blog/chasing-the-dao-attackers-wake)
- [http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf)

# Các chương trình thưởng cho những người tìm ra lỗi

Sau đây là các chương trình tiền thưởng cho lỗi đang diễn ra, tập trung vào việc tìm lỗi cuar các hợp đồng thông minh.

- [0xProject](https://0xproject.com/wiki#Bug-Bounty)
- [Airswap](https://medium.com/fluidity/smart-contracts-and-bug-bounty-ad75733eb53f)
- [Augur](https://www.augur.net/bounty/)
- [Aragon](https://wiki.aragon.org/dev/bug_bounty/)
- [BrickBlock](https://blog.brickblock.io/join-the-brickblock-bug-bounty-program-7b431f2bcc02)
- [Colony.io](https://blog.colony.io/announcing-the-colony-network-bug-bounty-f44cabaca9a3/)
- [Ethereum Foundation](https://bounty.ethereum.org/#bounty-scope)
- [Etherscan.io](https://etherscan.io/bugbounty)
- [Gitcoin Bounties](https://gitcoin.co/explorer)
- [MelonPort](https://melonport.com/bug-bounty)
- [Parity](https://www.parity.io/bug-bounty/)
- [Raiden.network](https://raiden.network/bug-bounty.html)

## Reviewers

## The following people have reviewed this document (date and commit they reviewed in parentheses): Bill Gleim (07/29/2016 3495fb5) Bill Gleim (03/15/2017 0244f4e)

## License

Licensed under [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)

Licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/)
