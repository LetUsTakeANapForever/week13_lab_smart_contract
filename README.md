# week13_lab_smart_contract
By ณิชมน ประกาศวุฒิชน 6621604548



## Explanation

อธิบายโค้ดและการทำงานทั้งหมด:

- อธิบายโค้ดที่ป้องกันการ lock เงินไว้ใน contract

```bash
// ใน Function addPLayer

timeUnit.setStartTime(); // ถ้าหากมีผู้เล่นคนหนึ่งถูกเพิ่มเข้ามาเล่นแล้วจะมีการจับเวลาทันที

if (numPlayer == 2) { // ถ้าผู้เล่นครบ 2 คนแล้วจะมีการ set start time อีกครั้งเพื่อจับเวลากันการ lock เงินใน contract 
    timeUnit.setStartTime();
    forcedEndGame(); // และเรียกใช้งาน function forcedEndGame();
}else if (numPlayer == 1){
    checkIfRefund();
}
```
```bash
   // Logic เพื่อเช็กว่าเวลาผ่านไปครบ 1 ชั่วโมง (3600 วินาที) หรือยัง
    function forcedEndGame() public {
        if (timeUnit.elapsedSeconds() > 3600){
            enableRetrieval = true; // ถ้าครบแล้วจะมีการเปลี่ยน status ของการคืนเงินหรือดึงเงินคืนให้ผู้เล่น set เป็น true
        }
    }
```
```bash
// Function ที่ไว้เรียกเงินคืนโดยจะสามารถเรียกได้หลังจากสถานะ enableRetrieval เป็น true
    function retriveETH() public {
        // เช็กว่าสถานะ enableRetrieval เป็น true และ มีคน input เข้ามาแล้ว 1 ท่าน และคนที่จะเรียกใช้งาน function ดึงเงินคืนนี้เป็นคนเดียวกันกับที่ commit ไปแล้วใช่หรือไม่
        if (enableRetrieval && numInput == 1 && committedStatus[msg.sender]){
            // ถ้าผ่านเงื่อนไขจะทำการคืนเงินให้ player ที่เรียกใช้งาน function รวมถึงลบ player คนนั้นออกไปจากเกม และ reset state ของเกมเพื่อรองรับการเล่นรอบถัดไป
            payable(msg.sender).transfer(1 ether);
            numInput--;
            numPlayer--;
            reward = 0;
            player_not_played[msg.sender] = false;
            removePlayer(msg.sender);
        }
    }

```
```bash
// Function ทำการลบ player ที่ทำการเรียกใช้งาน function retriveETH();
function removePlayer(address p) private {

        // check ว่าเป็น player คนที่เท่าไหร่
        uint playerIndex = (players[0] == p) ? 0 : 1;
        address temp;

        // เริ่มทำการลบ player ออก
        if (playerIndex == 1){ // ถ้าเป็น player 1 ที่อยู่ top สามารถ pop(); ออกได้เลย
            players.pop();
        }else{ // กรณีเป็น player 0 จะทำการสลับตำแหน่งกับ player 1 แล้ว pop(); ออก
            temp = players[1];
            players[1] = players[playerIndex];
            players[0] = temp;
            players.pop();
        }
    }
```


- อธิบายโค้ดส่วนที่ทำการซ่อน choice และ commit
```bash
// Function ที่จะให้แต่ละ player เริ่ม input choice ของตัวเองที่ถูก hash เพื่อทำ hiding ไว้เรียบร้อยแล้วเข้ามา
    function inputHash(bytes32 hashedChoice) public  {

        require(numPlayer == 2);
        require(player_not_played[msg.sender]); 
        
        // เรียกใช้งาน function commit จาก CommitReveal.sol เพื่อ commit choice ที่ถูก hash แล้วลงไป
        commitReveal.commit(hashedChoice);

        // Set สถานะ player ที่ commit แล้วเป็น true
        committedStatus[msg.sender] = true;
        

        player_not_played[msg.sender] = false;
        numInput++;


        // เริ่มจับเวลา startTime 
        timeUnit.setStartTime();
    }
```

- อธิบายโค้ดส่วนที่จัดการกับความล่าช้าที่ผู้เล่นไม่ครบทั้งสองคนเสียที
```bash
// ใน Function addPLayer จะมี logic เพื่อเช็กว่าผู้เล่นเข้ามาครบ 2 หรือยัง

timeUnit.setStartTime(); // โดยถ้าหากมีผู้เล่นคนหนึ่งถูกเพิ่มเข้ามาเล่นแล้วจะมีการจับเวลาทันที

if (numPlayer == 2) {
    timeUnit.setStartTime();
    forcedEndGame();
}else if (numPlayer == 1){ // ถ้าหากผู้เล่นไม่ครบเสียที โดยที่เวลาดำเนินไปเรื่อยๆ จะมีการเรียกใช้ function checkIfRefund();
    checkIfRefund();
}
```
```bash
// Function ไว้เช็กว่าถ้าหากเวลาผ่านไปเกิน 1 ชั่วโมง (3600 วินาที) จะทำการคืนเงินให้ผู้เล่น
function checkIfRefund() payable public{
        if (timeUnit.elapsedSeconds() > 3600){
            // คืนเงินให้ player 0 และ reset state ทั้งหมดเพื่อรองรับการเล่นรอบถัดไป
            payable(players[0]).transfer(1 ether);
            numInput = numPlayer = reward = 0;
            players.pop();
        }
    }
```

- อธิบายโค้ดส่วนทำการ reveal และนำ choice มาตัดสินผู้ชนะ
```bash
// Function จัดการการ reveal คำตอบ
    function inputHexToReveal(bytes32 dataInput) public{

        require(numInput == 2); // player ทั้งสองต้อง input choice ที่ตัวเองเลือกมาก่อนถึงจะเรียกใช้งานฟังก์ชันนี้ได้

        // Player แต่ละคนต้อง input dataInput ซึ่งไว้ reveal คำตอบแล้วเรียกใช้ function reveal จาก CommitReveal.sol
        commitReveal.reveal(dataInput, msg.sender);

        numInputToReveal++;

        // ตัด byte ตัวสุดท้ายของ dataInput ซึ่งถือเป็น choice ของ player
        choices[msg.sender] = uint(uint8(dataInput[31]));

        // เช็กว่า player ทั้งสองคนได้ทำการ input ข้อมูลเพื่อ reveal แล้วถึงจะสามารถทำการตัดสินผู้ชนะและมอบรางวัลได้
        if (numInputToReveal == 2) {
            _checkWinnerAndPay();
        }
    }
``` 
```bash
// Function เรียกเช็กสถานะ revealed ของผู้เล่นแต่ละคน
 function hasRevealedStatus(address player) private  view returns(bool) {
        (, , bool status) = commitReveal.commits(player);
        return status;
    }
``` 
```bash
// Function จัดการการตัดสินผู้ชนะและให้ reward 
function _checkWinnerAndPay() private {
        // เช็กอีกครั้งว่า player ทั้งสองคน reveal แล้วหรือยัง ถ้ายัง จะไม่สามารถตัดสินผู้ชนะได้
        require(hasRevealedStatus[players[0]] && hasRevealedStatus[players[1]]);

        // ดึง choice ของ player 0 ออกมา
        uint p0Choice = choices[players[0]];

        // ดึง choice ของ player 1 ออกมา
        uint p1Choice = choices[players[1]];

        address payable account0 = payable(players[0]);
        address payable account1 = payable(players[1]);
        uint winner = getWinner(p0Choice, p1Choice);

        if (winner == 0) {
        account0.transfer(reward);
        } else if (winner == 1) {
            account1.transfer(reward);
        } else {
            account0.transfer(reward / 2);
            account1.transfer(reward / 2);
        }
        // Reset ทุก state ของเกมเพื่อรองรับการเล่นรอบถัดไป
        numInput = numPlayer = reward = 0;
        players.pop();
        players.pop();
    }

``` 
```bash
// Function เอาไว้ตัดสินว่าใครชนะ และจะ return หมายเลขคนชนะออกไป โดยถ้าเป็น 2 หมายถึงเสมอกัน
   function getWinner(uint p0Choice, uint p1Choice) private pure returns (uint) {
        if (p0Choice == p1Choice) return 2; // เสมอ

        // รวมทุกกรณีที่ player 0 ชนะ
        if ((p0Choice == 0 && (p1Choice == 2 || p1Choice == 4)) || // Rock beats Scissors, Lizard
            (p0Choice == 1 && (p1Choice == 0 || p1Choice == 3)) || // Paper beats Rock, Spock
            (p0Choice == 2 && (p1Choice == 1 || p1Choice == 4)) || // Scissors beats Paper, Lizard
            (p0Choice == 3 && (p1Choice == 0 || p1Choice == 2)) || // Spock beats Rock, Scissors
            (p0Choice == 4 && (p1Choice == 1 || p1Choice == 3))) { // Lizard beats Paper, Spock
            return 0; // player 0 Wins
        }
        return 1; // player 1 ชนะ
    }
``` 



