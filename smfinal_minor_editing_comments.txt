pragma solidity ^0.4.11;
import "github.com/oraclize/ethereum-api/oraclizeAPI.sol";

contract HealthcareContract is usingOraclize {
   
    string public ETHUSD;
    address public owner; 
    uint256 public percentage;
    uint256 public checkTime;
    uint256 public addicted=0; // total number of patient still in rehab phase. status= false
    uint256 public totalPatients=0; // overall total number of patient, both in rehab and done with rehab (both patient.status true and false)
    
    // Unit test data
    address public BobAddress;
    address public JannaAddress;
    address public RobertAddress;
   
   /* struct doctor{
        address addrs;
    }*/
    struct patient{
      string name;
      bool    status; 
      // status=false still in the rehab facility
      // status=true means the patient finished sucsseflly the rehab program and left the rehab facility
      address payer;
      address herab; // health care facility 
      uint amount; // amount in ether
      uint256 startDate;
    }
    
    mapping(address => patient)public Patients;
    mapping(uint256 => address)public PatientsCounter;
    mapping(address => bool)public Doctors;
    
    
    event LogConstructorInitiated(string nextStep);
    event log(string msg,address to,uint256 amount);
    event LogPriceUpdated(string price);
    event LogNewOraclizeQuery(string description);
    
    //modifiers
	modifier onlyOwner(){
		if (owner!=msg.sender){
			throw;
		}else{
			_;
		}
	}
	//modifiers
	modifier onlyDoctors(){
		if (!Doctors[msg.sender]){
			throw;
		}else{
			_;
		}
	}
    function HealthcareContract() payable {
        LogConstructorInitiated("Constructor was initiated. Call 'start()' to send the Oraclize Query.");
        percentage=70;
        
        // Unit test data
        BobAddress=0x34172D052F22A3e57F346c6F9430F32Ff2880065;
        JannaAddress=0xcB20023eD7d39a5fC75Be72688d13045e00C7e6D;
        RobertAddress=0x992eA4c1bd672331E7Df5C327041f89ef0F89E52;
        
        owner=msg.sender;
        // check every 24h (86400 seconds)
        checkTime=86400;
        
        // Unit test data 
        //  _addPatient("Alex",owner);
        //  _addPatient("Bob",BobAddress);
        //  _addPatient("Janna",JannaAddress);
        //  _addPatient("Robert",RobertAddress);
        
    }

    function __callback(bytes32 myid, string result) {
        if (msg.sender != oraclize_cbAddress()) revert();
        ETHUSD = result;
        
        // Send Conditions 
        //  at t0: 70% the rehab
        //  at t0 + 6M 30% will be distributed as below
        // if clean then
        //          a)10% will go to rehab facility
        //          b)10% will go to payer facility    
        //          c)10% will go to patient facility
        // else = if red or nothing
        //          all remaining 30% goes to rehab facily
        
        // unit test data
        //etherSend(RobertAddress,0.1 ether);
    
        checkPatients();
        LogPriceUpdated(result);
        start();
        
    }
    
    //start trigger  
    function start() payable {
        if (oraclize_getPrice("URL") > this.balance) {
            LogNewOraclizeQuery("Oraclize query was NOT sent, please add some ETH to cover for the query fee");
        } else {
            LogNewOraclizeQuery("Oraclize query was sent, standing by for the answer..");
            
            oraclize_query(checkTime, "URL", "json(https://api.gdax.com/products/ETH-USD/ticker).price");
        }
    }
    //deposit from insurance for a patient
  function deposit(address _patient)public payable returns(bool){
        require (msg.value > 0);
        Patients[_patient].herab=msg.sender;
        //send 70% to rehab facility
        uint first_amount=percentage*msg.value/100;
        etherSend(Patients[_patient].herab,_patient,first_amount);
        Patients[_patient].amount=msg.value-first_amount;
        return true;
    }
    //update this function to be from patient to insurance 
    //just add Patients[patient].amount=Patients[patient].amount-amount;
    //and send to Patients[patient].insurance
    
     function etherSend(address to,address _patient,uint _amount)private returns(bool){
        to.transfer(_amount);
        Patients[_patient].amount=Patients[_patient].amount-_amount;
        log("send ether",to,_amount);
        return true;
    }
    
     //Add new patient 
    function  addPatient(string name,address _patient)public payable returns(bool){
        require (msg.value > 0);
        Patients[_patient].payer=msg.sender;
        //send 70% to facility
        uint first_amount=percentage*msg.value/100;
        Patients[_patient].amount=msg.value-first_amount;
        Patients[_patient].name=name;
        Patients[_patient].status=false;  //false means patient joined the rehab facility, see comment above in the patient struct
        Patients[_patient].amount=0;
        Patients[_patient].startDate=now;
        PatientsCounter[totalPatients]=_patient;
        addicted=addicted+1; // number of patients that are still addicted
        totalPatients=totalPatients+1; // total nomber of patients, both addicted and the one fully recovered
         return true;
    }
    
    //Add new patient - initial/basic version
     /* function  _addPatient(string name,address addrs)public{
        PatientsCounter[totalPatients]=addrs;
        Patients[addrs].name=name;
        Patients[addrs].status=false;  //false means patient joined the rehab facility, see comment above in the patient struct
        Patients[addrs].amount=0;
        Patients[addrs].startDate=now;
        addicted=addicted+1; // nomber of patients that are still addicted
        totalPatients=totalPatients+1; // total nomber of patients, both addicted and the one fully recovered
        
    }*/
    
    //change the status of a patient
    //onlyDoctors
    function changePatientStatus (address _patient) public {
        Patients[_patient].status=true; 
        //true means the patient finished sucsseflly the rehab program and left the rehab facility
        addicted=addicted-1;
    }
    
    function _getPatient(address _patient) public constant returns(string,address,uint256,bool){
         return (Patients[_patient].name,Patients[_patient].payer,Patients[_patient].amount,Patients[_patient].status);
    }
    
     function _getAddictedPatient() public constant returns(address[]){
        address[] memory v = new address[](addicted);
        uint counter = 0;
        for (uint i = 0;i < totalPatients; i++) {
            if (!Patients[PatientsCounter[i]].status) {
                v[counter] = PatientsCounter[i];
                counter++;
            }
        }
        return v ;
    }
    
    function checkPatients() returns(bool){
         address[] memory v = _getPatientToCheck();
         require(v.length>0);
          for (uint i = 0;i < v.length; i++) {
              if(Patients[v[i]].status)
                {
                    etherSend(Patients[v[i]].herab,v[i],Patients[v[i]].amount/3);
                    etherSend(Patients[v[i]].payer,v[i],Patients[v[i]].amount/2);
                    etherSend(v[i],v[i],Patients[v[i]].amount);
                }else
                    etherSend(Patients[v[i]].payer,v[i],Patients[v[i]].amount);//all rest amount to payer
          }
         
    }
     function _getPatientToCheck() public constant returns(address[]){
        address[] memory v = new address[](addicted);
        uint counter = 0;
        for (uint i = 0;i < totalPatients; i++) {
            if (now-Patients[PatientsCounter[i]].startDate>24 weeks&&Patients[PatientsCounter[i]].amount>10) {
                v[counter] = PatientsCounter[i];
                counter++;
            }
        }
        return v ;
    }
    
    
     function _getTotalBalance() public constant onlyOwner returns(uint256){
         return address(this).balance;
    }
    function setPercentage(uint256 per)public onlyOwner returns(bool) {
        percentage=per;
    }
    
}